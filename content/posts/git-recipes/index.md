+++
title = "Git recipes"
date = 2024-06-25
draft = false

[taxonomies]
tags = ["clap", "recipes", "git"]

[extra]
keywords = "Git, VCS"
toc = true
+++

Recently I lectured in my company about [`git`](https://www.git-scm.com/) and resolving specific situations with it. It inspired me to write this article. I know that there is tons of info about `git` and related stuff. I wanted to make this one as a comprehensive set of instructions about resolving concrete git situations.

_**Note.**_ This page can be changed and improved in the future (I'll try to save the backward compatibility)!

# How to read this page

You can read it in any order. If you are interested only in some particular scenario, then read only the corresponding section. All sections are independent from each other and you can start reading anywhere.

The required `git` knowledge level to read this article is pretty low. If you know what are _commit_, _branch_, and _index_ (_stage_), then everything is good.

## Goals

* Explain how to resolve common `git` tasks I face at work every time.
* Improve your `git` knowledge.
* Dispel the fear of working with `git` in uncommon cases (if you have one).

## Non-goals

* **Explain how `git` works inside.** There is a great section in [the book](https://www.git-scm.com/book/en/v2) about git internals: [Git Internals](https://www.git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain).
* **Propagate to work with `git` only from the terminal.** If the GUI is enough for you to perform all needed tasks, then it's great! The less we interact with `git`, the lower the probability of having problems with it :upside_down_face:.
* **Write zero to pro guide.** It's just impossible :smile:.
* **Replace the [Oh Shit, Git!?!](https://ohshitgit.com/) page**. The _Oh-Shit-Git_ purpose is to quickly resolve common problems. But my purpose is to offer guides for common tasks.

# Recipes

## Last commit to index

## Discard last commit

## Edit last commit

## Edit any commit in branch

## Split commit into two

## Safe dirty changes

## Split branch into a few pull requests

## Untrack file

## Aliases

# Conclusion

There are no any _smart_ conclusions. But I have stupid ones :grin::

* Make small atomic commits. The smaller your commits are, the better. It's easier to work with such commits. Moreover, everything committed in `git` always stays inside of it and can always be restored.

I just leave it here: [HN: Nobody really understands git](https://news.ycombinator.com/item?id=16807206).

# Doc, references, code

* The [Pro GIT](https://www.git-scm.com/book/en/v2) book.
* [Oh Shit, Git!?!](https://ohshitgit.com/).
