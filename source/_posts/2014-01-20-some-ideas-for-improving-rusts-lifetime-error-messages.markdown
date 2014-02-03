---
layout: post
title: "Some Ideas for Improving Rust's Lifetime Error Messages"
date: 2014-01-20 19:21:48 -0500
comments: true
categories: [Rust, Independent Study]
---

This semester I have an opportunity to do an independent study on the Rust compiler, supervised by my professor David Evans. The particular topic I'm thinking of exploring right now is the Rust lifetime system. This blog will serve as a place where I can document my progress (unless I feel lazy about blogging and this becomes the last post). It will also (hopefully) serve the dual purpose of showing professor Evans that I'm not slacking off.

If I have to rate my current knowledge of Rust, on a scale of 0 (no clue) to 10 (mastery), I would put myself at a 3. Embarrassingly, one crucial topic that I have always had trouble dealing with is Rust lifetime. People often say that we should keep learning different languages to explore new ways of thinking. For Rust, the new thing it offers is making the programmer explicitly aware of the lifetime of objects. Thus, given my lack of understanding of lifetime, I have plenty of reasons to study up on it.

As such, if anything I write is incorrect, I would appreciate it if someone were to point it out.

My current plan is to 1) read up the documentation on lifetime and the borrow-checker code (which handles lifetime)  and 2) improve the lifetime error messages. After going 1/4th of the way through the documentation `middle::borrowck::gather_loans::doc`, I feel that I understand it a little better now, and a few ideas have formed in my head.

## Idea #1

This idea wasn't thought of by me but given by Niko Matsakis and Patrick Walton. The idea is to detect common error patterns and suggest a fix. For example, currently, if we compile the following function:

```rust
struct Foo { y: int }
fn bar(x: &Foo) -> &int {
    &x.y
}
```

we get all these scary-looking messages

```rust
test.rs:3:5: 3:9 error: cannot infer an appropriate lifetime for borrow expression due to conflicting requirements
test.rs:3     &x.y
              ^~~~
test.rs:2:25: 4:2 note: first, the lifetime cannot outlive the anonymous lifetime #1 defined on the block at 2:24...
test.rs:2 fn bar(x: &Foo) -> &int {
test.rs:3     &x.y
test.rs:4 }
test.rs:3:5: 3:9 note: ...so that reference does not outlive borrowed content
test.rs:3     &x.y
              ^~~~
test.rs:2:25: 4:2 note: but, the lifetime must be valid for the anonymous lifetime #2 defined on the block at 2:24...
test.rs:2 fn bar(x: &Foo) -> &int {
test.rs:3     &x.y
test.rs:4 }
test.rs:3:5: 3:9 note: ...so that types are compatible (expected `&int` but found `&int`)
test.rs:3     &x.y
              ^~~~
```

It would be nice if in this particular case, we can just tell the user to introduce a lifetime parameter, or even better, give them something like: "perhaps you mean to declare `fn bar<'a>(x: &'a Foo) -> &'a int`?"

One feedback I should seek from others is: are all those notes necessary? Personally, my eyes just glance over them. I have never understood them. Perhaps there are other instances where they may be useful.


### Idea #2

Here's an idea that [ezyang](https://github.com/ezyang) had. He posted this [gist](https://gist.github.com/ezyang/8033155) on IRC last month, though at the time I had no idea what he was doing.

When the Rust compiler reports an error because of a previous borrow, it may be helpful to note where that borrow ends. For example, currently the program

```rust
fn main() {
    let mut x: uint = 2;
    let y = &mut x;
    let z = &x;
}
```

reports

```rust
test.rs:4:13: 4:15 error: cannot borrow `x` as immutable because it is also borrowed as mutable
test.rs:4     let z = &x;
                      ^~
test.rs:3:13: 3:19 note: previous borrow of `x` occurs here
test.rs:3     let y = &mut x;
                      ^~~~~~
```

It may be good to add something like:

```rust
          fn main() {
              let mut x: uint = 2;
              let y = &mut x;
              let z = &x;
          }
note: ^ `y`'s borrow of `x` ends here
```

One case this would be helpful in is when the borrow ends earlier or later than the user expects, and so he has a chance to correct his misconception. On the other hand, it adds extra noise to the compiler, and the compiler is already pretty noisy.

### Idea #3

I will begin this part with examples. First, let's modify the program in Idea #2 a little bit. We will change the last line of the function so that `z` borrows from `x` mutably instead:

```rust
fn main() {
    let mut x: uint = 2;
    let y = &mut x;
    let z = &mut x;
}
```

Compiling it would yield the following messages:

```rust
test.rs:4:13: 4:19 error: cannot borrow `x` as mutable more than once at a time
test.rs:4     let z = &mut x;
                      ^~~~~~
test.rs:3:13: 3:19 note: previous borrow of `x` as mutable occurs here
test.rs:3     let y = &mut x;
                      ^~~~~~
```

While these messages and the messages in Idea #2 are correct, they have two shortcomings:

1. A mutable borrow prevents *both* mutable and immutable borrows. The cause of these two errors are essentially the same, but currently it sounds like they are different. It is also a bit misleading to output "error: cannot borrow `x` as immutable because it is also borrowed as mutable" when it cannot even be borrowed as mutable.

2. They fail to convey one key point: **a borrow of a variable subsequently restricts the usage of that variable in some ways until that borrow ends**.

Thus, in this particular instance, I think we would be well served by outputting the same error message for both and elaborating the note:

```rust
test.rs:4:13: 4:19 error: cannot borrow `x` because it is already borrowed as mutable
test.rs:4     let z = &x      // or &mut x
                      ^~~~~~
test.rs:3:13: 3:19 note: previous borrow of `x` as mutable occurs here; a mutable borrow
prevents subsequent borrow or modification of variable `x` until the borrow ends.
test.rs:3     let y = &mut x;
                      ^~~~~~
```

My only qualm is the wording does not convey explicitly that the restriction is placed on the *variable* only (perhaps "subsequent borrow or modification *using* variable `x`" would be better?)

To summarize Idea #3, I believe that borrow checker errors can be conveyed better by focusing on the *restriction* that the original borrow places rather than reporting the precise details at the error site.