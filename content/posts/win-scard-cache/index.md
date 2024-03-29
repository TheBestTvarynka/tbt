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

a

## Goals

g

# What is WinS(mart)Card API?

## Overview

o

## Driver? Minidriver?

m

# Debugging

## What will we debug?

rdp

## How will we debug?

windbg

## Time Travel Debugging

ttd

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