+++
title = "Rust Clap recipes"
date = 2023-06-27
draft = false

[taxonomies]
tags = ["clap", "recipes", "rust"]

[extra]
keywords = "Rust, Clap, CLI, Args, Parsing"
toc = true
# mermaid = true
# thumbnail = "sspi-thumbnail.png"
+++

I bet you know but I still wanna recall it. [`Clap`](https://github.com/clap-rs/clap) is a full-featured, fast Command Line Argument Parser for Rust. [docs.rs](https://docs.rs/clap/latest/clap/). [crates.io](https://crates.io/crates/clap).

# Goals

**Goals:**

* Show non-standard examples of the cli arguments configuration.
* Tell you about alternate ways to parse cli args with the `clap`.

**Non-goals:**

* Write the *"ultimate"* guide to the `clap` library.
* Create an introduction (or for newbies) article about the clap.

The current article is basically recipes. It means, here we have described concrete problems, ways how to solve them, and examples of the execution.

# Recipes

## 1. Collecting multiple values

Imagine the following situation, you are writing some server. It can be the server side of some protocol, or proxy server, or something like that. At some point, you decided to add encryption. Now you need to add the user the ability to pass allowed encryption algorithms (ciphers) for server sessions.

Let's try to do it. The first idea that comes to mind is just to use a vector of strings:

```Rust
/// Server config structure
#[derive(Parser, Debug)]
struct Config {
    /// Allowed encryption algorithms list
    #[arg(long, value_name = "ENCRYPTION ALGORITHMS")]
    pub enc_algs: Vec<String>,
}
```

**Note 1:** personally, I prefer the [*derive*](https://docs.rs/clap/latest/clap/_derive/index.html) approach to configure clap instead of a [*builder*](https://docs.rs/clap/latest/clap/index.html) approach.

**Note 2:** for all examples in this article, I use the same `main` function:

```Rust
/// Just prints the server configuration.
/// `Config` is a previously defined structure (like in the example above).
fn main() {
    println!("{:?}", Config::parse());
}
```

It'll work, here is an example:

```bash
./cool-server --enc-algs aes256 --enc-algs des3 --enc-algs thebesttvarynka 

# Config { enc_algs: ["aes256", "des3", "thebesttvarynka"] }
```

Good but here we have a big problem: values are not validated and the user can specify unsupported or just wrong algorithms. It'll be cool if we delegate the algorithm validation to the `clap`.

First of all, we need to implement the `Cipher` enum. Most likely, you'll have such an enum already implemented in the project, but here we need to write it:

```Rust
#[derive(Debug, Clone, clap::ValueEnum)]
pub enum Cipher {
    Aes128,
    Aes256,
    Des3,
}

// The next traits implementations are omitted
// and left for the reader as a homework 😝
//
// impl AsRef<str> for Cipher { ... }
// impl Display for Cipher { ... }
// impl TryFrom<&str> for Cipher { ... }
```

**Pay attention** that we also added the `clap::ValueEnum` derive. It'll generate the [`ValueEnum`](https://docs.rs/clap/latest/clap/trait.ValueEnum.html) trait implementation and `clap` will be able to parse the raw string into concrete enum value. Now time to test it:

```bash
./cool-server --enc-algs aes256 --enc-algs des3
# Config { enc_algs: [Aes256, Des3] }

./cool-server --enc-algs aes256 --enc-algs des3 --enc-algs thebesttvarynka
# error: invalid value 'thebesttvarynka' for '--enc-algs <ENCRYPTION ALGORITHMS>'
#   [possible values: aes128, aes256, des3]
#
# For more information, try '--help'.
```

Very cool :fire:. Clap validates the input for us and even tells the user possible values if smth goes wrong. You can read all code of the example above [here](https://github.com/TheBestTvarynka/trash-code/commit/f5bffb732b3f79a2ef00f83e335ae80125fa0294).

In general, we can finish at this point this recipe, but I used to specify multiple values using a comma-separated string. The perfect arg looks for me like this:

```bash
./cool-server --enc-algs aes256,des3
# or even
./cool-server --enc-algs aes256:des3
```

*"This move will cost us"* some additional code writing :smiling_face_with_tear:. We need to create a wrapper over the `Vec<Cipher>` and implement parsing for it:

```Rust
#[derive(Debug, Clone)]
struct EncAlgorithms(pub Vec<Cipher>);

// Implementation of this trait is left for the reader.
// impl fmt::Display for EncAlgorithms { ... }

fn parse_encryption_algorithms(raw_enc_algorithms: &str) -> Result<EncAlgorithms, Error> {
    let mut parsed_enc_algs = Vec::new();

    for raw_enc_alg in raw_enc_algorithms.split(':').filter(|e| !e.is_empty()) {
        parsed_enc_algs.push(raw_enc_alg.try_into()?);
    }

    if parsed_enc_algs.is_empty() {
        return Err(Error::new(ErrorKind::InvalidValue));
    }

    Ok(EncAlgorithms(parsed_enc_algs))
}
```

And only after this, we can use it in our configuration:

```Rust
/// Server config structure
#[derive(Parser, Debug)]
struct Config {
    /// Allowed encryption algorithms list (separated by ':').
    #[arg(
        long,
        value_name = "ENCRYPTION ALGORITHMS LIST",
        // https://docs.rs/clap/latest/clap/struct.Arg.html#method.value_parser
        value_parser = ValueParser::new(parse_encryption_algorithms),
        default_value_t = EncAlgorithms(vec![Cipher::Aes256, Cipher::Aes128]),
    )]
    pub enc_algs: EncAlgorithms,
}
```

> *Why we should create the wrapper? Why is custom parser + `Vec<Cipher>` not enough?*

Yes, you can write just `Vec<Cipher>` with custom [`value_parser`](https://docs.rs/clap/latest/clap/struct.Arg.html#method.value_parser) and it'll compile. But it'll fail in runtime with the following error:

```txt
thread 'main' panicked at 'Mismatch between definition and access of `enc_algs`. Could not downcast to cool_server::cipher::Cipher, need to downcast to alloc::vec::Vec<cool_server::cipher::Cipher>
', src/main.rs:37:19
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

> *Understandable.*

Let's try to run the implementation with a wrapper:

```bash
./cool-server --enc-algs aes256:des3
# Config { enc_algs: EncAlgorithms([Aes256, Des3]) }

./cool-server --enc-algs aes256:des3:thebesttvarynka
# error: invalid value 'aes256:des3:thebesttvarynka' for '--enc-algs <ENCRYPTION ALGORITHMS LIST>': error: invalid value 'Invalid algorithm name: thebesttvarynka' for 'enc-algs'
# 
# For more information, try '--help'.

./cool-server --help
# Server config structure
# 
# Usage: cool-server [OPTIONS]
# 
# Options:
#       --enc-algs <ENCRYPTION ALGORITHMS LIST>
#           Allowed encryption algorithms list (separated by ':') [default: aes256:aes128]
#   -h, --help
#           Print help
```

The full [src](https://github.com/TheBestTvarynka/trash-code/commit/9154610b4ce2343d38ab51abbbe8b5953b35bd61) code of the example above.

## 2. Tw

## 3. Th
