<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# Borrows and re-entrancy

If you have used `base()`, `base_mut()` or `to_gd()` for a while, you have probably run into at least one of these:

- A method call that won't compile, even though the two operations involved look perfectly independent.
- An "already bound" panic, seemingly out of nowhere, often triggered by a method you didn't even call directly.
- Code that worked for months, until an unrelated change (a new signal connection, an overridden virtual method) made it panic.

This page assumes you're already familiar with [`base()`/`base_mut()`/`to_gd()`][book-functions-base]. It explains what these three situations
have in common, and how to write code that avoids all of them.


## Table of contents
<!-- toc -->


## What's actually going on

Your Rust struct lives inside a Godot object. Whenever code needs that struct -- to read a field, or to call a method taking `&self` or
`&mut self` -- godot-rust checks _at runtime_ that the usual Rust rules hold: any number of readers, or exactly one writer. This works like
[`RefCell`][rust-refcell]: a conflict causes a panic rather than undefined behavior.

Here is the part that trips people up: **calling an engine method doesn't touch your Rust struct at all.** `self.base().get_position()` runs
engine code; it never reads or writes your fields. On its own, an engine call can't conflict with anything.

What _can_ happen is that the engine calls back into your object before the engine method returns. Godot delivers a notification, runs a
connected signal handler, or invokes a virtual method you overrode -- and that lands in your Rust code again, needing access to the same
struct, _while your first method is still running_. Now there are two callers wanting the same object, and if the first one holds exclusive
(`&mut`) access, the second one panics. This "calling back in while a call is already in progress" is what we call **re-entrancy**.

So the one question that resolves almost every situation on this page is:

> **Can this engine call end up calling back into my object before it returns?**

If the answer is "no" (most engine calls, most of the time), there is no risk. If the answer is "yes" or "I'm not sure", you need to hand back
your exclusive access for the duration of the call -- that is exactly what `base_mut()` and `reentrant()` do for you.

Note that whether a call re-enters is a property of your _whole project_: it depends on which signals are connected, which virtual methods are
overridden -- sometimes in GDScript, where the Rust compiler cannot see them. This is why the check happens at runtime rather than at compile
time: the answer simply isn't in your function's signature.

The rest of this page walks through the three ways the problem shows up, in the order you're likely to meet them.


## Symptom 1: a compile error that looks wrong

```rust
fn apply_movement(&mut self) {
    self.base_mut().set_velocity(self.velocity);
    //   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    // error[E0502]: cannot borrow `*self` as immutable because it is also borrowed as mutable
}
```

This one has nothing to do with re-entrancy -- it's a limitation of the compile-time borrow checker. `base_mut()` needs exclusive access to
`self` for the whole statement, but the argument `self.velocity` also reads `self`, and the compiler isn't able to prove that the combination
is harmless (even though it is: arguments are fully evaluated before the engine call starts).

The fix is to read the value into a local variable first:

```rust
fn apply_movement(&mut self) {
    let velocity = self.velocity;
    self.base_mut().set_velocity(velocity);
}
```

The [`self_call!`][api-self-call] macro writes that local variable for you, for the common case of a single method call:

```rust
fn apply_movement(&mut self) {
    self_call!(self.set_velocity(self.velocity));
}
```

```admonish note title="Passing references still fails -- correctly"
If an argument is a reference into `self` (e.g. `&self.name`), storing it in a local doesn't help: the reference still points into `self`
while the engine call runs. Here the compiler is right to complain -- if the call re-enters your object and mutates that field, the reference
would observe it mid-change. Pass a copy or clone instead.
```


## Symptom 2: a panic out of nowhere

