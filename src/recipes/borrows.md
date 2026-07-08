<!--
  ~ Copyright (c) godot-rust; Bromeon and contributors.
  ~ This Source Code Form is subject to the terms of the Mozilla Public
  ~ License, v. 2.0. If a copy of the MPL was not distributed with this
  ~ file, You can obtain one at https://mozilla.org/MPL/2.0/.
-->

# Re-entrancy and borrows

If you have used `base()`, `base_mut()` or `to_gd()` for a while, you have probably run into at least one of these:

- A method call that won't compile, even though the two operations involved look perfectly independent.
- A "already bound" panic, seemingly out of nowhere, often triggered by a method you didn't even call directly.
- Code that worked for months, until an unrelated change (a new signal connection, an overridden virtual) made it panic.

This page assumes you're already familiar with [`base()`/`base_mut()`/`to_gd()`][book-functions-base], and goes deeper into _why_ these
situations occur, what they have in common, and how to structure code that avoids them.


## Table of contents
<!-- toc -->


## The core idea: engine calls don't borrow, re-entrant calls do

`Gd<T>` gives every user object [`RefCell`][rust-refcell]-like borrow checking: `bind()`/`bind_mut()` track whether you currently hold a
shared or exclusive Rust-level reference to the object, and panic on conflicts, the same way `RefCell` does. So far this is just ordinary
interior mutability.

The part that trips people up is this: **calling an engine method never touches that borrow by itself.** `self.base().add_child(...)` runs C++
code; it doesn't read or write your Rust struct at all. The Rust-level borrow only matters if Godot calls back into _your_ code during that
engine call -- because the engine, or a connected signal handler, or an overridden virtual function, ends up needing a `bind()`/`bind_mut()`
on the very same object whose `&mut self` frame is still open on the Rust call stack.

So the one question that resolves almost every situation on this page is:

> **Can this engine call end up calling back into my object before it returns?**

If the answer is "no" (most engine calls, most of the time), there is no risk, and you can use the cheapest tool available. If the answer is
"yes" or "I'm not sure", you need to release your `&mut self` borrow for the duration of the call, so that the re-entrant code can acquire its
own.

Notably, whether a call re-enters is a **dynamic** property of your whole project: it depends on what you connected, what virtuals you
override, and sometimes on engine internals. It is not something the Rust compiler -- or any partial-borrow language feature -- could ever
check for you, because the answer isn't in your function's signature at all. This is why the borrow check for this case happens at runtime,
via `Gd`'s internal cell, rather than at compile time via the borrow checker.


## Three ways this shows up

### 1. A compile-time borrow error

```rust
fn apply_movement(&mut self) {
    self.base_mut().set_velocity(self.velocity);
    //   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    // error[E0502]: cannot borrow `*self` as immutable because it is also borrowed as mutable
}
```

This one has nothing to do with re-entrancy at all -- it is a plain borrow-checker limitation. `base_mut()` mutably borrows `self` for the
duration of the whole statement (it needs to, to set up the reborrow that makes re-entrancy possible later), while the argument list still
reads `self.velocity`. The two operations are safe to combine in principle -- arguments are evaluated to owned values before `self` is
released -- but Rust's two-phase borrows don't extend across a user-defined method call to see that.

The fix is to evaluate the argument first, into a local:

```rust
fn apply_movement(&mut self) {
    let velocity = self.velocity;
    self.base_mut().set_velocity(velocity);
}
```

The [`self_call!`][api-self-call] macro does exactly this hoist for you, for the common case of a single method call:

```rust
fn apply_movement(&mut self) {
    self_call!(self.set_velocity(self.velocity));
}
```

```admonish note title="Hoisting a reference still fails -- correctly"
If an argument is a reference into `self` (e.g. `&self.name`), hoisting it into a local still borrows `self` across the `base_mut()` call, and
will fail to compile for the same reason plain code would. This is not a limitation to work around: code that Godot re-enters during the call
may mutate the field that reference would still be pointing to, so the compiler is right to reject it. Clone the data instead.
```


### 2. A runtime "already bound" panic

This is the re-entrancy case proper. A minimal version:

