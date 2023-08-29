+++
title = "Rust Clap recipes"
date = 2023-06-15
draft = false

[taxonomies]
tags = ["clap", "recipes", "rust"]

[extra]
keywords = "Rust, Clap, CLI, Args, Parsing"
toc = true
thumbnail = "clap-recipes-thumbnail.png"
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

## Collecting multiple values

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

> *Okay, but are you sure that we can't do better? I am still thinking about a better solution.*

IDK :confused:. Let's explore the docs more thoroughly... Oh, it turns out it's called the [value_delimiter](https://docs.rs/clap/latest/clap/struct.Arg.html#method.value_delimiter):

```rust
/// Server config structure
#[derive(Parser, Debug)]
struct Config {
    /// Allowed encryption algorithms list (separated by ':' or space).
    #[arg(
        long,
        value_name = "CIPHERS",
        default_values_t = vec![Cipher::Aes256],
        value_delimiter = ':',
        num_args = 1..,        // at least one allowed cipher type
    )]
    pub enc_algs: Vec<Cipher>,
}
```

> *Good job. It's way better now.*

Yep. With this approach, we remove all custom parsing and formatting stuff. Moreover, the help message and error reporting became better:

```bash
./cool-server --help
# Server config structure
# 
# Usage: cool-server [OPTIONS]
# 
# Options:
#       --enc-algs <CIPHERS>...  Allowed encryption algorithms list (separated by ':' or space) [default: aes256] [possible values: aes128, aes256, des3]
#   -h, --help                   Print help

./cool-server --enc-algs aes256:des3
# Config { enc_algs: [Aes256, Des3] }

./cool-server --enc-algs aes256:des4
# error: invalid value 'des4' for '--enc-algs <CIPHERS>...'
#   [possible values: aes128, aes256, des3]
# 
#   tip: a similar value exists: 'des3'
# 
# For more information, try '--help'.
```

The full [src](https://github.com/TheBestTvarynka/trash-code/commit/eae20b208693f6dafeb7910fd8b26ca24987937b) code of the example above.

## Commands and arg groups

Before we start this recipe, I wanna recall one thing: the difference between commands and args. First of all, commands don't have any dashed or slashed in the name. They are just words. Commands specify **what** to do, whereas args specify **how** to do it. Example:

```bash
gcloud auth login --cred-file creds.json
# What to do? `login`.
# How to do it? By taking a file with creds named `creds.json`.
```

In this recipe, we'll work with commands and args. You'll see a more complex example of the `clap` configuration.

> *Interesting. What do we need to configure? :smile:*

We all know the [Imgur](https://imgur.com/) site (if not, then just visit). It has an [API](https://api.imgur.com/). Let's imagine that we decided to write the CLI tool that helps us to work with the *Imgur* site using its API. So, now we need to design the tool interface. We do not plan to cover the whole API. Just downloading and uploading. A quick draft configuration:

```rust
#[derive(Debug, Clone, Subcommand)]
enum Command {
    Upload,
    Download,
}

/// Img tool config structure
#[derive(Parser, Debug)]
struct Config {
    /// command to execute
    #[command(subcommand)]
    pub command: Command,

    /// Path to the api key file
    #[arg(long, env = "API-KEY")]
    pub api_key: PathBuff,
}
```

We immediately have a few interesting moments: the [Subcommand](https://docs.rs/clap/latest/clap/trait.Subcommand.html) trait derive and the `PathBuff` type in the `api_key` field. We are not forced to use only simple types for args like `String`s, numbers, etc. If you have a concrete type that describes your value ([`PathBuff`](https://doc.rust-lang.org/stable/std/path/struct.PathBuf.html) for file path, [`url::Url`](https://docs.rs/url/latest/url/struct.Url.html) for urls, or even custom ones), then use this type in the configuration. It'll handle more errors during parsing and make further work easier.

Now we add params for downloading command:

```rust
#[derive(Debug, Clone, Subcommand)]
enum Command {
    Upload,
    Download {
        /// Source image link
        #[arg(long)]
        link: Url,  // <--- Pay attention to the type. We use the `Url` here and not the `String`.

        /// Path for the image
        #[arg(long)]
        dest_file: PathBuf,
    }
}
```

To download the image we need only two things: the source image link and the destination file path. This is how it works:

```bash
./img-tool download --help
# Usage: img-tool --api-key <API_KEY> download --link <LINK> --dest-file <DEST_FILE>
# 
# Options:
#       --link <LINK>            Source image link
#       --dest-file <DEST_FILE>  Path for the image
#   -h, --help                   Print help

./img-tool --api-key key.json download --link https://imgur.com/gallery/vNOUshX --dest-file ferris.png
# Config { command: Download { link: Url { scheme: "https", cannot_be_a_base: false, username: "", password: None, host: Some(Domain("imgur.com")), port: None, path: "/gallery/vNOUshX", query: None, fragment: None }, dest_file: "ferris.png" }, api_key: "key.json" }

./img-tool --api-key key.json download --link :imgur.com/gallery/vNOUshX --dest-file ferris.png
# error: invalid value ':imgur.com/gallery/vNOUshX' for '--link <LINK>': relative URL without a base
# 
# For more information, try '--help'.
```

Cool and pretty simple. But the upload is a little bit more complex. We wanna have two options where take the photo to upload: file image on the device or any public URL on the Internet. Here is the configuration for the upload command:

```rust
#[derive(Debug, Clone, Args)]
#[group(required = true, args = ["file", "link"])]
/// Possible types of the file source
struct FileSource {
    /// Path to the image on the device
    #[arg(long)]
    file: Option<PathBuf>,
    
    /// Url to the image on the Internet
    #[arg(long)]
    link: Option<Url>,
}

#[derive(Debug, Clone, Subcommand)]
enum Command {
    Upload {
        /// File to upload
        #[command(flatten)]
        file_source: FileSource,
        
        /// Folder for the image on the site
        #[arg(long)]
        folder: String,
    },
    Download { /* omitted */ },
}
```

Okay, we have two file sources: file or link. And we use [ArgGroup](https://docs.rs/clap/latest/clap/struct.ArgGroup.html) to specify that the user must specify only one of them: either file or link. And here is the demo:

```bash
./img-tool upload --help
# Possible types of the file source
# 
# Usage: img-tool --api-key <API_KEY> upload --folder <FOLDER> <--file <FILE>|--link <LINK>>
# 
# Options:
#       --file <FILE>      Path to the image on the device
#       --link <LINK>      Url to the image on the Internet
#       --folder <FOLDER>  Folder for the image on the site
#   -h, --help             Print help

./img-tool --api-key key.json upload --folder ferris --file crab_ferris.png
# Config { command: Upload { file_source: FileSource { file: Some("crab_ferris.png"), link: None }, folder: "ferris" }, api_key: "key.json" }

./img-tool --api-key key.json upload --folder ferris --link https://i.imgflip.com/7gq1em.jpg --file crab_ferris.png
# error: the argument '--file <FILE>' cannot be used with '--link <LINK>'
# 
# Usage: img-tool --api-key <API_KEY> upload --folder <FOLDER> <--file <FILE>|--link <LINK>>
# 
# For more information, try '--help'.

./img-tool --api-key key.json upload --folder ferris --link https://i.imgflip.com/7gq1em.jpg
# Config { command: Upload { file_source: FileSource { file: None, link: Some(Url { scheme: "https", cannot_be_a_base: false, username: "", password: None, host: Some(Domain("i.imgflip.com")), port: None, path: "/7gq1em.jpg", query: None, fragment: None }) }, folder: "ferris" }, api_key: "key.json" }
```

Another interesting thing: we can see the separated `--file` and `--link` args in the help message by the `<>` triangle brackets. It shows the user that only one of the is needed. Cool, right? :sunglasses:

The full [src](https://github.com/TheBestTvarynka/trash-code/commit/aba3c688fa83a397d14a9339f564efe4625516e1) code of the example above.

## Parsing args into a custom structure

This recipe will be smaller than the previous ones and similar to the second one. But I just want to show that we can do such tricks.

We are going to improve prev example by adding a more flexible way to pass the API key. Assume that for authentication we need two tokens: app id and app secret. And we want to operate them as one structure. Usually, in such cases, developers take them as two separate strings and then create one structure based on those strings. But we a smarter and know how to use commands:

```rust
#[derive(Debug, Clone, Args)]
struct ApiKey {
    /// app id
    #[arg(long)]
    pub api_app_id: String,

    /// app secret
    #[arg(long)]
    pub api_app_secret: String,
}

/// Img tool config structure
#[derive(Parser, Debug)]
struct Config {
    /// command to execute
    #[command(subcommand)]
    pub command: Command,

    /// API key data
    #[command(flatten)]
    pub api_key: ApiKey,
}
```

Let's test it and see the trick:

```bash
./img-tool --help
# Img tool config structure
# 
# Usage: img-tool --api-app-id <API_APP_ID> --api-app-secret <API_APP_SECRET> <COMMAND>
# 
# Commands:
#   upload    Upload image to the Imgur
#   download  
#   help      Print this message or the help of the given subcommand(s)
# 
# Options:
#       --api-app-id <API_APP_ID>          app id
#       --api-app-secret <API_APP_SECRET>  app secret
#   -h, --help                             Print help
```

We have two separate args `api-app-is` and `api-app-secret`, but they will be parsed and placed in the one `ApiKey` structure. It's very convenient and we can continue to work with the auth data as one structure without any additional actions.

The full [src](https://github.com/TheBestTvarynka/trash-code/commit/c3a77caad5bb861c15bd007cb8091d19e5e001c3) code of the example above.

## Unsolvable problem

This section is not an actual recipe. I'll tell you about a problem that doesn't have a perfect solution so far (or I just cannot find it).

Let's take the previous recipe and make the task more difficult. Assume that the user should specify the app id and secret **OR** API key file. In other words, the user should provide the one `--api-key-file` arg or `--api-app-id` and `--api-app-secret` args. It means one **OR** two arguments.

I found a few similar questions on the Internet:

* [Clap either -a OR (-b AND -c) arguments](https://users.rust-lang.org/t/clap-either-a-or-b-and-c-arguments/80331)
* [Using clap-derive with two groups of arguments](https://stackoverflow.com/questions/74846776/using-clap-derive-with-two-groups-of-arguments)

But both of them didn't have any useful answers. So far, the following code is the best how we can solve this problem:

```rust
#[derive(Debug, Clone, Args)]
struct ApiKeyData {
    /// app id
    #[arg(long, requires = "api_app_secret")]
    pub api_app_id: Option<String>,

    /// app secret
    #[arg(long, requires = "api_app_id")]
    pub api_app_secret: Option<String>,
}

#[derive(Debug, Clone, Args)]
#[group(required = true, args = ["api_key_file", "api_app_id", "api_app_secret"])]
/// Possible types of the api key source
struct ApiKey {
    /// Path to the json file with API key
    #[arg(long)]
    api_key_file: Option<PathBuf>,

    /// Specify API key data in args
    #[command(flatten)]
    api_key_data: ApiKeyData,
}

// The `Config` structure remains unchanged since the last recipe
```

The main idea is to create an [`ArgGroup`](https://docs.rs/clap/latest/clap/struct.ArgGroup.html) and require additional args in the `ApiKeyData` structure if one of the needed args is not specified. This approach has a lot of big inconveniences:

* Fields in the `ApiKeyData` structure are optional. Yes, during parsing they will be validated and 100% have values, but for further work, we are forced to unwrap them and create another structure.
* The help message for the user is not fully informative. It shows the among those three args only one is required. That is a lie because we need a key file **OR** app id + secret.

Enough talking. Let's see it in action:

```bash
./img-tool --help
# Img tool config structure
# 
# Usage: img-tool <--api-key-file <API_KEY_FILE>|--api-app-id <API_APP_ID>|--api-app-secret <API_APP_SECRET>> <COMMAND>
# 
# Commands:
#   upload    Upload image to the Imgur
#   download  
#   help      Print this message or the help of the given subcommand(s)
# 
# Options:
#       --api-key-file <API_KEY_FILE>      Path to the json file with API key
#       --api-app-id <API_APP_ID>          app id
#       --api-app-secret <API_APP_SECRET>  app secret
#   -h, --help                             Print help

./img-tool --api-key-file key.json download --link https://imgur.com/gallery/vNOUshX --dest-file ferris.png
# Config { command: Download { link: Url { scheme: "https", cannot_be_a_base: false, username: "", password: None, host: Some(Domain("imgur.com")), port: None, path: "/gallery/vNOUshX", query: None, fragment: None }, dest_file: "ferris.png" }, api_key: ApiKey { api_key_file: Some("key.json"), api_key_data: ApiKeyData { api_app_id: None, api_app_secret: None } } }

./img-tool --api-app-id tbt --api-app-secret secret download --link https://imgur.com/gallery/vNOUshX --dest-file ferris.png
# Config { command: Download { link: Url { scheme: "https", cannot_be_a_base: false, username: "", password: None, host: Some(Domain("imgur.com")), port: None, path: "/gallery/vNOUshX", query: None, fragment: None }, dest_file: "ferris.png" }, api_key: ApiKey { api_key_file: None, api_key_data: ApiKeyData { api_app_id: Some("tbt"), api_app_secret: Some("secret") } } }

./img-tool --api-app-id tbt download --link https://imgur.com/gallery/vNOUshX --dest-file ferris.png
# error: the following required arguments were not provided:
#   --api-app-secret <API_APP_SECRET>
# 
# Usage: img-tool <--api-key-file <API_KEY_FILE>|--api-app-id <API_APP_ID>|--api-app-secret <API_APP_SECRET>> <COMMAND>
# 
# For more information, try '--help'.
```

The full [src](https://github.com/TheBestTvarynka/trash-code/commit/6d5ec50ea36aa2ca6fab7409e68c08a46d1a346e) code of the example above. Conclusion: this problem is solvable but in a very inconvenient way.

# References & final note

1. Official docs: [derive](https://docs.rs/clap/latest/clap/_derive/index.html) and [builder](https://docs.rs/clap/latest/clap/index.html) references.

The end. The official reference has all you need.

As you can see, the `clap` gives us a lot of opportunities to configure args parsing and validation. Sometimes it's really worth it to read the docs or references. **Advice for the future:** try to move as much work as you can to the `clap`. It's a very powerful tool, so you shouldn't be bothered by manual parsing or validation of the args.
