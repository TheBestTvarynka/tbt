+++
title = "crypto-helper"
description = "Web app that can hash/encrypt/sign the data. JWT debugger."
date = 2023-04-09
draft = false

[taxonomies]
tags = ["yew", "rust", "tools"]

[extra]
toc = true
keywords = "Rust, Yew, Crypto, JWT, Encrypt, Decrypt, Hash, HMAC, MAC, RSA"
+++

Visit this tool at [crypto.qkation.com](https://crypto.qkation.com).

The crypto-helper is an online app that helps to work with the different crypto algorithms. This app can hash/hmac, encrypt/decrypt, and sign/verify the data, debug the JWT tokens. All computations are performed on the client side. This tool never sends the data the any servers.

### Motivation

#### Crypto helper

During my work, I often need to work with bytes sequences that I need to encrypt or decrypt, hash or verify. For example, I might intercept the traffic, extract encrypted Kerberos cipher data, and try to decrypt it using the user's password. Or take payload + signature from the message and try to recalculate it. Such small tasks are very annoying and I decided to optimize them.

Basically, it just takes the input bytes in some format and performs the selected algorithm on it.

{{ img(src="crypto-helper-demo-krb.png" alt="crypto helper krb encryption demo") }}

#### JWT debugger

Another interesting feature is the JWT debugger. I had some time working with Azure AD authorization and its JWT tokens. Why did I decide to implement my own if jwt.io already exists? The answer is pretty obvious: the existing one is very inconvenient:

* I really don't like its big header. AzureAD tokens are pretty big. Such a massive header is just a waste of space.
* Problems with scrolling. Did you ever try to scroll the header or payload section that contains more than five fields? If not then try.
* Header and payload fields are not resizable.
* Every header/payload/key change immediately overwrites the original JWT token. I often want to see the original one, compare it to a new one, or create a few different tokens based on the initial token.
* Ads. Ads. Ads.

Here is how my JWT debugger in action:

{{ img(src="jwt-demo.png" alt="JWT demo") }}

#### Features

* Written in [Rust](https://github.com/rust-lang/rust) :crab: using [yew](https://github.com/yewstack/yew) :sparkles:
* `MD5`
* `SHA1`/`SHA256`/`SHA512`
* Kerberos ciphers: `AES128-CTS-HMAC-SHA1-96`/`AES256-CTS-HMAC-SHA1-96`
* Kerberos HMAC: `HMAC-SHA1-96-AES128`/`HMAC-SHA1-96-AES256`
* `RSA`
* JWT debugger. Supported signature algorithms:
  * `none`
  * `HS256`
  * `HS384`
  * `HS512`
  * `RS256`
  * `RS384`
  * `RS512`

### Languages

It's a fun story because I rewrote this tool two times. Firstly, I plan to make it very simple and start implementing using plain html/css/js (commit). Then I realized that it's hard to improve, maintain, and cause a lot of stupid bugs. The second choice was React. In the middle of the rewriting, I thought: why not write it in Rust?

After some research, I decided to use the [Yew](https://github.com/yewstack/yew) framework. Basically, it's like React but for Rust. From my experience, I can tell that *yew* is a great framework to write web applications (SPAs) in Rust. It works well, has great documentation, and is easy to work with. I faced only one problem: styling. We don't have (so far) such convenient libraries as we have for React.

### How to use it

Just follow [the link](https://crypto.qkation.com/) and paste your data. The interface is pretty intuitive.

### Moving further

At this point, this tool has the all needed functionality for me. I plan to improve it continuously according to my needs and goals. If you have any feature requests, then create an [issue](https://github.com/TheBestTvarynka/crypto-helper/issues/new) with the description. It'll be a priority for me.
