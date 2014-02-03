---
layout: post
title: "Rust's Region Inference Error Diagnostic"
date: 2014-01-31 19:00:20 -0500
comments: true
categories: [Rust, Independent Study]
---

My [PR](https://github.com/mozilla/rust/pull/11718/) that implements ideas #2 and #3 of [previous post]({% post_url 2014-01-20-some-ideas-for-improving-rusts-lifetime-error-messages %}) was accepted last week, so earlier this week I set out to do idea #1. That is, I want to simplify the error message for the following code snippet:

```rust
struct Foo { y: int }
fn bar(x: &Foo) -> &int {
    &x.y
}
```

It turns out that the error diagnostic for this case does not lie in the borrow checker but the region inference system (where "region" is synonymous to "lifetime"). Thus, I spent Monday and Tuesday reading the codes inside `rustc::middle::typeck::infer`. I felt quite down by the end of it, though, because I couldn't figure out a straightforward way to detect the common pattern above, and I was at a loss of what to do. The purpose of this post is to sort out my thinking, console myself, and document some of what I have learned so far.

## A brief description

The compiler's [documentation](https://github.com/mozilla/rust/blob/5a618129b842f875dac5531ce7b8385fe4fcda6c/src/librustc/middle/typeck/infer/region_inference/doc.rs) contains a nice description of how region inference system works. On the contrary, this description will be brief and omit many details. Its main purpose is to introduce some terminologies.

The basic problem is that many times the compiler has to infer the lifetime of certain expressions. When that happens, it creates a "region variable". By contrast, a "concrete region" may be a lifetime associated with some lexical scope (e.g. block of a function) or a free lifetime (I don't quite get what this means, but it appears to refer to a lifetime that's not bounded above). What the compiler does is that as it walks through a function, it accumulates "constraints", and then it tries to solve those constraints by the end of the function. A constraint has the form `constraint(a, b)`, meaning that `a` is a subregion of (i.e., bounded by) `b`, where `a` and `b` may either be a region variable or a concrete region. The compiler would report the error if these constraints happen to conflict.

## Types of error

When the compiler runs through these constraints and deduces region inference errors, it collects them and then reports them later. Region inference errors are categorized into three types, as described by the [RegionResolutionError](https://github.com/mozilla/rust/blob/5a618129b842f875dac5531ce7b8385fe4fcda6c/src/librustc/middle/typeck/infer/region_inference/mod.rs#L62-L85) enum:

```rust
pub enum RegionResolutionError {
    /// `ConcreteFailure(o, a, b)`:
    ///
    /// `o` requires that `a <= b`, but this does not hold
    ConcreteFailure(SubregionOrigin, Region, Region),

    /// `SubSupConflict(v, sub_origin, sub_r, sup_origin, sup_r)`:
    ///
    /// Could not infer a value for `v` because `sub_r <= v` (due to
    /// `sub_origin`) but `v <= sup_r` (due to `sup_origin`) and
    /// `sub_r <= sup_r` does not hold.
    SubSupConflict(RegionVariableOrigin,
                   SubregionOrigin, Region,
                   SubregionOrigin, Region),

    /// `SupSupConflict(v, origin1, r1, origin2, r2)`:
    ///
    /// Could not infer a value for `v` because `v <= r1` (due to
    /// `origin1`) and `v <= r2` (due to `origin2`) and
    /// `r1` and `r2` have no intersection.
    SupSupConflict(RegionVariableOrigin,
                   SubregionOrigin, Region,
                   SubregionOrigin, Region),
}
```

As described by the comments, this is roughly what these errors correspond to:

* `ConcreteFailure` - there is a `constraint(a, b)`, where `a` and `b` are concrete regions, that does not hold

* `SubSupConflict` - there are `constraint(sub_r, v)`, `constraint(v, sup_r)`, where `sub_r` and `sup_r` are concrete regions and `v` is a region variable. Since `sub_r` is a subregion of `v` and `v` is a subregion of `sup_r`, it follows that `sub_r` is a subregion of `sup_r`. However, that constraint is not satisfied.

* `SupSupConflict` - there are `constraint(v, r1)` and `constraint(v, r2)`. Since `v` is a subregion of both `r1` and `r2`, they must overlap. However, they do not.

Recall that we create a region variable when we need to infer the lifetime of some expression. Here a `RegionVariableOrigin` is a type used to record *why* we created the region variable in the first place. On the other hand, `SubregionOrigin` records why we created the constraint. Thus, suppose some region variable `v` has the RegionVariableOrigin `v_origin`, then `SubSupConflict(v_origin, sub_origin, sub_r, sup_origin, sup_r)` encodes the following information:

* `v_origin` - why the region variable `v` is created

* `sub_origin` - why we created `constraint(sub_r, v)`

* `sup_origin` - why we created `constraint(v, sup_r)`

## Case study

Compiling the function at the beginning of this post gives us the following error:

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

It turns out that the error above falls into the `SubSupConflict` category. In general, `SubSupConflict` error message contains one error message, four notes, and has the following format (as an aside, `SupSupConflict` has a similar format):

```rust
error: error message
note: description that lifetime of region variable v (that we want to infer) is bounded by sup_region
note: description of why constraint(v, sup_region) is created
note: description that lifetime of sub_region is bounded by v
note: description of why constraint(sub_region, v) is created
```

The error message (+ notes) above has the following deficiencies:

* It's too long and intimidating

* The description is fairly opaque

* Even though it's long, **it does not even describe the problem completely**

To elaborate on the last bullet point, the description of the problem is this: 1) `sub_region` is subregion of `v`, 2) `v` is subregion of `sup_region`, 3) thus, `sub_region` is subregion of `sup_region`, but that does not hold. As we can see, number 3 is missing.

## Suggestion

