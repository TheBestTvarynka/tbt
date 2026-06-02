+++
title = "Obsidian photo-tagging"
description = "Simple Obsidian plugin for tagging people on photos."
date = 2026-02-17
draft = false
template = "post.html"

[taxonomies]
tags = ["javascript", "typescript", "project", "react", "frontend"]

[extra]
toc = true
keywords = "TypeScript, Obsidian, Plugin, Tagging, Genealogy"
thumbnail = "photo-tagging-thumbnail.png"
enddate = "present"
status = "development"
+++

TL;DR: [github/TheBestTvarynka/photo-tagging](https://github.com/TheBestTvarynka/photo-tagging).

# The problem

I store my genealogy research data inside the Obsidian vault.
Its structure assumes that I have one page per person.
When I got a lot of old photos from many relatives, I started thinking about how to store them.
Obviously, I cannot group them by person because there are many photos with many people in them.
After a bit of thinking and visualizing :face_in_clouds: :monocle_face:, I realized that the best solution for me would be to show all the photos they currently have on a person's page.

# The solution

I tried to find a similar thing in existing Obsidian plugins, but failed.
Unfortunately, I did not find anything like this plugin.
So, as you may already guess, I decided to write my own plugin! :star_struck:

The idea is simple: manually tag people on photos and store the connections data in the `.json` file.
Then the user can add a special code block that turns into a gallery of this person's images.

# Usage example

First things first: the user needs to enable the plugin in the Obsidian app:

![](./enable_plugin.png)

Then, optionally, the user can configure the target `.json` file location.
The default value is `photo-tags.json`, and it can be changed in the plugin settings:

![](./settings.png)

Next, click on any image inside the vault with the right-mouse button, and you will see a new `Open in tagger` option:

![](./menu.png)

Then, in the open tagger, the user can tag people and optionally add custom hashtags.
These hashtags work the same way as regular hashtags: their only purpose is to group photos into different categories.

![](./tagger.png)

After that, on the corresponding person's page, add the `tagged-photos` code block. It will become the person's photo gallery.
Optionally, if the user adds `group: hashtags` inside the code block, it will group a person's photos by hashtags and render a separate gallery for every hashtag.
See the example:

| Usual | Grouped by hash-tags |
|-|-|
| ![](./usual.png) | ![](./groupped.png) |

Click on the image to see it in full screen:

![](./gallery.png)

Basically, that's all.
Easy, simple, and convenient :sparkles:.

# Features

1. Built-in photo tagger.
  ![](./tagger.png)
2. A special `tagged-photos` code block processor that turns the current person's photos into a gallery.
  ![](./persons_photos.png)
  When the user clicks on any image, it will be opened in the full-screen gallery (powered by [`photoswipe`](https://photoswipe.com/)):
  ![](./gallery.png)
3. Hash-tags support (see screenshots above).

# What's next

The core features are already implemented and serve their purpose well.
I like them.

I still have a few improvements in mind:

- **Optimizations/caching.** Almost all old photos are just scanned from old paper photos.
  I usually scan at 1200 DPI, and the resulting image is huge.
  Sometimes the page loads slowly. I want to fix that.
- **Better tagger.** Current photo tagger is OK, but not good.
  I plan to make it more user-friendly.
- **Automatic connections updates.** Catch when a photo is deleted/renamed/moved, and automatically update the connections in the internal `.json` file.
- **Photo explorer?** I am not sure what this feature should look like, and even if it's worth implementing.
  But from time to time, I am thinking about a generic photo explorer: a special page where the user can query photos using different combinations of selectors (like hashtags + people, AND and OR operations in the query).
  The future will show me the right path. :relieved:

# Useful links

1. Source code: [github/TheBestTvarynka/photo-tagging](https://github.com/TheBestTvarynka/photo-tagging).
