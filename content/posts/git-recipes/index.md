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

I want to leave a few clarifications before I write recipes.

* `HEAD`. How does Git know what branch you’re currently on? It keeps a special pointer called `HEAD`. [3.1 Git Branching - Branches in a Nutshell](https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell). [What is HEAD in Git?](https://stackoverflow.com/a/2304106/9123725)
* `~<number>` and `^<number>` is the same in many cases, but not always. Read this SO answer for more details: [https://stackoverflow.com/a/2222920/9123725](https://stackoverflow.com/a/2222920/9123725).

## Last commit to index

So, you want to *"undo"* the last commit but save changes. The `git` has a [`git reset`](https://git-scm.com/docs/git-reset) command for that:

```bash
git reset --soft HEAD~1
```

As a result, you'll not have the last commit and all changes will be in the stage. And yes, you can enter any number of commits instead of `1`. If you want changes to be untracked, then use the save command but with the `--mixed` parameter:

```bash
git reset --mixed HEAD~1
```

## Discard last commit

I know that it's a bad idea, but if you really want, then this is how you can **discard** the last commit:

```bash
git reset --hard HEAD~1
```

**Note.** If you regret your decision, then you can restore lost commit with its hash:

```bash
git cherry-pick <commit-hash>
```

If you don't know its hash, then try to find it with `git reflog`.

## Edit last commit

Actually, you only need the `git commit --amend` to do it. `git commit` has a lot of flags and possible use cases, but I list here only common ones so you can copy and use them.

If you have any changes in the stage, then they will be added to commit after running `git commit --amend`. For example:

```bash
vim README.md
git commit --amend --no-edit
```

Now the last commit contains changes we made in `README.md` file. The `--no-edit` arg means do not edit the commit message. If you want to edit the commit message, then use the `-m` arg and just nothing (`git` will ask you for the message). It's also possible to alter the commit author, date, and so on:

```bash
# Reset commit author saving commit date. This command is helpful when you've changed the `.gitconfig` file.
git commit --amend
git commit --amend -m "new commit message"
git commit --amend --reset-author --no-edit --date="$(git log -n 1 --format=%aD)"
git commit --amend --author=<author>
```

Run `git commit --help` for more flags.

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
