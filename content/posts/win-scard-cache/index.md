+++
title = "Windows smart card cache"
date = 2024-04-15
draft = false

[taxonomies]
tags = ["debugging", "windows", "rust", "scard"]

[extra]
keywords = "Rust, API Monitor, Debugging, API, Windows, smart card"
toc = true
+++

# Getting Started

## What I'm going to read?

A few months ago I had a great opportunity to implement [the smart card emulation](https://github.com/Devolutions/sspi-rs/pull/210). The whole point was to emulate the smart card behavior without an actual smart card. I can talk a lot about such stuff, but in this article, I'm going to share my experience in smart card cache exploration and implementation.

To make all my decisions clear for you, I provide you with some additional context. Imagine the reimplemented `winscard.dll` without actual calls to the security device but with the smart card emulation under the hood. Boom :boom:! We have scard auth without a physical device.

You can have questions like *"How can we replace the original dll"*, *"What is the point of such implementation"*, *"What are the limitations of it"*, and so on. Unfortunately, it's out of the context of this article. Here I focused only on debugging techniques and smart card cache (I'll explain the reason for it further in the article).

## Goals

* A brief overview of the smart card architecture in Windows.
* Debugging techniques and tips.
* Explain smart card cache items for [**PIV**](https://www.idmanagement.gov/university/piv/) smart cards in Windows.
* Fun :partying_face:.

Who should read this article:

* Ones who are interested in how PIV smart cards work on Windows.
* Ones who are interested in debugging.
* Ones who just need PIV scard cache items format (Hi :wave:, `msclmd.dll`).

## Non-goals

* Explain every piece of smart card architecture in Windows.
* Teach you everything about WinDbg (only a small part of the whole WinDbg functionality is covered).

# What is WinS(mart)Card API?

## Overview

In Windows, [WinSCard](https://learn.microsoft.com/en-us/windows/win32/api/winscard/) is the lowest accessible API for communicating with smart cards. You operate only using handles (`SCARDCONTEXT`/`SCARDHANDLE`/etc), raw [APDU](https://en.wikipedia.org/wiki/Smart_card_application_protocol_data_unit)s ([SCardTransmit](https://learn.microsoft.com/en-us/windows/win32/api/winscard/nf-winscard-scardtransmit) function), and a bunch of other low-level functions. If you look at the CSP and KSP-based architecture diagram, you'll see WinScard almost at the bottom of the picture:

[![](https://learn.microsoft.com/en-us/windows/security/identity-protection/smart-cards/images/sc-image206.gif)](https://learn.microsoft.com/en-us/windows/security/identity-protection/smart-cards/smart-card-architecture#base-csp-and-ksp-based-architecture-in-windows)

Let's analyze this diagram above. Usually, when you deal with any crypto operations in Windows, you use some high-level API - [CryptoAPI](https://learn.microsoft.com/en-us/windows/win32/seccrypto/cryptoapi-system-architecture). But the CryptoAPI is just a high-level API and every [CSP](https://en.wikipedia.org/wiki/Cryptographic_Service_Provider) implements it. In the case of smart cards, the corresponding CSP (`basecsp.dll`, in our case) can not contain the CryptoAPI implementation because all crypto operations should be performed on the security device. This is why BaseCSP uses (depends) on the smart card minidriver. Such a minidriver translates high-level functions into WinSCard API calls. Nothing more.

Usually, every smart card vendor ships its smart card driver (you can see it on the diagram). The Windows itself contains two smart card drivers:

* One driver for Windows proprietary smart cards with closed specifications and documentation.
* And another driver for PIV-compatible smart cards. It has open specifications ([Smart Card Minidriver Specification](https://learn.microsoft.com/en-us/previous-versions/windows/hardware/design/dn631754(v=vs.85)) + [NIST SP 800-73-4](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-73-4.pdf)) but closed documentation.

> *"...two smart card drivers:"*

It was a small lie :sweat_smile:. I mean, yes, it's true, but those two drivers are implemented inside one dll: `msclmd.dll`. *For the rest of this article, we'll talk **only** about `msclmd.dll` and **PIV-compatible** smart cards.*

> *Got it. But why is card cache the main topic of this article?*

I have a few reasons for it:

* No matter how good our WinSCard implementation is, authentication will fail without a working cache.
* Only a few cache items are described in the Window's minidriver specification. Many of them are undocumented and unknown.
* It was the hardest part to debug and implement :stuck_out_tongue_closed_eyes:.

# Debugging

## What and how will we debug?

It's easier to work if we have a concrete destination point. Let's take the Remote Desktop Client (`mstsc.exe`), replace the `wiscard.dll`, and try to connect to the remote machine using the emulated smart card. The established RDP connection is our goal.

To make my life easier, I took the [MsRdpEx](https://github.com/Devolutions/MsRdpEx/) and hooked the `winscard.dll` using the `MSRDPEX_WINSCARD_DLL` environment variable. Pretty simple and usable.

For the further work, only two instruments will be used:

* IDA.
* [WinDbg](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/).

[**Time Travel Debugging**](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/time-travel-debugging-overview). I encourage you to try it! This technique saved me a lot of time. The possibility of re-playing and moving backward gives us almost unlimited power :muscle:.

# Let's start the journey

t

# Smart card container name

w

# Final results

sc

# Conclusion

c

# Doc, references, code

1. [Smart Card Architecture](https://learn.microsoft.com/en-us/windows/security/identity-protection/smart-cards/smart-card-architecture).
2. [`winscard.h`](https://learn.microsoft.com/en-us/windows/win32/api/winscard/).
3. [Smart Card Minidrivers](https://learn.microsoft.com/en-us/windows-hardware/drivers/smartcard/smart-card-minidrivers).
4. [Smart Card Minidriver Specification](https://learn.microsoft.com/en-us/previous-versions/windows/hardware/design/dn631754(v=vs.85)).
5. Implemented smart card caches: [`scard_context.rs`](https://github.com/Devolutions/sspi-rs/blob/eee2c0b481e63d660cb0cff2c99599fb30b5dd0d/crates/winscard/src/scard_context.rs#L146-L394).
6. [Time Travel Debugging](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/time-travel-debugging-overview).