+++
title = "Smart card container name"
date = 2024-05-04
draft = false

[taxonomies]
tags = ["debugging", "windows", "scard"]

[extra]
keywords = "Debugging, API, Windows, Smart Card, Reverse Engineering"
toc = true
+++

# Getting Started

In my previous article, I explained the different smart card cache items (files), their format, and their purpose. The current one is some continuation of talking about PIV smart cards in Windows. The setup is the same: Windows smart card minidriver + PIV smartcard.

## Goals

* Explain how the Windows [minidriver](https://learn.microsoft.com/en-us/windows-hardware/drivers/smartcard/smart-card-minidriver-overview) treats the smart card container name.
* Tell how a bad container name can break scard auth.
* Fun :partying_face:.

## Non-goals

* Explain every piece of smart card architecture in Windows.
* Explain every smart card minidriver function.

# Overview

# Conclusion

# Doc, references, code

* [Smart Card Minidriver Overview](https://learn.microsoft.com/en-us/windows-hardware/drivers/smartcard/smart-card-minidriver-overview).