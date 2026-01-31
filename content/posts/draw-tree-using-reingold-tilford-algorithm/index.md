
+++
title = "Drawing Genealogy Graphs. Part 1: Tree Drawing Using Reingold-Tilford Algorithm"
date = 2026-02-05
draft = false

[taxonomies]
tags = ["algorithms", "tree-data-structure", "data-structures", "typescript", "drawing-genealogy-graphs-series"]

[extra]
keywords = "Algorithm, Reingold-Tilford Algorithm, Trees, Node positioning, Data structures, Algorithms"
toc = true
+++

# Motivation

Last year I started my own genealogy research. Honestly, I planned to start it many years ago.
One day in 2025, I realized that if I did not start, then it may never happen and I would lost a huge part of my family history.

I spent a lot of time visiting and asking my old relatives many questions, scanning old photos and documents.
At some point, I questioned myself about how it's better to store all this data.
Some day I will write a detailed post about considered approaches, proc and cons of each of them and why I chose what I chose.
But today, in this post, I am going right to the solution.

I chose [Obsidian](https://obsidian.md/) as a main app for storing all genealogy data including photos, conversations recordings, texts, peoples information, scanned documents, etc.
I wanted to have a full control over my data. I did not want to trust any websites.
Obsidian is very flexible, convenient, and extendable. For me, it's a perfect choice.
Additionally, I periodically back up genealogy data to external SSD and [Google Cloud Storage](https://cloud.google.com/products/storage) bucket.
I do not know about you, but I feel myself pretty comfortable and reliably with this approach.

Let me answer one question before we go further. Yes, I know about [myheritage](https://www.myheritage.com/).
I still fill up all family and ancestors basic info like names, born, marriage, and death dates, etc.
I consider myheritage a great tool for a research.
But again, I do not trust it and do not want to trust it to have all my family personal photos, stories, some documents, and other sensitive information.

So, where did I stop. Oh, yes, Obsidian. Obviously, I would want to visualize the family graph.
The Obsidian has a built-in [Graph View](https://help.obsidian.md/plugins/graph).
It looks funny, but does not look even close to what I want.
I browsed community plugins and did not find anything helpful at all.
So, I decided to write my own Obsidian plugin that will render family members as a pretty graph.
Plugin details are out of the scope of this article. I will definitely write a separate post about it in the future.

**This article is focused only on the nodes positioning algorithm.**

# Intro

I knew that calculating positions for nodes during graph or tree drawing is not a trivial task and it would definitely be hard.
But I still decided to try to solve this task by myself. I spent a few days on it without achieving any results.
Then I googled for existing algorithms (yes, I should have done it before spending a few days on nothing).

I did not find the exact algorithm, but I found a similar (and much simpler) one: the Reingold-Tilford Algorithm for rendering pretty trees.
This algorithm is not suitable for me, because it is for tree node positioning. But all family relationships are a (complex) graph.
This fact introduces a lot of complexity to the task, but I decided to implement the simpler version first and then improve it to meet my requirements.

But why write one more post when other explanations exist?
The problem with existing explanations is that they explain every step of the algorithm, but do not explain **_WHY it works_** and **_WHAT THE PURPOSE_** of each step of the algorithm is.
Many posts feel like the code, but converted to English words :pensive:.

# The Reingold-Tilford Algorithm

## First walk

## Second walk

# Demo: Grafily

I hope the reader does not forget that my primary goal is family graph rendering.
Also I hope the reader does not forget that the Reingold-Tilford Algorithm for tree nodes positioning.
Obviously, we cannot render all family members but **only direct ancestors**: parents, parents of parents, etc.

I adjusted the Reingold-Tilford Algorithm to family tree rendering and implemented it in [github/TheBestTvarynka/grafily/b9d281d3/src/layout.ts](https://github.com/TheBestTvarynka/grafily/blob/b9d281d35fe5def9a3b0b260c82375d44f4755b1/src/layout.ts).
The whole implementation tool me ~500 LoC. And this is the example of this algorithm in action:

![](./demo.jpg)

(I was too lazy to create a separate vault with non-existent persons, so I just took a small subset of my real ancestors and blurred alive ones :upside_down_face:)

As you can see from the screenshot above, the algorithm successfully positioned marriages; there are no overlaps and (almost) all marriages are aligned.
At this point, I can call it a success :star_struck: :sunglasses:.

The main limitations of this approach: **only direct ancestors allowed**. I also call it a school-level family tree.
It's impossible to render siblings, their spouses, and their spouses' families (I call them parallel families).

> _I adjusted the Reingold-Tilford Algorithm to family tree rendering..._

I did not change the algorithm itself, but add a small trick to the nodes representation.
Every tree note is actually a 3 nodes combined together: parent1 node + marriage node + parent2 node.
It was done to simplify the algorithm implementation and better edge rendering.

```ts
// https://github.com/TheBestTvarynka/grafily/blob/b9d281d35fe5def9a3b0b260c82375d44f4755b1/src/layout.ts#L13-L19
// +------------+                             +------------+
// |  parent1   |--------------o--------------|  parent2   |
// +------------+                             +------------+
//
// | NODE_WIDTH | MARRIAGE_GAP | MARRIAGE_GAP | NODE_WIDTH |
// |                    MARRIAGE_WIDTH                     |
const MARRIAGE_WIDTH = (NODE_WIDTH + MARRIAGE_GAP) * 2;
```

# Conclusions

I hope my explanation is pretty clear for you, my dear reader. I did my best to make it easier to understand :blush:.

Additionally, I would like to mention that implementing such algorithm is a great exercise to boost your programming skills.
I have not had such algorithmic tasks in years. And when I started writing the Reingold-Tilford Algorithm for my plugin, I felt a small numbness.
Most of the time I looked like this:

![](./meme.jpg)

The progress was super slow and I spent a lot of time thinking about representations of different parts of the algorithm in the code.
But in the end, I managed to get it done and even a bit proud of myself.

# References

1. [A Node-Positioning Algorithm for General Trees. John Q. Walker II. September, 1989](https://www.cs.unc.edu/techreports/89-034.pdf). This paper is awful. It was useless for me. I understand a bit more than nothing from it. I mentioned it just for curiosity.
2. [Reingold Tilford Algorithm Explained With Walkthrough](https://towardsdatascience.com/reingold-tilford-algorithm-explained-with-walkthrough-be5810e8ed93/). But this blog post is excellent :star_struck: — good explanations, nice images, and illustrations. I recommend reading it.
3. [Algorithm for Drawing Trees](https://rachel53461.wordpress.com/2014/04/20/algorithm-for-drawing-trees/). Another blog post I found, but it was confusing for me and raised more questions than it answered.
4. My Reingold-Tilford Algorithm implementation: [github/TheBestTvarynka/grafily/b9d281d3/src/layout.ts](https://github.com/TheBestTvarynka/grafily/blob/b9d281d35fe5def9a3b0b260c82375d44f4755b1/src/layout.ts).
5. [Obsidian](https://obsidian.md/) official website.
6. [myheritage](https://www.myheritage.com/) official website.
