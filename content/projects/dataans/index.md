+++
title = "dataans"
description = "Take notes in the form of markdown snippets grouped into spaces."
date = 2024-10-16
draft = false

[taxonomies]
tags = ["rust", "tauri", "leptos", "project"]

[extra]
toc = true
keywords = "Rust, Tauri, Leptos, Note-taking, Markdown"
+++

Source code: [github/TheBestTvarynka/Dataans](https://github.com/TheBestTvarynka/Dataans/).

The Dataans is a desktop app that allows you to take notes in the form of markdown snippets grouped into spaces. Yes, it's another note-taking app, but with unique features I miss in all other note-taking apps.

# Motivation

I write notes almost every time I use my computer. Usually, it's small pieces of data/code/docs/links I found during the research or bug fixing. So, I need some place to store them and the ability to find needed pieces of information easily. I used to use a single text file and open it in Notepad or VS Code. After the research, I got the file with almost random symbols:

// 

Do I need to explain that it isn't convenient and hard to find anything? I needed a better solution. I tried other note-taking apps, but every one of them had flaws I didn't like a lot.

## What's wrong with existing note-taking apps?

I relate a lot to this blog post ([Why build a messenger app only for sending to yourself?](https://monoline.io/posts/2021/11/11/why-build-a-messenger-app-only-for-sending-to-yourself/)). I quote many key points from it but explain them in my own way and how they relate to me.

1. **Too many options**. I don't need so much test editing functionality. I just want to take notes and have simle markup (styling) functionality. Something like [markdown](https://www.markdownguide.org/) or [asciidoc](https://docs.asciidoctor.org/).
2. **Unwanted features**. This echoes the previous point, but I want to draw your attention specifically to the UNWANTED features. For example, I don't need AI to search for info in my notes. I only need a typical search engine. I don't want AI to write notes for me. I can write down my thoughts by myself.
3. **Article-oriented mindset**. People don't think in articles. Many note-taking apps look like you are going to write an article instead of just writing down the idea, thought, or a random piece of data. When I see a blank screen with the cursor, I feel I need to write a document with a defined structure, style, and line of thought. It throws off the thoughts I wanted to write at the beginning.

## Missing features

Despite the listed flaws above, I still have missing features I would like to have in my note-taking app. Some of them may feel weird and illogical to you, but I know what I want :stuck_out_tongue_closed_eyes:.

All note-taking apps I tried lack one or more features listed below. No one contains all of them at once (but if you find such an app, please, tell me about it).

1. **Desktop app**. Yes, you heard it right. In the era of the web, I want a desktop app. Usually, the browser has dozens of opened tabs across multiple windows. It becomes hard to find the tab with notes (even when it's pinned).
2. **Quake (drop-down) mode**. I have used the [Quake](https://github.com/Guake/guake/) terminal since 2019. I like it a lot and the most pleasant feature is a drop-down mode. I set a keybinding to the `F1` key and always have my terminal with me. I would like to have the same ability for my note-taking app because I use it often.
3. **Cross platform**. It should behave the same on _Windows_ and _Linux_. Other platforms mey may be supported too, but it's not a requirement for me.
4. **Markdown**. It is simple, easy to learn, and looks good. It contains a perfect set of styling and markup functionality for me.

# Solution

## Features

## How to use it

# Moving further

# References & final note