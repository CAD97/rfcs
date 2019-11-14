- Feature Name: `fn-pin-mut`
- Start Date: 2019-11-13
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Add the `FnPinMut` trait as a way of representing fully general stackless semicoroutines.
Generators (and as such, `async`) are built on top of `FnPinMut`.
This replaces the eRFC #2033 as the implementation of the functionality first proposed therein.

```rust
trait FnPinMut<Args>: FnOnce<Args> {
    // type Output; // from FnOnce
    extern "rust-call" fn call_pin(self: Pin<&mut Self>, args: Args) -> Self::Output;
}
```

# Motivation
[motivation]: #motivation

`FnMut` represents the class of functions that take self by `&mut Self`.
`FnPinMut` is merely the class of functions that take self by `Pin<&mut Self>`.

This abstraction is immensely useful, as it allows the language to represent
self-referential semicoroutines. In fact, this is so useful that a
specialization of this interface is already stable: `Future`!

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}
```

Specifically, `Future` is isomorphic to `FnPinMut(&mut Context) -> Poll<Output>`.

The primary additional use case that justifies adding `FnPinMut` in addition to
`Future` is `Generator`s, which are experimentally accepted with eRFC #2033.
As of writing, `Generator` is defined in nightly as:

```rust
trait Generator {
    type Yield;
    type Return;
    fn resume(self: Pin<&mut Self>) -> GeneratorState<Self::Yield, Self::Return>;
}
```

and there are ongoing proposals to add optional arguments to `resume`,
which this RFC provides a potential avenue for.

`FnPinMut` is not fundamentally limited to producing `Generator`,
and can be used for other potentially self-referential use cases.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

(see reference-level; I've written a reference-level including guide-level like bits)

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

We add a new trait, `FnPinMut`, with the signature above, to `std::ops`.
It is subject to the same restrictions as the stable `Fn` traits:
it cannot be named directly (use the `FnPinMut()` form),
and it cannot be implemented by hand.
(These restrictions may be lifted in the future.)

An implementation of `FnPinMut` is provided for all closures where said implementation is sound.
Practically, this means all currently stable `FnMut` closures, as they are `Unpin`, get the following implementation:

```rust
impl FnPinMut<Args> for $GeneratedClosureType {
    extern "rust-call" fn call_pin(self: Pin<&mut Self>, args: Args) -> Self::Output {
        (&mut *self).call_mut(args)
    }
}
```

Closures are now allowed to contain **yield expression**s of the form `yield $expr`.
If a closure contains a yield expression, it _may not_ also contain a return expression.
This includes the trailing return: a closure containing a `yield` must "return" `!`.
This will most often be accomplished by a `panic!` or a `loop` at the tail end.
In most ways (allowed positions, return type), a yield statement acts the same as a return expression.

We call a closure that uses `yield` instead of `return` a **yield closure**.

The key difference is that rather than execution restarting at the top of
the closure on the next call, execution resumes after the yield statement.
The compiler will generate a state machine to jump to the correct place upon each resumation.

This is subject to the normal rules about what `Fn` traits are implemented!
This means that a yield closure can implment all four traits: `Fn`, `FnMut`, `FnPinMut`, and `FnOnce`!
For example, [`std::iter`'s `Counter` example](counter) can be reimplemented as simply

```rust
std::iter::from_fn(|| {
    for count in 1..6 {
        yield Some(count);
    }
    loop {
        yield None;
    }
})
```

This closure will implement all four `Fn` traits.

So far, however, everything we've seen has been within the domain of the stable `Fn` traits.
`FnPinMut` is required to support borrowing over `yield` points.

(TODO)

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Exact naming of `trait FnPinMut`. `FnPinMut` was chosen for this RFC as an
  obvious but somewhat flawed name. `Coroutine` or `Semicoroutine` would also
  be a good fit for this concept.
- The exact fate of `trait Generator`. This RFC suggests focuses on the
  `FnPinMut` foundation layer instead. `trait Generator` is still a usefull
  abstraction boundary, just like `Iterator` is distinct from `FnMut() -> Item`.
- Lifetimes, lifetimes, lifetimes!
  - Passing in single-yield lifetimes!
  - Yielding references to closure state!

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how the this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.

  [counter]: <https://doc.rust-lang.org/1.39.0/std/iter/index.html#implementing-iterator>
