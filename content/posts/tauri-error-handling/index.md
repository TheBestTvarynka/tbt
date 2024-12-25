+++
title = "Tauri error handling recipes"
date = 2024-12-26
draft = false

[taxonomies]
tags = ["recipes", "rust", "tauri", "leptos"]

[extra]
keywords = "Rust, Tauri, Leptos"
toc = true
+++

# Intro

I'm developing the desktop app to meet my own needs: [Dataans](https://github.com/TheBestTvarynka/Dataans).

> Take notes in the form of markdown snippets grouped into spaces.

I use [Tauri](https://tauri.app/) as the main framework and [Leptos](https://leptos.dev/) as the frontend. I am satisfied with the developing experience and the result I got.

During the initial implementation (PoC phase), I didn't implement any error handling and called `.unwrap()` every time. However, after the first release, I needed to add proper error handling and make the app more fault-tolerant. The refactoring process wasn't easy and I spent more time than expected. I learned some important lessons and decided to write this article to summarize my knowledge.

_**Disclaimer**_: I have experience using `Tauri` only with `Leptos` and other Rust-based frontend frameworks. Error handling approaches may be different when using JS-based frontend frameworks.

# Let's handle it

I set up a simple project (basically it is a `cargo create-tauri-app` with minimal changes) to demonstrate the approach and the progress: [github/TheBestTvarynka/trash-code/tauri-error-handling](https://github.com/TheBestTvarynka/trash-code/tree/feat/tauri-error-handling/tauri-error-handling). This article consists of continuous steps of problem-solving. I left a corresponding commit link for each step in this article. So, you can follow this process and even reproduce it with me.

So, what do we have? We have one simple command:

```Rust
// src-tauri/src/lib.rs
#[tauri::command]
fn greet(name: &str) -> String {
    format!("Hello, {}! You've been greeted from Rust!", name)
}

// src/backend.rs
pub async fn greet(name: &str) -> String {
    let args = to_value(&GreetArgs { name }).unwrap();
    
    invoke("greet", args).await.as_string().unwrap()
}
```

## Naive approach

But usually, most of the commands may fail. We may want to return a `Result` and handle the error on the frontend side. Let's try it. Suppose we need to validate the name. I came up with the following implementation (let's keep it simple):

```Rust
use thiserror::Error;

#[derive(Debug, Error)]
enum Error {
    #[error("invalid name: {0}")]
    InvalidName(String),
}

fn validate_name(_name: &str) -> Result<(), Error> {
    Err(Error::InvalidName("Tbt".into()))
}

#[tauri::command]
fn greet(name: &str) -> Result<String, Error> {
    validate_name(name)?;

    Ok(format!("Hello, {}! You've been greeted from Rust!", name))
}
```

Unfortunately, now we have a compilation error:

```
error[E0599]: the method `blocking_kind` exists for reference `&Result<String, Error>`, but its trait bounds were not satisfied
   --> src-tauri/src/lib.rs:14:1
    |
5   | enum Error {
    | ---------- doesn't satisfy `Error: Into<InvokeError>`
...
14  | #[tauri::command]
    | ^^^^^^^^^^^^^^^^^ method cannot be called on `&Result<String, Error>` due to unsatisfied trait bounds
...
25  |         .invoke_handler(tauri::generate_handler![greet])
    |                         ------------------------------- in this macro invocation
    |
   ::: /home/pavlo-myroniuk/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/core/src/result.rs:527:1
    |
527 | pub enum Result<T, E> {
    | --------------------- doesn't satisfy `Result<std::string::String, Error>: IpcResponse` or `_: ResultKind`
    |
    = note: the following trait bounds were not satisfied:
            `Error: Into<InvokeError>`
            which is required by `Result<std::string::String, Error>: tauri::ipc::private::ResultKind`
            `Result<std::string::String, Error>: IpcResponse`
            which is required by `&Result<std::string::String, Error>: tauri::ipc::private::ResponseKind`
```

:thinking:. Our `Result<String, Error>` type must implement the `IpcResponse` trait. Implementing `serde::Serialize` should be enough (because of the `impl<T: Serialize> IpcResponse for T` trait bound. [docs](https://docs.rs/tauri/latest/tauri/ipc/trait.IpcResponse.html#impl-IpcResponse-for-T)).

```Rust
#[derive(Debug, Error, Deserialize, Serialize)]
enum Error {
    #[error("invalid name: {0}")]
    InvalidName(String),
}
```

Great. Compilation successful. Alternatively, you can read another explanation and example: [The Tauri Documentation WIP/Inter-Process Communication#error-handling](https://jonaskruckenberg.github.io/tauri-docs-wip/development/inter-process-communication.html#error-handling).

_**Lesson 1**_. All `Error`s must implement the `serde::Serialize` trait.

What about frontend? How are we going to handle the error? Let's start with the simplest solution:

1. Move the `Error` type to a separate crate `common` which can be used on frontend and backend.
2. Expect `JsValue` to be parsed into `Result<String, Error>`.

```Rust
pub async fn greet(name: &str) -> Result<String, Error> {
    let args = to_value(&GreetArgs { name }).expect("GreetArgs to JsValue should not fail");

    let js_value = invoke("greet", args).await;

    from_value(js_value).expect("JsValue to Result should not fail")
}
```

Oops, we have a problem in runtime:

![](./err-exception.png)

:sob: But why? According to the documentation ([v2.tauri.app/calling-rust/#error-handling](https://v2.tauri.app/develop/calling-rust/#error-handling)), Tauri command error will be an exception on frontend :confused:. So, we should not expect the `Result<String, Error>`, but handle a JS exception instead.

OR we can ask `wasm_bindgen` to handle this exception. Tauri docs say nothing about it, but if we open the wasm bindgen documentation instead, we will find this treasure ([wasm-bindgen/attributes/on-js-imports/catch](https://rustwasm.github.io/wasm-bindgen/reference/attributes/on-js-imports/catch.html)):

> The `catch` attribute allows catching a JavaScript exception. This can be attached to any imported function or method, and the function must return a `Result` where the `Err` payload is a `JsValue`:

:astonished:. Let's use it. Now I'm going to change the binding generated by `cargo create-tauri-app`.

```Rust
#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_namespace = ["window", "__TAURI__", "core"], catch)]
    async fn invoke(cmd: &str, args: JsValue) -> Result<JsValue, JsValue>;
}
```

And now we can handle the exception as an error:

```Rust
pub async fn greet(name: &str) -> Result<String, Error> {
    let args = to_value(&GreetArgs { name }).expect("GreetArgs to JsValue should not fail");

    match invoke("greet", args).await {
        Ok(msg) => Ok(from_value(msg).unwrap()),
        Err(err) => Err(from_value(err).unwrap()),
    }
}
```

And finally, it works :tada:. Here is the resulting code: [github/TheBestTvarynka/trash-code/9f6437fe7799e87439430e483c92c0ed590b12f3/tauri-error-handling](https://github.com/TheBestTvarynka/trash-code/tree/9f6437fe7799e87439430e483c92c0ed590b12f3/tauri-error-handling).

_**Lesson 2**_. Tauri command error is an exception on frontend. The exception can be caught by using the `catch` attribute of the `#[wasm_bindgen]` macro.

# Lessons we learned

1. All `Error`s must implement the `serde::Serialize` trait.
2. Tauri command error is an exception on frontend. The exception can be caught by using the `catch` attribute of the `#[wasm_bindgen]` macro.

# Doc, references, code

* [The Tauri Documentation WIP/Inter-Process Communication#error-handling](https://jonaskruckenberg.github.io/tauri-docs-wip/development/inter-process-communication.html#error-handling).
* [v2.tauri.app/calling-rust/#error-handling](https://v2.tauri.app/develop/calling-rust/#error-handling).
* [wasm-bindgen/attributes/on-js-imports/catch](https://rustwasm.github.io/wasm-bindgen/reference/attributes/on-js-imports/catch.html).
