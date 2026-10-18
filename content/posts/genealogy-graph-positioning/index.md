+++
title = "Drawing Genealogy Graphs. Part 3: Genealogy Graph Positioning"
date = 2026-10-13
draft = false
template = "post.html"
description = "This post describes two approaches implemented inside the [Grafily](https://github.com/TheBestTvarynka/grafily/) plugin for positioning genealogy graph nodes"

[taxonomies]
tags = ["algorithms", "data-structures", "typescript", "drawing-genealogy-graphs-series"]

[extra]
keywords = "Algorithm, Graphs, Data structures, Algorithms, Nodes positioning"
toc = true
# thumbnail = "dgg2-thumbnail.png"
+++

# Intro

This is the third post in a series about building and rendering genealogy graphs.
You can see all posts in the series here: [/tags#drawing-genealogy-graphs-series](https://tbt.qkation.com/tags/#drawing-genealogy-graphs-series).

In [the first post](https://tbt.qkation.com/posts/draw-tree-using-reingold-tilford-algorithm/), I described the Reingold-Tilford algorithm for positioning tree nodes (calculating `(x; y)` for each tree node).

In [the second post](https://tbt.qkation.com/posts/genealogy-graph-interactivity/), I described what is genealogy graph interactivity and how it works inside the Grafily plugin.

In the first post, I said that trees as a data structure for representing relatives is inconvenient because it simply does not allow you to see siblings of ancestors or parallel families.
So, graphs.
What's with them?
We have a graph that represents a family relationships with other families.
This graph can be huge and of any form.
How can we calculate each node coordinates so we can render this graph?
In this post, I will try to answer this question and explain my thoughts about it.

Oh, one more thing:

{% note_info_block() %}
All persons below are generated using AI. If you find any coincidences with real people, please contact me, and I will fix them.
{% end %}

# Reingold-Tilford++

My first intuitive approach was to improve the Reingold-Tilford algorithm.
It supports only trees in its original version.
But I wanted to adapt it to graphs and develop an universal graph node positioning algorithm.
Surprisingly, I had some (partial) success.
I was able to render such graphs:

{{ img(src="rt++.png" alt="tr++" class="ci b1")}}

(The screenshot above is around 8 month old. At the moment of writing this post, the plugin UI has changed a lot)

How I did it?
Let me try to explain quickly.
First, you need to understand how the original Reingold-Tilford work.
I already explained it in my blog here: [Drawing Genealogy Graphs. Part 1: Tree Drawing Using Reingold-Tilford Algorithm](https://tbt.qkation.com/posts/draw-tree-using-reingold-tilford-algorithm/).

The Reingold-Tilford algorithm works because of two things:

* The Reingold-Tilford Algorithm uses a depth-first tree traversal.
  It means that when we calculate node `preX`, `mod`, and `shift` values, these values are already calculated for all child nodes.
* The nodes traversal order is fixed.
  When we calculate the node `x` coordinate, all child nodes and all nodes to the left from the current node are already centered and we can calculate the current node properties (`preX`, `mod`, and `shift`).

Overall, I could reuse `preX`, `mod`, and `shift` parameters and apply then to a graph.
I only need to follow two requirements above.

And I did it: [https://github.com/TheBestTvarynka/grafily/pull/1/changes](https://github.com/TheBestTvarynka/grafily/pull/1/).

1. I defined a unambiguous way to walk through the graph.
2. I simplified the task: I decided to join siblings into one unit (I called it `SiblingsUnit`).
   And instead of centering a lot of nodes, the algorithm centered sibling units.

Let's look at the image below:

{{ img(src="sibling-units.png" class="ci b1")}}

Instead of centering nodes, the algorithm centers purple blocks (sibling units).
The algorithm reuses the same `preX`, `mod`, and `shift` parameters but applies them to sibling units instead of nodes.
Now let's see the traversing order:

{{ img(src="sibling-units-traversing-order.png" class="ci b1")}}

There are always a starting person.
Starting from the starting person (:laughing: :rofl:), the algorithm first calculates coordinates for the left half of the graph, and then for the right one.
It uses the depth-first search: it starts from the topmost rightmost node of the left half and moves to the left and to the bottom.
Pay attention to the numbers inside sibling units.
These numbers show the order of `x` coordinate calculation for the sibling unit.
At the moment when we calculate parameters of the unit 4, we already know parameters of unit 3 and unit 2.
When it calculates parameters of unit 20, it already knows parameters of units 16 and 19.

Pay attention that in the left half it visits rightmost nodes first (from right to left), and in the right half it visits leftmost nodes first (from left to right).

That way I guaranteed depth-first traversal and unambiguous way to walk through the graph.
**Every unit knows parameters of units to the top of it and to the left or right from it.**

So, what's the problem?
Why did I discard this approach?

{% note_info_block() %}
_**Every unit knows parameters of units to the top of it and to the left or right from it.**_
{% end %}

This is a problem.
In reality, genealogy graphs are way more complex.
For example, the described approach does not allow to unambiguously traverse graphs where genealogy lines are expanded to top and bottom many times.
Bleh, it sounds complex :face_with_spiral_eyes: :woozy_face:.
It's much easier to show on example:

{{ img(src="impossible-case.png" class="ci b1")}}

In order to find coordinates for orange nodes, I algorithm needs to know coordinated for red solid nodes.
But red solid nodes coordinates will be calculated only when the algorithm will enter the red dashed nodes.
But the algorithm cannot enter red dashed nodes until it finishes with orange ones.
We have a contradiction here: one node depends on another and another node depends on the one node.
There is not way to resolve this unambiguously :face_with_spiral_eyes:.

Overall, I can crate a special version of depth-first search especially for that case, but it will not work for any kind of graphs.
So, I wrapped up this approach and started searching for another one.

# Brandes-Köpf

Fast enough I found the Brandes-Köpf algorithm.
It was introduced in [Fast and Simple Horizontal Coordinate Assignment](https://scispace.com/pdf/fast-and-simple-horizontal-coordinate-assignment-2aawem94ts.pdf) paper.
It says _simle_ but do not trust it.
My experience says that there is no easy papers to read :upside_down_face:.

I spent a few evenings exploring it.
It looked like a right call.
I decided to try it.

I am not going to explain in details how it work.
Overall, it works in three stages:

// todo

I did not want to implement it from scratch.
I decided to look up for open-source implementation.
I did not find a generic enough implementation that would suit my needs.
But I found vey good open-source implementation of the Brandes-Köpf algorithm inside the [dagree](https://github.com/dagrejs/dagre) project: [github/dagrejs/dagre/2595d05a0f/lib/position/bk.js](https://github.com/dagrejs/dagre/blob/2595d05a0fbb7721f35bfcaab5fbf40f5b3858ca/lib/position/bk.js).
Back then I took that algorithm, rewrote it to TypeScript, and adapted to my case.
Soon after, the dagree project also rewrote it to TS.
But I decided to keep my implementation because it was already adapted and worked well.

I loaded a few tests and it looked like a right call.
I was pleased by the result:

{{ img(src="bk-example.png" class="ci b1")}}

Since there is a third option, you may already guessed that this is not enough.
And you are right.
In some cases, the Brandes-Köpf approach produces very poor layouts.
Just look:

{{ img(src="bad-bk.png" class="ci b1")}}

You can clearly see a vertical alignment and when it does not work well.
That extra space between sibling nodes is caused by nodes belonging to different classes (horizontal compaction step in Brandes-Köpf algorithm).
At first, I thought that this is a rare case and I eventually will get used to it.
Unfortunately, I saw it too often and did not get used to it.
So, I decided to look for an alternate approach.

# Quadratic programming

I spent many hours trying to adapt algorithm I knew.
I took many tries to find an existing one.
Eventually, I gave up and decided to go _all-vibe_.

LLM has suggested me many possible approached to improve the layout calculation.
There were Quadratic Programming (QP), physic simulation (edges are rubbers and pull nodes), etc.
The QP was the easiest one to start with.
Under the _easiest_ term I mean the less code to generate and integrate into the existing system.
Physic simulation also looks like a overkill for such a small problem.

{% note_info_block() %}
**Quadratic programming (QP)** is the process of solving certain mathematical optimization problems involving quadratic functions.
Specifically, one seeks to optimize (minimize or maximize) a multivariate quadratic function subject to linear constraints on the variables.

[wikipedia/Quadratic_programming](https://en.wikipedia.org/wiki/Quadratic_programming)
{% end %}

In general terms, there is a certain function (math equation) that defined the layout (edges lengths or node's x coordinate relative to other nodes).
And the unified solver tried to solve that equation.
I know nothing about QP and I want all in with vibe-coding.

First, I built a test playgroung for testing graph node-positioning algorithms: [github/TheBestTvarynka/graph-layout-playground](https://github.com/TheBestTvarynka/graph-layout-playground).
It was inconvenient to experiment in grafily repo because I wanted an easy way to inject a test graph and see its layout.
Then, I comprehensively described all my requirements for the desired graph layout and nodes positions.
Then I added 3 to 4 test graphs of different complexity and structure.
And finally asked the Claude Code (CC) to implement a QP-based layout algorithm.

Honestly, it did a great job and I liked it a lot.
The resulting graph layout looks better than all my previous tries almost from the first try.
I added more test and and more instructions.
CC has improved the code and I finally was satisfied with the result.

Of course, the final layout algorithm is not perfect either.
There is no _perfect_ positioning algorithm at all.
The current one is _good enough_ for 95% of cases.
For the rest 5% I recommend using the Brandes-Köpf algorithm.

| QP | BK|
|-|-|
| {{ img(src="qp-big-example.png" class="ci b1")}} | {{ img(src="bk-example.png" class="ci b1")}} |

From comparison above you can see that QP-based approach produces more compact and more pretty graph layouts.
As I said above, the QP-based algorithm work better in 95% cases (from my experience).
When the graph starts looking more like a simple tree, the BK algorithm will be better:

| QP | BK|
|-|-|
| {{ img(src="q3.png" class="ci b1")}} | {{ img(src="bk3.png" class="ci b1")}} |

# Outro

At the moment, I am fully satisfied with the Grafily graph rendering.
Using Brandes-Köpf and Quadratic algorithms I can get a readable and pretty graph.

The Grafily plugin keeps and will keep both Brandes-Köpf and Quadratic algorithms.
The user can switch between them in any time.

# References

1. [Brandes-Köpf approach implementation](https://github.com/TheBestTvarynka/grafily/blob/f7e74a5ef078ca2706c7099c80dc021c0cfb38fe/src/layout/positioning/brandesKopf.ts).
2. [The Dagre project on GitHub](https://github.com/dagrejs/dagre).
3. [Fast and Simple Horizontal Coordinate Assignment](https://scispace.com/pdf/fast-and-simple-horizontal-coordinate-assignment-2aawem94ts.pdf) (paper).
4. [QP-based approach implementation](https://github.com/TheBestTvarynka/grafily/tree/f7e74a5ef078ca2706c7099c80dc021c0cfb38fe/src/layout/positioning/quadratic).