This is re-entrancy proper. The minimal version ([#338][issue-338]):

```rust
#[godot_api]
impl INode for Monster {
    fn ready(&mut self) {
        // add_child() makes Godot send a "child order changed" notification -- which it delivers
        // by calling on_notification() on this very object, while ready() is still running.
        // to_gd() doesn't hand back our &mut access, so the incoming call has nowhere to go.
        self.to_gd().add_child(&Node::new_alloc());
    }

    fn on_notification(&mut self, what: NodeNotification) {
        // Panics: "Gd<T>::bind_mut() failed, already bound".
    }
}
```

In real projects, the loop back to your object is rarely this direct. A typical case ([#916][issue-916]): a state machine, where the manager
forwards input to the current state, and states report back via a signal.

1. `FsmManager::input(&mut self)` runs and forwards the event to the active state node.
2. The state's `on_input` is overridden in GDScript, which decides to switch states and emits its `state_transition` signal.
3. The manager connected that signal to its own `on_state_transition(&mut self)` -- so Godot calls back into the manager...
4. ...whose `input()` from step 1 is still running. Second `&mut self` on the same object: panic.

No single step looks suspicious, and half the chain isn't even visible from Rust. Other shapes of the same problem:

- Emitting a signal on yourself that a handler on the same object is connected to.
- Calling a method on _another_ object (say, a manager or autoload) while holding its `bind_mut()` guard, when that method eventually needs
  the same object again ([#1177][issue-1177]).

The fix is the same in every case: make the call through `base_mut()` or [`reentrant()`][api-reentrant], which hand back your access to `self`
for the duration of the call, so the incoming call can take its turn:

```rust
#[godot_api]
impl INode for Monster {
    fn ready(&mut self) {
        self.base_mut().add_child(&Node::new_alloc());
    }

    fn on_notification(&mut self, what: NodeNotification) {
        // Fine now: our access to self was handed back before add_child() ran.
    }
}
```

```admonish tip title="The panic message tells you where the conflict is"
In debug builds, a failed `bind()`/`bind_mut()` reports where the conflicting access was taken (file and line, plus a full backtrace if you
run with `RUST_BACKTRACE=1`). If you're not sure which of your own calls is causing a panic, that message is the first place to look.
```


## Symptom 3: it worked for months, then it didn't

The most confusing variant, because there's no error when you write the code:

```rust
fn ready(&mut self) {
    let this = self.to_gd();
    this.get_tree().call_group("enemies", "aggro", &[]);
}
```

`to_gd()` returns an independent, owned `Gd<Self>` -- it knows nothing about the `&mut self` your method holds, and hands nothing back. This
compiles and runs fine, right up until some future change makes the call re-entrant: a new signal connection, a newly overridden virtual
method, another class starting to listen to `"aggro"`. At that point this exact code starts panicking, and the panic looks completely
unrelated to whatever you just changed.

The rule to remember:

> **`to_gd()` is for handles you store or pass elsewhere** (into a `HashMap`, a `Callable`, another object's field) --
> **never for making an engine call from inside a method of the same class.** For calls, go through `base()`/`base_mut()`/`reentrant()`,
> even when nothing seems to need it yet. They cost nothing extra when the call doesn't re-enter, and they save you from exactly this trap
> when it later does.


## Which access method, when

| Method | Returns | Mutating calls | Hands back `self` | Use for |
|---|---|---|---|---|
| [`base()`][api-withbasefield-base] | `&Gd<Base>` guard | no (`&self` only) | no | Quick reads; can't re-enter, nothing to hand back. |
| [`reentrant()`][api-reentrant] | closure result | yes | yes, for the closure | Calls that may re-enter. Prefer by default. |
| [`base_mut()`][api-withbasefield-basemut] | `&mut Gd<Base>` guard | yes | yes, while guard lives | Like `reentrant()`, when a closure doesn't fit. |
| [`to_gd()`][api-withbasefield-togd] | owned `Gd<Self>` | -- (don't call) | no | Handles stored/passed elsewhere, never same-class calls. |

Prefer `reentrant()` over `base_mut()` by default: the closure makes the re-entrancy window visible as braces in your code, and there's no way
to accidentally drop the guard too early (with `base_mut()`, writing `let _ = self.base_mut();` silently ends the window on the same line).
Reach for `base_mut()` when the window needs to span control flow a closure can't express -- an early return, or an outer loop body.

```admonish note title="Why do base() and base_mut() exist separately?"
Because _which engine methods can re-enter you_ lines up almost perfectly with _which methods Godot marks as const_: read-only queries
essentially never call back into your code, while setters and anything that triggers notifications or signals might. So `base()` being limited
to const methods isn't arbitrary -- it matches "this can't re-enter you, so there's nothing to hand back".

The split is bypassable (`Gd` is `Clone`, so `self.base().clone().add_child(..)` compiles) and thus not a safety guarantee -- the runtime
check described above is what actually protects you. But like `&self` vs. `&mut self` anywhere else in Rust, it catches a good share of
accidental-modification mistakes for free.
```


## Patterns that keep this painless

**Read first, then call.** Gather everything you need from `self` into local variables, then make your engine calls. This is what `self_call!`
automates for one call, and it's a good habit beyond pleasing the compiler: code that re-enters during the call won't catch your fields
mid-computation.

```rust
fn apply_movement(&mut self, delta: f32) {
    let motion = self.velocity * delta;

    self.reentrant(|base| {
        let pos = base.get_position();
        base.set_position(pos + motion);
    });

    self.on_position_updated();
}
```

**Keep the window small.** While a `reentrant()` closure or `base_mut()` guard is active, _nothing else_ can access your struct -- including
re-entrant calls unrelated to the one you're making. Batch adjacent engine calls into one window, but don't stretch it over code that doesn't
need it.

**For signal-heavy classes, consider `Gd<Self>` receivers.** If a class spends most of its time routing signals and virtual calls (state
machines, event buses), `#[func(gd_self)]` lets methods receive `Gd<Self>` instead of `&mut self` -- including for virtual methods, see
[Script-virtual functions][book-virtual-functions]. You then `bind()`/`bind_mut()` only for the brief moments you actually touch fields, and
hold no access at all while forwarding calls -- which sidesteps most re-entrancy panics for this style of class.

**Compute a plan, then apply it.** For logic that alternates between field access and engine calls, consider splitting it: a pure step that
reads `self` and returns a small "what to do" value, then a single `reentrant()` block that carries it out. This avoids ping-ponging between
borrowed and released code line-by-line, and tends to make the logic easier to test.


## Under the hood

The mechanism behind all of this lives in the `godot-cell` crate. Each user object is wrapped in a cell that counts, at runtime, how many
shared and exclusive references are currently handed out -- conceptually a `RefCell`, with one extra trick: `base_mut()`/`reentrant()` can
mark the current exclusive reference as _temporarily parked_, allowing a new one to be handed out and given back, without two usable `&mut`
references to the same data ever existing at once.

This checking can't be made more lenient without becoming unsound: two simultaneously usable `&mut` references are undefined behavior in Rust,
no matter how careful the code around them is. And since whether a given engine call re-enters depends on your project's dynamic state --
connections, script overrides, engine behavior -- a runtime check is also the only place where the question can be answered at all.


[api-reentrant]: https://godot-rust.github.io/docs/gdext/master/godot/obj/trait.WithBaseField.html#method.reentrant
[api-self-call]: https://godot-rust.github.io/docs/gdext/master/godot/obj/macro.self_call.html
[api-withbasefield-base]: https://godot-rust.github.io/docs/gdext/master/godot/obj/trait.WithBaseField.html#method.base
[api-withbasefield-basemut]: https://godot-rust.github.io/docs/gdext/master/godot/obj/trait.WithBaseField.html#method.base_mut
[api-withbasefield-togd]: https://godot-rust.github.io/docs/gdext/master/godot/obj/trait.WithBaseField.html#method.to_gd
[book-functions-base]: ../register/functions.html#base-access-from-self
[book-virtual-functions]: ../register/virtual-functions.html
[issue-338]: https://github.com/godot-rust/gdext/issues/338
[issue-916]: https://github.com/godot-rust/gdext/issues/916
[issue-1177]: https://github.com/godot-rust/gdext/issues/1177
[rust-refcell]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
