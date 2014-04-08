---
layout: post
title: "Cannot move out of dereference of &amp;-pointer"
date: 2014-04-07 19:24:35 -0400
comments: true
categories: [Rust, Independent Study]
---

Currently I'm looking at the "cannot move out of dereference of `&`-pointer" class of errors, which I personally dread the most and find difficult to deal with. I don't have a good idea about a better general way to present that error, however. Instead, I would like to simply special-casing the error in a match expression, which is where I encounter it the most.

The problem with the way the error is currently presented in match expression is that (sometimes) it's a half-way between "where" the value moves out and "where" we are attempting to move to. For example, from the following code snippet,

```rust
fn main() {
    let opt = &Some(~1u32);
    match *opt {
        Some(num) => (),
        None => ()
    }
}
```

we will get this error:

```rust
test.rs:10:9: 10:18 error: cannot move out of dereference of `&`-pointer
test.rs:10         Some(num) => (),
                   ^~~~~~~~~
```

Notice that the caret and tilde span through the whole `Some(num)` pattern. It's accurate, too, because here we are not allowed to move the value out of `*opt`, and one variant of `*opt` is `Some(num)`. On the other hand, what we really want to tell to the user is that since `num` is capturing an owned pointer by value, it's causing a move, and that's essentially where the problem lies.

I believe giving the user both information will make the story about this error message clearer:

```rust
test.rs:9:11: 9:15 error: cannot move out of dereference of `&`-pointer
test.rs:9     match *opt {
                    ^~~~
test.rs:10:14: 10:17 note: attempting to move inner value to here (to prevent the move, you can use `ref` to capture value by reference)
test.rs:10         Some(num) => (),
                        ^~~
```

Perhaps we can tell the user explicitly to use `ref num`, as opposed to just `ref`. In many cases, this simple step will solve the problem.

The second way we could present the note is to insert `ref` into the match arm pattern explicitly, as follows:

```rust
test.rs:9:11: 9:15 error: cannot move out of dereference of `&`-pointer
test.rs:9     match *opt {
                    ^~~~
test.rs:10:9: 10:18 note: to prevent the move and capture value by reference instead, change the arm to `Some(ref num)`
test.rs:10         Some(num) => (),
                   ^~~~~~~~~
```

However, I'm not too fond of doing this because just injecting `ref` into the AST is more work than it looks. And writing a routine to generate a new arm seems overkill, especially when just telling the user to use `ref num` seems sufficient. In addition, I actually prefer the first note.

That said, in some cases the second way has the advantage of generating less notes. Take this code snippet as an example:

```rust
enum Foo {
    Foo1(~u32, ~u32),
    Foo2,
}

fn main() {
    let f = &Foo1(~1u32, ~2u32);
    match *f {
        Foo1(num1, num2) => (),
        Foo2 => ()
    }
}
```

Using the first method, the error and notes we generate would be:

```rust
test.rs:8:11: 8:13 error: cannot move out of dereference of `&`-pointer
test.rs:8     match *f {
                    ^~
test.rs:9:14: 9:18 note: attempting to move inner value to here (to prevent the move, you can use `ref num1` to capture value by reference)
test.rs:9         Foo1(num1, num2) => (),
                       ^~~~
test.rs:9:20: 9:24 note: attempting to move inner value to here (to prevent the move, you can use `ref num2` to capture value by reference)
test.rs:9         Foo1(num1, num2) => (),
                             ^~~~
```

With the second method, we can generate one less note:

```rust
test.rs:8:11: 8:13 error: cannot move out of dereference of `&`-pointer
test.rs:8     match *f {
                    ^~
test.rs:9:9: 9:25 note: to prevent the move and capture value by reference instead, change the arm to `Foo1(ref num1, ref num2)`
test.rs:9         Foo1(num1, num2) => (),
                  ^~~~~~~~~~~~~~~~
```

I think either of them is an improvement to the current error message:

```rust
test.rs:9:9: 9:25 error: cannot move out of dereference of `&`-pointer
test.rs:9         Foo1(num1, num2) => (),
                  ^~~~~~~~~~~~~~~~
test.rs:9:9: 9:25 error: cannot move out of dereference of `&`-pointer
test.rs:9         Foo1(num1, num2) => (),
                  ^~~~~~~~~~~~~~~~
```