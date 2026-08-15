+++
title = "Drawing Genealogy Graphs. Part 2: Genealogy Graph Interactivity"
date = 2026-08-16
draft = false
template = "post.html"
description = "This post describes a set of algorithms implemented inside the [Grafily](https://github.com/TheBestTvarynka/grafily/) plugin for building and dynamically altering genealogy graphs"

[taxonomies]
tags = ["algorithms", "data-structures", "typescript", "drawing-genealogy-graphs-series"]

[extra]
keywords = "Algorithm, Graphs, Data structures, Algorithms, Depth-first search, Graph building"
toc = true
# thumbnail = "dgg1-thumbnail.png"
+++

# Intro

This is a second post in a series of posts about building and rendering genealogy graphs.
You can see all posts in the series here: [/tags#drawing-genealogy-graphs-series](https://tbt.qkation.com/tags/#drawing-genealogy-graphs-series).

In [the first post](https://tbt.qkation.com/posts/draw-tree-using-reingold-tilford-algorithm/), I described the Reingold-Tilford algorithm for positioning tree nodes (calculating `(x; y)` for each tree node).
In the same post I said that trees as data structure for representing relatives is inconvenient because it simply does not allow to see siblings of ancestors or parallel families.

So, I moved on and implemented a set of algorithms for building and rendering genealogy graphs.
Graph is a way complicated data structure then a tree, so I decided to split it into two posts:

- the first post (the one you are currently reading) dedicated to the graph building and interactivity;
- the second one about node positioning algorithm.

It's too much for one post.
Good, let's start :rocket:

{% note_info_block() %}
All persons below are generated using AI. If you find any coincidences with real people, please contact me, and I will fix them.
{% end %}

Before we move on, I want to remind you that I made one simplification and follow it till this day: the graph node is either a person node or a marriage node.
Structurally, there is no difference between person node and marriage node.
Both of them are just nodes.

{{ img(src="node-types.png" alt="Node types" class="ci b1")}}

# What is genealogy graph?

If you say that **genealogy graph** is a graph that represents family relationships, ancestors, descendants, etc, then you will be 100% correct.
What properties such a graph has?

1. Every node has at most two parent nodes. Every node can have any number of children.
2. Horizontally, all nodes are split into layers. Every node has a layer.
3. Every edge connects two nodes of adjacent layers.
   That's not true for all cases, but I do not plan to support even in the future.
4. Edges are not crossing - this constraint is added by me.

The last property is not true by default.
If we try to include **all** people on the big-enough graph and render all of them at the same time, we will eventually have edges crossing.
In some cases it's just not possible to render all nodes without edges crossing.

But keep in mind that I do not set a purpose to render all persons at the same time.
My goal is to give the user an ability to construct any graph they have in mind.
So, that okay if I forbid edges crossing.
Interactivity solves this limitation perfectly :wink:.

# What is interactivity?

In general terms, interactivity is the ability to interact or to communicate with a system.
The user can do some actions to the system and it will respond accordingly.
But what does it mean for the genealogy graph?

I define genealogy graph interactivity as a set of actions that allow the user to edit the graph as they want.
It includes:

* Building the initial graph starting from the selected person.
  {{ img(src="initial-graph.png" alt="Initial graph")}}
* Removing (collapsing) any parents/children from the graph.
  {{ img(src="nodes-collapsing.png" alt="Collapsed nodes")}}
* Adding (expanding) person parents or marriage children.
  {{ img(src="nodes-expanding.png" alt="Nodes expanding")}}
* Ability to swap spouses inside the marriage.
  {{ img(src="swapped-spouses.png" alt="Swapped spouses")}}
* Ability to rearrange siblings of the marriage.
  {{ img(src="siblings-rearrangement.png" alt="Siblings rearrangement")}}

The set of actions defined above allow us to build a genealogy graph of any complexity.
In the following sections I explain how every of these actions work, what drawbacks and constraints I put on them and why.

# 

# Conclusions

# References

1. 