My suggestion is to add a note to include 3. Adding another note, however, would make an already long error message even longer. Personally, I feel that since the second and fourth notes do not describe the problem directly, they are of secondary importance and should be removed. I would also swap the first and third notes and change the language a bit to make it a smoother reading experience.

Putting all the above together, we would have something as follows:

```rust
test.rs:3:5: 3:9 error: cannot infer an appropriate lifetime 'v for borrow expression due to conflicting requirements
test.rs:3     &x.y
              ^~~~
test.rs:2:25: 4:2 note: first, the anonymous lifetime #2 (defined on the block at 2:24) is bounded by 'v
test.rs:2 fn bar(x: &Foo) -> &int {
test.rs:3     &x.y
test.rs:4 }
test.rs:2:25: 4:2 note: also, 'v is bounded by the anonymous lifetime #1 (defined on the block at 2:24)
test.rs:2 fn bar(x: &Foo) -> &int {
test.rs:3     &x.y
test.rs:4 }
note: however, the anonymous lifetime #2 is not bounded by the anonymous lifetime #1
```

Instead of "is bounded by", something like "is a sub-lifetime of" may be good, too.

## Discussions and caveats

The example above turns out to be rather silly since anonymous lifetimes #1 and #2 seem to refer to the same block, and yet there's conflict. I tried to investigate how this error arose in the first place by looking at the debug output, but there are too many details I do not understand as of now. I'll try to enumerate some of them in a later section.

Also, I'm not sure if removing the second and fourth notes (about why the constraints are there) is a good idea. I personally wouldn't miss them since I have never found them helpful, but someone more knowledgeable about Rust's lifetime inference may. A solution to this would be to have a verbose option for the power users.

For long block, I would probably replace the current span note with the custom span note that I added in my PR from last week. For example, suppose our function is more than 6 lines long:

```rust
fn foo(x: &Foo) -> &int {
    2;
    3;
    4;
    5;
    6;
    &x.bar
}
```

What the default span note does when displaying a span of more than 6 lines is to strip out all the remaining lines, which looks like this:

```
test.rs:2:25: 9:2 note: first, the lifetime cannot outlive the anonymous lifetime #1 defined on the block at 2:24...
test.rs:2 fn foo(x: &Foo) -> &int {
test.rs:3     2;
test.rs:4     3;
test.rs:5     4;
test.rs:6     5;
test.rs:7     6;
          ...
```

However, this does not give a good view of the whole scope of the lifetime. What my custom span note does is display the first and last lines, and blank out the middle (currently it also always add an arrow at the end, so I'll have to modify it a bit). One added advantage is that it would make the error message takes less space:

```rust
test.rs:2:25: 9:2 note: first, the lifetime cannot outlive the anonymous lifetime #1 defined on the block at 2:24...
test.rs:2 fn foo(x: &Foo) -> &int {
...
test.rs:9 }
```

Regrettably, all this is a far cry from giving a concrete feedback like "missing a lifetime parameter" or "you may need to insert a lifetime" (that said, one thing that I wonder lately is: are all SubSupConflict and SupSupConflict errors caused by missing lifetime parameter?), but I will need to study up more on lifetime, which seems to include a lot of subtle details. If possible, I would like to just suggest outright "perhaps you mean to declare `fn bar<'a>(x: &'a Foo) -> &'a int`?"

## Things I still need to understand

This is the section where I get to wail like a baby and lament about all that is wrong with the world. The data structures are pretty well-commented, but since there are so many details, I end up getting confused. For example, this is what represents a [region](https://github.com/mozilla/rust/blob/5a618129b842f875dac5531ce7b8385fe4fcda6c/src/librustc/middle/ty.rs#L460-L495) (comments removed):

```rust
pub enum Region {
    ReEarlyBound(ast::NodeId, uint, ast::Ident),
    ReLateBound(ast::NodeId, BoundRegion),
    ReFree(FreeRegion),
    ReScope(NodeId),
    ReStatic,
    ReInfer(InferRegion),
    ReEmpty,
}
```

What is a "region bound", and once again, what exactly is a free region? The `FreeRegion` enum also has a `BoundRegion` associated with it: why is that the case?

Here is the signature for [SubregionOrigin](https://github.com/mozilla/rust/blob/5a618129b842f875dac5531ce7b8385fe4fcda6c/src/librustc/middle/typeck/infer/mod.rs#L144-L195):

```rust
pub enum SubregionOrigin {
    Subtype(TypeTrace),                        // (*)
    InfStackClosure(Span),                     // (*)
    InvokeClosure(Span),                       // (*)
    DerefPointer(Span),
    FreeVariable(Span),                        // (*)
    IndexSlice(Span),                          // (*)
    RelateObjectBound(Span),
    Reborrow(Span),                            // (*)
    ReferenceOutlivesReferent(ty::t, Span),
    BindingTypeIsNotValidAtDecl(Span),         // (*)
    CallRcvr(Span),                            // (*)
    CallArg(Span),                             // (*)
    CallReturn(Span),                          // (*)
    AddrOf(Span),
    AutoBorrow(Span),                          // (*)
}
```

This means that there is a ton of specific reasons for a constraint to be added. The ones that are marked with stars are those I'm not quite clear on yet. Also, it seems that this enum serves the dual purpose of indicating why a constraint is added and the cause of error (in particular, `ReferenceOutlivesReferent` and `BindingTypeIsNotValidAtDecl` look like they are for error reporting).

There are also many variants of the `RegionVariableOrigin` that I do not understand, but I think they will be clearer once I know what a region bound is.

## Moving forward

I expect many of these questions will become clearer as I read the code more, but unfortunately it's a slow process. I'm not quite clear on what to do now. Maybe I can start implementing the suggestions I made in this post.