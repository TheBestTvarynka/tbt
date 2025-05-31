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

You can find such comments in many crates and std. [Example](https://github.com/rust-lang/rust/blob/7a7bcbbcdbf2845164a94377d0e0efebb737ffd3/library/core/src/ptr/non_null.rs#L391-L394):

```Rust
// SAFETY: `NonNull` is `transparent` over a `*const T`, and `*const T`
// and `*mut T` have the same layout, so transitively we can transmute
// our `NonNull` to a `*mut T` directly.
unsafe { mem::transmute::<Self, *mut T>(self) }
```

Such comments help us to understand safety preconditions and invariants of the unsafe operation, how they are upheld. Devs usually tend to treat them as one another boring thing in development process. I kinda feel the same, but benefits of properly written `unsafe` blocks are almost immeasurable.

But what is _"a properly written `unsafe` block"_? A properly written `unsafe` block has the following characteristics:

* Has a single unsafe operation.
* Documented using `SAFETY:` comment. In turn, to write a proper `SAFETY:` comment, you must follow the following requirements ([src](https://github.com/Devolutions/sspi-rs/pull/430#pullrequestreview-2850078587) of the quote below):

> * Explain why the code is "sound", detailing the invariants and preconditions that are maintained.
> * Avoid assumptions without justifications. 
>   * Do not use phrases like _"we assume this is safe because…"_ or _"this should be safe"_.
>   * Provide concrete evidence or reasoning that justifies the safety of the operation.
> * Explicitly reference type invariants and prior checks when safety relies on it.
>   ```rust
>   // SAFETY: index is guaranteed to be within bounds due to the prior check.
>   unsafe { array.get_unchecked(index) };
>   ```
> * Address each safety requirement individually, using a bullet list.
>   ```rust
>   // SAFETY:
>   // - ptr is non-null and properly aligned.
>   // - The memory region ptr points to is valid for reads of len elements.
>   unsafe { std::slice::from_raw_parts(ptr, len) };
>   ```
> * Document FFI boundaries clearly, but if the only safety requirements are standard (e.g.: non-dangling pointers), for consistency, use the following concise comment verbatim:
>   ```rust
>   // SAFETY: FFI call with no outstanding preconditions.
>   unsafe { straightforward_ffi_function(ptr) };
>   ```
> * Do not document how the resulting value is safely used later.
>   * Really focus only on why the current unsafe operation is sound.
>   * Subsequent operations should be documented as appropriate in the code.
> * It’s welcomed to document how and why the function is used, but this should not be part of the `SAFETY:` comment. 
>     * Write a separate paragraph out of the safety comment, typically above the safety comment.
>       ```rust
>       // Passing None to GetModuleHandleW requests the handle to the
>       // current process main executable module
>       //
>       // SAFETY: FFI call with no outstanding preconditions.
>       let h_module: HMODULE = unsafe { GetModuleHandleW(None)? };
>       ```
> * To keep the comment concise, do not use phrases like _"this is safe because…"_. Go straight to the point (e.g.: _"pointer is guaranteed not null due to the prior check"_).
> * For readability and clarity, keep the `unsafe` block as small as possible, and untangled from other safe code.  If needed, use intermediate variables.
> ```rust
> // SAFETY: …
> let intermediate_variable = unsafe { transform_credentials_handle(credentials_handle) };
> match intermediate_variable { … }
> ```
> * If a precondition must be upheld by the caller, mark the function as unsafe and document it with a `# Safety` section listing the invariants.
> ```rust
> /// Returns the length in characters of the C-string.
> ///
> /// # Safety
> ///
> /// - `s` must be a valid, null-terminated C-string.
> unsafe fn strlen(s: *const u8) -> usize { … }
> ```

:face_exhaling:. I hope you didn't get tired at this point. It is true that writing a good `SAFETY:` comment can be hard and exhausting. But it is worth it.

For me, the most important point in `SAFETY:` comments is that writing them requires thinking about preconditions and invariants. Devs seldom think about invariants and what can go wrong. When explaining why the `unsafe` operation is safe, you check the corresponding unsafe function preconditions and how to uphold them.

You can use AI to help you write `SAFERY:` comments, but remember that **it is your responsibility** to check that the generated ones list all needed preconditions and invariants.

**Do not neglect `SAFETY:` comments.**

### Example

[As Linus Torvalds said](https://lkml.org/lkml/2000/8/25/132),

> Talk is cheap. Show me the code.

So, let's show some code. Suppose we have the following `unsafe` code:

```rust
unsafe fn do_some_job(context: PCtxtHandle, buf: *mut u8, len: *mut u32, attrs: *mut u32) -> u32 {
    if buf.is_null() || len.is_null() || attrs.is_null() {
        return 1;
    }

    let out_data = /* do the job */;
    let out_attrs = /* Attributes::... */;

    unsafe {
        from_raw_parts_mut(buf, out_data.len()).copy_from_slice(&out_data);
        *len = out_data.len().try_into().unwrap();
        *attrs = out_attrs.bits();
    }

    0
}
```

The function above is pretty simple. It does some job and saves the resulting buffers and attributes into the out parameters. Now let's make it better. I know we can.

```rust
/// Does some job.
///
/// # SAFETY:
///
/// - The `context` pointer must be valid non-null pointer to the application context and obtained using the [init_context] function.
/// - The `len` and `attrs` pointers must be non-null.
/// - The `buf` pointer must be non-null. The entire memory range behind it must be contained within a single allocated object,
///   be properly initialized and aligned.
unsafe fn do_some_job(context: PCtxtHandle, buf: *mut u8, len: *mut u32, attrs: *mut u32) -> u32 {
    if buf.is_null() || len.is_null() || attrs.is_null() {
        return 1;
    }

    let out_data = /* do the job */;
    let out_attrs = /* Attributes::... */;

    // SAFETY:
    // - The `buf` pointer is not null due to the prior check.
    // - We create only one slice at a time, so slice do not alias any other references.
    // - The caller must ensure that the memory is properly initialized and aligned.
    let buf = unsafe { from_raw_parts_mut(buf, out_data.len()) };
    buf.copy_from_slice(&out_data);

    let out_len = out_data.len().try_into().unwrap();
    // SAFETY:
    // The `len` pointer is not null due to the prior check.
    unsafe {
        *len = out_len;
    }

    let out_attrs = out_attrs.bits();
    // SAFETY:
    // The `attrs` pointer is not null due to the prior check.
    unsafe {
        *attrs = out_attrs;
    }

    0
}
```

Much better, right? Let's recall what has changed:

* Added function doc comment with `# Safety` requirements.
* Split unsafe operations into many `unsafe` blocks containing exactly one unsafe operation.
* Every `unsafe` block now has the `SAFETY:` comment.
* Moved safe operation out of the unsafe block (I'm talking about the `.copy_from_slice` method call).

Now the caller knows that it is not enough to just allocate the memory. The memory needs to be initialized. Now the caller knows that `len` and `attrs` pointers cannot be NULL. And more... It sounds obvious, but the unsafe code becomes very dangerous in the blink of an eye.

## Put A Finger Down

Let's play [Put A Finger Down](https://officialgamerules.org/game-rules/put-a-finger-down/) game. _(This section is optional and written mostly for fun)_. The rule is simple: you put one finger down for every true statement about your unsafe Rust code.

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

The whole [first part](#safety-comments-matter) of the article is about it. You should already understand the importance of such comments in the code.

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