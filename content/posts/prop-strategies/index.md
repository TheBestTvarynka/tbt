+++
title = "Proptest strategies"
date = 2023-10-27
draft = false

[taxonomies]
tags = ["rust", "proptest"]

[extra]
keywords = "Rust, proptest, testing"
toc = true
# mermaid = true
# thumbnail = "sspi-thumbnail.png"
+++

# Goals

[Property-based](https://en.wikipedia.org/wiki/Software_testing#Property_testing) testing (or just *property testing*) is a very useful testing approach that can help you find a lot of problems in the code you can't even think of. I focused my attention on the [`proptest`](https://docs.rs/proptest/latest/proptest/) crate. So, if you are using another library for property testing, then this post isn't for you. This article is a great start if you already have wrote your first prop test and now want to write more complex property tests and strategies.

**Goals**:

* Show non-standard strategies examples.
* Give examples of how different structures can be generated in different ways using the `protest` library.

**Non-goals**:

* Recall the whole theory of prop testing and testing at all.
* Write the *"ultimate"* guide to the `proptest` library.
* Replace the existing [proptest book](https://proptest-rs.github.io/proptest).
* Explain the difference between *prop-* and *fuzz-* testing.

How my guide differs from the official book? By approach. The proptest book tells you *"what we can do with our strategies"* and *"what this library is capable of"*. I've chosen another approach: we'll write strategies from the simplest one and make them more complicated with each step. In such a way we'll cover most of the situations you can have in your project.

# Overview