```rust
#[godot_api]
impl INode for Monster {
    fn ready(&mut self) {
        // add_child() triggers a CHILD_ORDER_CHANGED notification, which Godot delivers by calling
        // on_notification() on this very object -- while this &mut self frame is still on the stack.
        // to_gd() doesn't release that frame, so the notification's own bind_mut() has nowhere to go.
        self.to_gd().add_child(&Node::new_alloc());
    }

    fn on_notification(&mut self, what: NodeNotification) {
        // Panics: "Gd<T>::bind_mut() failed, already bound".
    }
}
```

The same shape shows up whenever a call you make can loop back to `self`, however indirectly:

- Emitting a signal that's connected to a handler on the same object.
- A virtual method overridden in GDScript that calls back into Rust (e.g. via `Callable`), while Rust still holds a Rust-side borrow of the
  object underneath.
- Calling a method on another object which, as part of its own logic, calls back into yours (a shared FSM manager connecting child signals to
  itself, for example).

The fix is the same in every case: don't call through a plain `Gd<Self>` handle while a re-entrant call might come back. Use `base_mut()` (or
[`reentrant()`][api-reentrant], see below) instead, which releases `self` for the duration of the call so the re-entrant `bind_mut()` can
succeed:

```rust
#[godot_api]
impl INode for Monster {
    fn ready(&mut self) {
        self.base_mut().add_child(&Node::new_alloc());
    }

    fn on_notification(&mut self, what: NodeNotification) {
        // Fine now: this call is no longer nested inside an open bind_mut() of the same object.
    }
}
```

```admonish tip title="The panic message tells you where the conflict is"
In debug builds, a failed `bind()`/`bind_mut()` reports the call site of the borrow it conflicted with (file and line, plus a full backtrace
if you run with `RUST_BACKTRACE=1`). If you're not sure which of your own borrows is causing a panic, that message is the first place to look.
```


### 3. "It compiled and worked... until it didn't"

This is the most confusing variant, because there's no error at the time you write the code:

```rust
fn ready(&mut self) {
    let this = self.to_gd();
    this.get_tree().unwrap().call_group("enemies", "aggro", &[]);
}
```

`to_gd()` returns an owned `Gd<Self>`, entirely independent of the `&mut self` borrow -- there is no reborrow relationship to release, because
none was set up. This compiles and runs fine, right up until some unrelated future change makes the call re-entrant: a new signal connection,
a new virtual override, another class starting to listen to `"aggro"`. At that point, this exact code starts panicking, and the panic looks
unrelated to whatever you just changed.

