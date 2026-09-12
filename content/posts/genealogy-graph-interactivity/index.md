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

# Interactivity explained

## Nodes adding

I am going to start from nodes adding because this part is the most interesting and the most complicated one.
Lets run through a few examples of possible cases.
After that I will describe the final algorithm.

Look at the graph below. Where should we insert parents of Adam Crosby?

{{ img(src="adam-crosby.png" alt="adam-crosby.png")}}

Obviously, right after the parents of Karen Crosby.
Okay, let's take something more difficult.
Where should we insert parents of Nancy Mondor?

{{ img(src="nancy-mondor.png" alt="nancy-mondor.png")}}

Apparently, we have two option: insert parent nodes between Elizabeth and Arthur or between Helen and Richard.
What should we choose?
Our intuition says to choose the second option.
Okay, what if I change the graph a bit: I will add Arthur and Helen children to the graph.

{{ img(src="nancy-mondor-2.png" alt="nancy-mondor-2.png")}}

Now we have only one option: between Elizabeth and Arthur.

How the algorithm decide where to insert parent nodes?
What if parents also have their own parents, should we add them too?
So many options and possibilities :face_with_spiral_eyes:



## Nodes rearrangement

The hard thing was to decide what functionality to sacrifice in to simplify the development.
In the end, nodes rearrangement is implemented very simply.

{% note_info_block() %}
When the user wants to swap spouses of the marriage, **parents of at most one spouse may be expanded** (can be present).
{% end %}

When both spouses have no parents or when only one of them has parents expanded, then swapping spouses is trivial - we can just swap nodes and call it a day.

When both spouses have parents expanded, then it's not trivial.
Because it is a graph, spouse parents can have many ancestors and that ancestors can have a lot of descendants.
For instance, how would you swap Robert Mondor and Alica Mondor on the following graph?

{{ img(src="complicated-spouses-swapping.png" alt="Complicated spouses swapping")}}

To swap them, you would need to remove all nodes from the one side, swap spouses, and then insert all that nodes on the other side.
I do not say that it is not possible, but I do say that it is too complicated.
Even more, I can create an example where swapping spouses without a farther clarifications is impossible or at least would be unpredictable for the end user.

A roughly the same situation with moving the sibling node to the left/right.
We cannot swap two sibling nodes if both of them has descendants.
Descendants sub-graphs can be too complicated to deal with. But it is easy and possible to swap two sibling nodes when no one or only one of them has descendant nodes.

## Removing nodes

Removing nodes is not hard: you ✨ _just_ ✨ recursively walk thought the graph and remove nodes from it.

# Conclusions

Often too much complicated algorithms not worth the gain.

Often we should sacrifice some part of functionality to simplify the implementation ([_the worse-is-better_](https://www.dreamsongs.com/RiseOfWorseIsBetter.html)).

# References

1. Implementation: [github/TheBestTvarynka/grafily/b8dc4e9331/src/layout/builder/index.ts](https://github.com/TheBestTvarynka/grafily/blob/b8dc4e93316f8b8bb2b51b9bf702634dc86b2936/src/layout/builder/index.ts).
2. Try it yourself: [`GETTING_STARTED.md`](https://github.com/TheBestTvarynka/grafily/blob/master/doc/GETTING_STARTED.md).
