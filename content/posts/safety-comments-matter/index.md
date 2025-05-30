+++
title = "Safety Comments Matter"
date = 2025-06-01
draft = false

[taxonomies]
tags = ["rust"]

[extra]
keywords = "Rust"
toc = true
# thumbnail = "thumbnail.png"
+++

TL;DR: This article is highly inspired by [this GitHub comment](https://github.com/Devolutions/sspi-rs/pull/430#pullrequestreview-2850078587).

## Safety Comments Matter

`SAFETY` comments are always welcome when dealing with `unsafe` code in Rust. The clippy even has a lint to enforce writing them: [rust-lang.github.io/rust-clippy/#undocumented_unsafe_blocks](https://rust-lang.github.io/rust-clippy/master/#undocumented_unsafe_blocks):

```Rust
#![warn(clippy::undocumented_unsafe_blocks)]
```

You can find such comments in many crates and std. [Example](https://doc.rust-lang.org/src/core/ptr/non_null.rs.html#388):

```Rust
// SAFETY: `NonNull` is `transparent` over a `*const T`, and `*const T`
// and `*mut T` have the same layout, so transitively we can transmute
// our `NonNull` to a `*mut T` directly.
unsafe { mem::transmute::<Self, *mut T>(self) }
```



## Put A Finger Down

Let's play [Put A Finger Down](https://officialgamerules.org/game-rules/put-a-finger-down/) game. The rule is simple: you put one finger down for every true statement about your unsafe Rust code.

If you put down zero fingers, my congratulations 🎊. You are probably aware of what you are doing, and you can sleep peacefully. If you put down all fingers, then, please, reconsider your life choices 🙂

:ballot_box_with_check: Rust 2018 edition or older.

:ballot_box_with_check: `unsafe` blocks without `SAFETY` comments.

:ballot_box_with_check: Many unsafe operations per one unsafe block.

:ballot_box_with_check: Your unsafe code has never been run using [Miri](https://tbt.qkation.com/posts/miri/).

:ballot_box_with_check: Raw pointer <-> `int` cast is a usual thing.

The reader understands that these statements mean something bad for your unsafe code. Let's discuss each of them. I want to make sure that we are on the same page.

### Rust 2018 edition or older

> _Why is the Rust edition important in our case?_

(I already described it [here](https://t.me/tbtpm/348).)

The 2024 edition brought many important features related to `unsafe` code and FFI.

* [`unsafe_op_in_unsafe_fn` warning](https://doc.rust-lang.org/nightly/edition-guide/rust-2024/unsafe-op-in-unsafe-fn.html).

```rust
unsafe fn set_id(id: u8, dest: *mut u8) {
    // warning[E0133]: dereference of raw pointer is unsafe and requires unsafe block
    *dest = id;
    // Solution:
    unsafe { *dest = id; }
}
```

I like it a lot. Because in huge `unsafe` functions, it's not clear which operations are unsafe and which are not. 

* [Unsafe attributes](https://doc.rust-lang.org/nightly/edition-guide/rust-2024/unsafe-attributes.html).

```rust
// Before Rust 2024 the following code was accepted:
#[no_mangle]
pub fn tbt() {}
// In Rust 2024 you need to add the unsafe attribute:
#[unsafe(no_mangle)]
pub fn tbt() {}
```

Starting with the 2024 Edition, it is now required to mark these attributes as unsafe. This one applies to [export_name](https://doc.rust-lang.org/nightly/reference/abi.html#the-export_name-attribute), [link_section](https://doc.rust-lang.org/nightly/reference/abi.html#the-link_section-attribute), and [no_mangle](https://doc.rust-lang.org/nightly/reference/abi.html#the-no_mangle-attribute).
It is crucial because previously the app could crash even if it contained only safe code:

```rust
fn main() {
    println!("Hello, world!");
}
#[export_name = "malloc"]
fn foo() -> usize { 1 }
```

* [Disallow references to static mut](https://doc.rust-lang.org/nightly/edition-guide/rust-2024/static-mut-references.html).

```rust
static mut X: i32 = 23;
unsafe {
    // ERROR: shared reference to mutable static
    let y = &X;
}
```

> Merely taking such a reference in violation of Rust's mutability XOR aliasing requirement has always been instantaneous undefined behavior,
> even if the reference is never read from or written to. 

* [Unsafe extern blocks](https://doc.rust-lang.org/nightly/edition-guide/rust-2024/unsafe-extern.html).

> Adding the unsafe keyword helps to emphasize that it is the responsibility of the author of the extern block to ensure that the signatures are correct.

It produces a warning in Rust 2021, but in Rust 2024 it results in error.

Now you understand why the 2024 edition is preferred in unsafe code :smiley:.

### `unsafe` blocks without `SAFETY` comments

The whole first part of the article is about it. You should already understand the importance of such comments in the code.

### Many unsafe operations per one unsafe block

It is partially coveted by 2024 edition, but still can violated:

```rust
// Compiles without any warnings.
unsafe fn tbt(buf: *mut *mut u8, len: *mut u32) {
    unsafe {
        *buf = null_mut();
        *len = 0;
    }
}
```

It is not recommended (by me and some other dudes :wink:) to have multiple unsafe operations in one `unsafe` block.
Because it becomes much harder to keep the code safe and follow all safety preconditions.
Every unsafe operation should have its own `unsafe` block and `SAFETY` comment above it.

TODO: example?

### Miri

I already described all pros and cons of Miri here: [https://tbt.qkation.com/posts/miri/](https://tbt.qkation.com/posts/miri/). Please, read this article so I don't need to repeat myself.

TL;DR: if you have many unsafe operations, then consider using Miri. Miri is awesome and will help you to prevent a lot of bugs.

### Pointer <-> integer cast

Oh, this one deserves a separate article, but I will try to fit it in a few sentences.

I'm still convinced that the Rust documentation is the best place to learn about unsafe. To your attention, quotes from the [`std::ptr`](https://doc.rust-lang.org/std/ptr/#provenance) module level documentation:

> **Pointers are not simply an “integer” or “address”.** ...pointers need to somehow be more than just their addresses: they must have provenance. Provenance can affect whether a program has undefined behavior:
> * It is undefined behavior to access memory through a pointer that does not have provenance over that memory.
> * It is undefined behavior to [`offset`](https://doc.rust-lang.org/std/primitive.pointer.html#method.offset) a pointer across a memory range that is not contained in the allocated object it is derived from, or to [`offset_from`](https://doc.rust-lang.org/std/primitive.pointer.html#method.offset_from) two pointers not derived from the same allocated object. Provenance is used to say what exactly “derived from” even means: the lineage of a pointer is traced back to the Original Pointer it descends from, and that identifies the relevant allocated object. In particular, it’s always UB to offset a pointer derived from something that is now deallocated, except if the offset is 0.

[and](https://doc.rust-lang.org/std/ptr/#pointers-vs-integers):

> From this discussion, it becomes very clear that a `usize` cannot accurately represent a pointer, and converting from a pointer to a `usize` is generally an operation which only extracts the address. Converting this address back into pointer requires somehow answering the question: which provenance should the resulting pointer have?
>
> Rust provides two ways of dealing with this situation: _Strict Provenance_ and _Exposed Provenance_.

There is a lot more to say. But I think you can read the Rust doc by yourself. My point is that you should never cast a pointer to an integer and vice versa. Ot if you even dare to do it, then at least use an appropriate API: [`expose_provenance`](https://doc.rust-lang.org/std/primitive.pointer.html#method.expose_provenance) and [`with_exposed_provenance`](https://doc.rust-lang.org/std/ptr/fn.with_exposed_provenance.html).

## Conclusion

Pointers are not simply an "integer" or "address" :slightly_smiling_face:.