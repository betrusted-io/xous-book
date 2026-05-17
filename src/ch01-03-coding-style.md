# Coding Style

## Lint
Any code in `main` should be free of `rustc` lint.

## Formatting

Generally, most stable APIs and crates should be formatted with the Rust formatter.
However, there are some idioms in Xous code that format quite poorly with the default
Rust formatter options.

We are waiting on the stabilization of more `rustfmt` features to define a custom
`rustfmt.toml` to address this before making formatting mandatory. It's been a few
years waiting for this, though, so we might just bite the bullet and run `rustfmt`
with `+nightly` and the features we need in `rustfmt.toml`, so that
we can have a uniform style.

Trailing whitespaces are frowned upon.

## Exception Handling

### Background
A mistake was made early on in defining the Xous API where all errors were
propagated, even for operations that are supposed to be infalliable or there
is no sensible way to handle an error if it were to arise.

The subtlety is that some errors in Rust are more like `assert` statements.
For example, a function that unpacks a message into a struct could fail. If such
a failure is encountered, then you'd like to see a panic showing that exact line
of code so you can fix the bug. There isn't a sensible alternative code path at run-time,
because the root cause was likely a type mismatch error.

In most of the original code base, that error would be propagated up the call
stack as an `InternalError`, until you get back to your main loop, at which point
the main loop just throws up its hands and reports a panic but at a line of code
several call frames away from the offending statement. Normally this problem
can be fixed by reading the stack trace from a panic-unwind, but Xous does not have
mature panic-unwind support and also the device's screen real estate is limited
so a deep call stack cannot be displayed entirely on the screen.

### Recommendation
We are in the process of refactoring code from going to a "propagate all errors"
rule to a "panic on infalliable failures" rule. Infallible operations include
most syscalls (except ones that are explicitly fallible) and helper methods meant
to transform *inter-process* messages and buffers into types (note that this does
not include methods that receive, for example, arbitrary messages over network).

A simple `.unwrap()` is probably sufficient to check the results of most infalliable operations,
because the panic handler will print a panic on that line.

`unwrap()` may even be preferable to a `.expect("helpful error message")` in most
cases, because "helpful error message" takes memory to store, increases the
binary size, and in most cases the line of code where the panic happened is the most
informative part of the error message.

Any error that happen in fallible operations (timeouts, OOMs, disconnects, etc.)
should be handled and/or passed up the stack.
