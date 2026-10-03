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
* The nodes traversal order it fixed.
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
Every unit knows parameters of units to the top of it and to the left or right from it.

So, what's the problem?
Why did I discard this approach?

> Every unit knows parameters of units to the top of it and to the left or right from it.

This is a problem.
In reality, genealogy graphs are way more complex.
For example, the described approach does not allow to unambiguously traverse graphs where genealogy lines are expanded to top and bottom many times.
Bleh, it sounds complex :face_with_spiral_eyes: :woozy_face:.
It's much easier to show on example:

// todo

So, I wrapped up this approach and started searching for another one.

# Brandes-Köpf



# Quadratic programming

.

# Outro

.

# References

1. [Brandes-Köpf approach implementation](https://github.com/TheBestTvarynka/grafily/blob/f7e74a5ef078ca2706c7099c80dc021c0cfb38fe/src/layout/positioning/brandesKopf.ts).
2. [The Dagre project on GitHub](https://github.com/dagrejs/dagre).
3. [Fast and Simple Horizontal Coordinate Assignment](https://scispace.com/pdf/fast-and-simple-horizontal-coordinate-assignment-2aawem94ts.pdf) (paper).
4. [QP-based approach implementation](https://github.com/TheBestTvarynka/grafily/tree/f7e74a5ef078ca2706c7099c80dc021c0cfb38fe/src/layout/positioning/quadratic).
