+++
title = "Debugging with API Monitor"
date = 2023-07-25
draft = false

[taxonomies]
tags = ["debugging", "api-monitor", "rust"]

[extra]
keywords = "Rust, API Monitor, Debugging, API"
toc = true
+++

The [API monitor](http://www.rohitab.com/apimonitor) speaks for itself. It's a great tool for the system API calls debugging, monitoring, information extraction, exploring. Especially in cases when you don't know how exactly API works.

# Goals

**Goals:**

* Explain how to deal with cases when existing API definitions are not enough.
* Show on the example how we can debug custom DLLs and how to write definitions for them.

**Non-goals:**

* Write the *"ultimate"* guide to the `API Monitor`.
* Describe how the `API Monitor` works under the hood.

# Let's debug WinSCard API

Debugging example, problem, what is not captures, memory editor.

# Time to fix XML definitions

Example how we can fix it, recording after.

# What if we have custom DLL to monitor

Write a simple program, dll (in Rust of course), XML definitions, show how it works.

# References & final note

1. [Trace APDU on Windows](https://www.mysmartlogon.com/knowledge-base/trace-apdu-on-windows/).
2. [Process memory editor, Unions and Arrays](http://www.rohitab.com/discuss/topic/37197-api-monitor-v2-r6-release-process-memory-editor-unions-and-arrays/).

Conclusions.