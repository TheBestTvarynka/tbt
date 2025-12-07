+++
title = "Announcing crypto-helper v.0.16.0"
date = 2025-12-14
draft = false

[taxonomies]
tags = ["rust", "tool", "project", "yew", "crypto-helper"]

[extra]
keywords = "Rust, Yew, ASN1, ASN1 parser, ASN1 editor"
toc = true
# thumbnail = "dataans-thumbnail.png"
+++

Visit this tool at [crypto.qkation.com](https://crypto.qkation.com).

Short release notes can be found here: `TODO`.

# Intro

[The previous release](https://github.com/TheBestTvarynka/crypto-helper/releases/tag/v.0.15.0) was around 1.5 years ago.
I've made many improvements and fixes since then. It's time to publish a new release.

Note:

> _Actually, releases are just checkpoints during the `crypto-helper` development and do not mean anything special._
> _All features and fixes are deployed and available right after the merging into the `main` branch._

This post contains a comprehensive live of new changes with additional explanations of how to use new functionality.

# ASN1 major features

## ASN1 tree editing

[feat(asn1): implement asn1 tree editing (#105)](https://github.com/TheBestTvarynka/crypto-helper/pull/105).

## Wide strings auto-decoding

[feat(crypto-helper): asn1: autodecode wide strings; (#103)](https://github.com/TheBestTvarynka/crypto-helper/pull/103).
I often see that in some cases strings can be encoded as UTF16 inside `OctetString` node. For example, [CredSSP](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-cssp/34ee27b3-5791-43bb-9201-076054b58123) smart card credentials are encoded this way:

![](./credssp-scard-credentials.png)

But I forgot that many other buffers can be a valid UTF-16. For example, on the screenshot below you can see the highlighted encryption key which is decoded as UTF-16:

![](./encryption-key-as-wide-string.png)

I still decided to keep this feature because it looks fun :satisfied:. If I need to look at buffer bytes, I can always look at the hex-viewer to the right.

# crypto-helper major features

## HMAC-SHA support

`HMAC-SHA` algorithm support: [crypto-helper/commit/dcc4e41f](https://github.com/TheBestTvarynka/crypto-helper/commit/dcc4e41f73e84e5e6f49d1d24e3abac128368e53). Including the following SHA variations: `SHA256`, `SHA384`, and `SHA512`. Demo:

![](./hmac-sha-example.gif)

## Redirect to ASN1

[Add button to redirect output to ans1 page (#95)](https://github.com/TheBestTvarynka/crypto-helper/pull/95). Many thanks [@grok-rs](https://github.com/grok-rs) for his contribution!
The idea is simple: redirect to the ASN1 page and try to decode the result of the cryptographic operation as ASN1 DER structure.

This feature is very useful for me when I want to decrypt-and-parse the encrypted part of the Kerberos message. For example:

![](./asn1-redirect-example.gif)

# Other changes

* More OIDs: [#81](https://github.com/TheBestTvarynka/crypto-helper/pull/81), [#102](https://github.com/TheBestTvarynka/crypto-helper/pull/102).
* Fix incorrect diff calculation: [#94](https://github.com/TheBestTvarynka/crypto-helper/pull/94).
* Allow empty `BitString` decoding: [#93](https://github.com/TheBestTvarynka/crypto-helper/pull/93).
* CI: add build WASM step: [crypto-helper/commit/814d1675](https://github.com/TheBestTvarynka/crypto-helper/commit/814d16755e95d4f4e69b736c7917d578beb6d881).
* Update dependencies and refactoring: [#83](https://github.com/TheBestTvarynka/crypto-helper/pull/83), [#87](https://github.com/TheBestTvarynka/crypto-helper/pull/87), [Rust 2024 edition (#101)](https://github.com/TheBestTvarynka/crypto-helper/pull/101), [#106](https://github.com/TheBestTvarynka/crypto-helper/pull/106).
* `README.md` and typos fixes: [#79](https://github.com/TheBestTvarynka/crypto-helper/pull/79), [#80](https://github.com/TheBestTvarynka/crypto-helper/pull/80), [crypto-helper/commit/15785bfe](https://github.com/TheBestTvarynka/crypto-helper/commit/15785bfecb6d23c7aea0c19f7953af07fa2e08c4), [crypto-helper/commit/dd47d963](https://github.com/TheBestTvarynka/crypto-helper/commit/dd47d963e8f56c3d65e80f4da059db094e474dfb).

# References

1. GitHub release.
2. 