The underlying rule: **`to_gd()` is for handles you store or pass elsewhere (into a `HashMap`, a `Callable`, another object's field) -- never
for making an engine call from inside a method of the same class.** For calls, always go through `base()`/`base_mut()`/`reentrant()`, even
when nothing appears to need it yet. They cost nothing extra when the call happens not to re-enter, and they save you from exactly this trap
when it later does.


## The vocabulary

| Method | Returns | Mutating calls | Releases `self` | Use for |
|---|---|---|---|---|
| [`base()`][api-withbasefield-base] | `&Gd<Base>` guard | no (`&self` only) | no | Quick reads; nothing to release. |
| [`base_mut()`][api-withbasefield-basemut] | `&mut Gd<Base>` guard | yes | yes | Re-entrant calls, release held across statements. |
| [`reentrant()`][api-reentrant] | closure result | yes | yes, scoped | Same as `base_mut()`, self-contained. Prefer by default. |
| [`to_gd()`][api-withbasefield-togd] | owned `Gd<Self>` | -- (don't call) | no | Handles stored/passed elsewhere, never for same-class calls. |

Why does `base()` split from `base_mut()` at all, rather than there being one universal accessor? Because *which engine methods can
re-enter you* correlates almost perfectly with *whether Godot considers the method "const"*: notifications, property setters, and most calls
that trigger callbacks are non-const; read-only queries essentially never call back into your code. So `base()`'s restriction to const
methods isn't arbitrary -- it exactly matches "this can't re-enter you, so there's nothing to release".

```admonish note title="Const-ness isn't a safety guarantee, but it catches real mistakes"
The const/non-const split is bypassable: since `Gd` is `Clone`, nothing stops you from writing `self.base().clone().add_child(...)` to work
around a `base()` restriction. It's not a soundness boundary. In practice, though, requiring `mut` to call mutating methods catches a good
share of accidental-modification bugs for free, the same way `&self`/`&mut self` does anywhere else in Rust -- so the split is kept even
though it's not the mechanism protecting you from re-entrancy panics. That mechanism is the runtime borrow check described above.
```

Prefer `reentrant()` over `base_mut()` by default: a closure gives you a visible scope in the code, and there's no way to accidentally write
`let _ = self.reentrant(...)` and have the release end immediately (a mistake `let _ = self.base_mut();` makes easy, since the guard's whole
job is to be held, not used). Reach for `base_mut()` directly only when the release needs to span control flow a closure can't express
(early returns, or holding it across an outer loop body).


## Patterns

### Hoist, then release

The general shape behind [`self_call!`][api-self-call]: read everything you need from `self` into locals first, _then_ call through
`base_mut()`/`reentrant()`. This isn't a workaround for the compiler -- it's the same discipline you'd want even without the borrow checker's
help, since re-entrant code invoked during the call might otherwise observe or mutate your fields mid-computation.

### Scope releases tightly

```rust
fn apply_movement(&mut self, delta: f32) {
    // Artificial scope, or use reentrant() instead -- see below.
    {
        let mut base = self.base_mut();
        let pos = base.get_position();
        base.set_position(pos + self.velocity * delta);
    } // Guard dropped here; self is fully accessible again.

    self.on_position_updated();
}
```

Holding a `base_mut()`/`reentrant()` release for longer than necessary blocks _all_ access to `self` from anywhere else for that whole
duration -- including from re-entrant calls that have nothing to do with the one you're making. Keep the scope as tight as the call requires.

### `gd_self` + short binds, for signal/FSM-heavy classes

If a class spends most of its time reacting to signals and virtual calls that route through other objects (state machines, event buses),
consider `#[func(gd_self)]` receivers (including on virtuals, see [Script-virtual functions][book-virtual-functions]) instead of `&mut self`.
With a `Gd<Self>` receiver, you `bind()`/`bind_mut()` only for the brief moment you actually touch fields, and hold no borrow at all while
routing calls onward -- which sidesteps most re-entrancy panics for this style of class, at the cost of an explicit bind per field access.

### Plan, then apply

For logic that's naturally "compute, then make several engine calls, then compute some more", consider splitting it into a pure step that
reads `self` and returns a plan (a small enum or struct describing what to do), and a single `reentrant()`/`base_mut()` block that applies it.
This avoids interleaving borrowed and released regions statement-by-statement, and as a side effect tends to make the logic easier to test.


## Under the hood

The mechanism behind all of this lives in the `godot-cell` crate: each user object is wrapped in a cell that tracks, at runtime, how many
shared and exclusive Rust references are currently handed out -- conceptually a `RefCell`, plus the extra "inaccessible" state that
`base_mut()`/`reentrant()` use to mark the current exclusive reference as temporarily released, so a new one can be handed out and then
handed back without ever having two live `&mut` references to the same data at once.

This is also why the checking happens at runtime rather than compile time, and why it can't be made more lenient without becoming unsound:
whether a given engine call re-enters your object is not knowable from your function's types, only from the dynamic state of your whole
project (connections, overrides, engine behavior) -- so there is no sound way for the compiler to wave it through.


[api-reentrant]: https://godot-rust.github.io/docs/gdext/master/godot/obj/trait.WithBaseField.html#method.reentrant
[api-self-call]: https://godot-rust.github.io/docs/gdext/master/godot/obj/macro.self_call.html
[api-withbasefield-base]: https://godot-rust.github.io/docs/gdext/master/godot/obj/trait.WithBaseField.html#method.base
[api-withbasefield-basemut]: https://godot-rust.github.io/docs/gdext/master/godot/obj/trait.WithBaseField.html#method.base_mut
[api-withbasefield-togd]: https://godot-rust.github.io/docs/gdext/master/godot/obj/trait.WithBaseField.html#method.to_gd
[book-functions-base]: ../register/functions.html#base-access-from-self
[book-virtual-functions]: ../register/virtual-functions.html
[rust-refcell]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
