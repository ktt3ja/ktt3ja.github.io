---
layout: post
title: "Understanding Rust's Lifetime Inference"
date: 2014-02-10 17:12:11 -0500
comments: true
categories: [Rust, Independent Study]
---

Last week, I continued my single case study of the following function and the errors it generated:

```rust
struct Foo { y: int }
fn bar(x: &Foo) -> &int {
    &x.y
}

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

Recall from the [previous post]({% post_url 2014-01-31-rusts-region-inference-error-diagnostic %}) that this describes a `SubSupConflict`, meaning that lifetime #2 is a sub-lifetime of the lifetime of `&x.y` (which we are trying to infer), and the lifetime of `&x.y` is a sub-lifetime of lifetime #1. However, lifetime #2 is not necessarily a sub-lifetime of lifetime #1, and so we have to explicitly declare that it is. Note that the anonymous lifetime #1 came from `&Foo` in the function parameter, and anonymous lifetime #2 came from the return value `&int`. In this case, we indicate that they are the same lifetime by introducing a lifetime parameter (binding) `'a` and give both the same binding:

```rust
fn bar<'a>(x: &'a Foo) -> &'a int {
    &x.y
}
```


## Terminologies

One potential source of confusion when it comes to lifetime parameter is that it's slightly different from function parameter. Here, it's helpful to introduce a couple terminologies: free lifetime and bound lifetime.

The terms "free" and "bound" came from lambda calculus. For traditional function, a binding is analogous to a parameter, and other variables are considered free. For example, for some function `fn foo(x: int) { y }`, `x` would be a binding and `y` a free variable.

Free lifetime and bound lifetime slightly differ from their traditional function counterparts in the following ways:

1. Return value *can* get a lifetime binding.

2. If a function parameter or return value uses lifetimes, it gets lifetime bindings even when we don't explicitly give it any. A function parameter or return value without a named binding may be given anonymous bindings of the form #*n* for some *n* (this is where the name "anonymous lifetime" came from). Note that just as you can write multiple lifetimes on a type declaration (for example, `&'a &'b Foo<'c>`), there may be multiple anonymous lifetimes associated with a single function parameter or return value.

3. When we typecheck the body of a function, we treat those lifetimes as free instead.

Thus, "free lifetime" and "bound lifetime" are not mutually exclusive; instead, they represent different concepts. The reason why we treat lifetimes from function parameters as free is because they may have arbitrary lifetimes depending on what are passed in (that's why another way to view free lifetime is "one that's unbounded above"). Also, Rust doesn't attempt to infer lifetime of return value, so it's given the same treatment.

What is the point of having lifetime binding, then? Essentially, it serves these two purposes:

* To indicate that a group of function parameters and/or return value have the same lifetime

* To name a lifetime in case of error (though this ends up being not very helpful when a binding is anonymous)


## Case study continued

This section doesn't describe anything new except giving a small peek inside the Rust compiler. Here is Rust's representation of [free lifetime](https://github.com/mozilla/rust/blob/cf9164f94c6a7e3717f67b1fb74a7662639835f0/src/librustc/middle/ty.rs#L507-L510):

```rust
pub struct FreeRegion {
    scope_id: NodeId,
    bound_region: BoundRegion
}
```

Here, `scope_id` refers to the function block the lifetime is bounded to. [BoundRegion](https://github.com/mozilla/rust/blob/cf9164f94c6a7e3717f67b1fb74a7662639835f0/src/librustc/middle/ty.rs#L513-L525) refers to bound lifetime and has the following representation:

```rust
pub enum BoundRegion {
    /// An anonymous region parameter for a given fn (&T)
    BrAnon(uint),

    /// Named region parameters for functions (a in &'a T)
    ///
    /// The def-id is needed to distinguish free regions in
    /// the event of shadowing.
    BrNamed(ast::DefId, ast::Ident),

    /// Fresh bound identifiers created during GLB computations.
    BrFresh(uint),
}
```

From the function that we are studying,

```rust
fn bar(x: &Foo) -> &int {
    &x.y
}
```

the Rust compiler creates the free lifetimes with bindings `BrAnon(0)` and `BrAnon(1)` out of the function parameter and return value, respectively. For brevity, we will refer to the former free lifetime as `free(a#0)` and the latter `free(a#1)`. Also, let `'v` be the lifetime of `&x.y`, which we are trying to infer, and let `a <= b` be the constraint that lifetime `a` has to be a sub-lifetime of lifetime `b`.

As the compiler walks through this function, it creates four constraints. The two that are relevant are `free(a#1) <= 'v` and `'v <= free(a#0)`. At the beginning, it will go through all the constraints where `'v` is the "bigger" lifetime, which in this case is `free(a#1) <= 'v`. At this point, it infers that `'v` also has the lifetime `free(a#1)`. Thus, it follows that `free(a#1) <= free(a#0)`. However, since these lifetimes are free, we cannot be sure how "big" either is, and so it's not necessarily the case that one is a sub-lifetime of the other. As the result, we get an error.


## Improving the error message

From this case study, it occurred to me that all SubSupConflict `sub_r <= ... <= 'v <= ... <= sup_r`, where `sub_r` and `sup_r` are free lifetimes can use the error message "define the same lifetime parameter on `r_sub` and `r_sup`".

This leads me to believe that I now have a strategy to make Idea #1 from my [first post]({% post_url 2014-01-20-some-ideas-for-improving-rusts-lifetime-error-messages %}) a reality. Namely, for functions with similar form to the one above, I want to make a concrete suggestion:

```rust
test.rs:3:5: 3:9 error: cannot infer an appropriate lifetime for borrow
expression due to conflicting requirements
test.rs:3     &x.y
              ^~~~
test.rs:3:5: 3:9 note: consider using an explicit lifetime parameter as shown:
fn foo<'a>(x: &'a Foo) -> &'a int
```

Here is the strategy:

* Check that `sub_r` and `sup_r` are both free lifetimes.

* Check that at least one of `sub_r` or `sup_r` has anonymous binding.

* Somehow identify that they come from a function declaration and obtain the AST of that function declaration.

* Since the compiler appears to identify the first anonymous binding of a function with the number 0 and increment each subsequent one by 1, we may be able to use the same strategy to identify "where" to insert/replace the lifetime in the AST.

* Use the Rust pretty printer to print the new function declaration.


## Questions(s)

Here are some things I'm currently unclear about:

* Suppose that `sub_r` and `sup_r` come from function declaration. Is `sub_r` always the lifetime of the return value?

* Suppose that `sub_r` has a named binding `'a` and `sup_r` has an anonymous binding, and we infer that `sub_r` and `sup_r` should have the same binding, would there be a case where giving `sup_r` a binding `'a` would cause a new error?
