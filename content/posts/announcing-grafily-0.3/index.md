+++
title = "Announcing Grafily v.0.3.0"
date = 2026-06-04
draft = false
template = "post.html"
description = "Obsidian plugin for rendering pretty family graphs (family trees)"

[taxonomies]
tags = ["javascript", "typescript", "project", "react", "algorithms", "data-structures"]

[extra]
keywords = "TypeScript, Graphs, Algorithms"
toc = true
mermaid = true
# thumbnail = "dataans-thumbnail.png"
+++

Short release notes: [github/TheBestTvarynka/grafily/v.0.3.0](https://github.com/TheBestTvarynka/grafily/releases/tag/v.0.3.0).

# Intro

Around a month ago I released Grafily v.0.3.0. I was really happy to reach this milestone.
Grafily is the most algorithmically challenging project so far and I had a lot of enjoyment working on it.
The fact that I use this tool personally in the scope of my own family research makes me happy.

Let me recall what is Grafily and what its purpose.
Grafily is an Obsidian plugin for rendering family relationships graphs and trees.
It scans person's pages inside the vault and builds the tree/graph based on it.

> For the sake of convenience: When I say _graph_ assume that I mean both _graph_ and _tree_ cases.
Graphs include trees.

The resulting graph is interactive.
It means that the user is able to change the graph structure, expand relationships (add nodes), or collapse them (hide), swap spouses in the marriage, and even rearrange siblings.

This article explains what have been implemented, how it works, includes a showcase, and describes plans for the future.
You can use the page index to jump to any section you are interested in.

# Philosophy

## Do one thing and do it well

The Grafily has one concrete goal, purpose: render pretty family relationships graphs.
It will never become an all-in-one genealogy research tool.
It will never become an universal graphs renderer. Or anything like that.
The Grafily follows the [Unix philosophy](https://en.wikipedia.org/wiki/Unix_philosophy#Do_One_Thing_and_Do_It_Well):

> Do one thing and do it well.

The Grafily is good in building graphs layouts.
It does not even render them, because the [`reactflow`](https://reactflow.dev/) library is used for that.

{% mermaiddiagram() %}
flowchart LR
    first["bunch of .md files"] -->|Grafily| second["Pretty graph ✨"]
{% end %}

## The Worse Is Better

Did you hear about [the _worse-is-better_ philosophy](https://www.dreamsongs.com/RiseOfWorseIsBetter.html)? If not, I encourage you to read [The Rise of Worse is Better](https://www.dreamsongs.com/RiseOfWorseIsBetter.html) article.

TL;DR. This is a citation from the mentioned article above:

> The worse-is-better philosophy:
>   - Simplicity -- the design must be simple, both in implementation and interface. It is more important for the implementation to be simple than the interface.
>   - Correctness -- the design must be correct in all observable aspects. It is slightly better to be simple than correct.
>   - Consistency -- the design must not be overly inconsistent. Consistency can be sacrificed for simplicity in some cases, but it is better to drop those parts of the design that deal with less common circumstances than to introduce either implementational complexity or inconsistency.
>   - Completeness -- the design must cover as many important situations as is practical. All reasonably expected cases should be covered. Completeness can be sacrificed in favor of any other quality. Consistency can be sacrificed to achieve completeness if simplicity is retained.

:thinking: What does it mean for the app?
It means that some features can be discarded in favor of app simplicity.
The benefits of some features may not justify their implementation complexity.
I would rather keep the app simple then unreasonably complex.

# Showcase

<sup><sub>All persons in demo screenshots below are generated using AI. If you find any coincidences with real people, please contact me, and I will fix them.</sub></sup>

# Features

- **Start-up menu.** The start-up menu shows when the user opens the plugin.
  It allows the user to either load a saved graph or set parameters and generate a new one.
  ![](./start-up_menu.png)
- **Persistance.** The user can save generated graphs and reopen them in the next session.
  It allows the user to have a different relationships graphs on hand without the need to rebuild the whole graph every time.
  The saved graphs are listed on the right side of the start-up menu.
- **Data directory configuring.** The user can configure the directory that the plugin will scan for people's data.
  ![](./settings.png)
- **Graphs layout building and rendering:**
  * Tree-baed layout called [Reingold-Tilford](#reingold-tilford).
    ![](./tree.png)
  * Graph-based layout called [Brandes-Köpf](#brandes-kopf).
    ![](./graph.png)
- **Interactivity.** The user is able to collapse/expand nodes, rearrange nodes (swap siblings, swap spouse nodes in the marriage, etc).
  UI buttons for the interactivity are located at the top right corner of the graph view and called the _side panel_:

  ![](./side-panel.png)
    1. Move the node left among its siblings.
    2. Move the node right among its siblings.
    3. Swap the selected person's place with their spouse.
    4. Show the current node: the viewpoint will move so the selected node will be in the center of the screen.
    5. Return to the start-up menu.
    6. Reload persons' data. This button will not update relationships. But it will update a person's data, such as names, images, and dates.
    7. Save the current graph.
    8. Delete the current graph.

# How it works

When the user opens the plugin, it automatically starts vault scannings for family person's pages.
It expect the vault to have one page per person.
It's not necessary to scan all `.md` files inside the vault.
The user can configure the target directory and the app will scan files only inside this dir.

![](./settings.png)

To be successfully accepted, the `.md` page must have predefined metadata at the beginning of the page:

```md
# <surname> <name>

**Spouse**: [[<spouse page>]]
**Parents**: [[<1st parent page>]], [[<2nd parent page>]]
**Birth**: <year>-<month>-<day>
**Death**: <year>-<month>-<day>
**Image**: [[<profile picture file>]]

---

Person's page content.
```

Example:

```md
# Myroniuk Pavlo

**Spouse**: [[Kateryna]]
**Parents**: [[Yaroslav]], [[Halyna]]
**Birth**: 2001-07-10
**Image**: [[TheBestTvarynka.png]]

---

I was born in the Volyn region, the westest part of Ukraine.
```

Based on the specified relationships in each person metadata, the app is able to build the full relationships graph.
You can type any information you want after the `---`. The `# <surname> <name>` line is required. All other key-value pairs are optional.
Moreover, you do not need to specify the spouse link for both; only one link is sufficient.
For example, if you specified in the metadata that Bob's spouse is Emma, then it is not required to specify Bob in Emma's metadata.

The most interesting part is how the app build the graph. It is explained in the next section :relaxed:.

## Architechture

The app works in 4 main stages:

<table style="border:none;border-collapse:collapse;table-layout:fixed;">
<colgroup>
    <col style="width: 40%;">
    <col style="width: 60%;">
  </colgroup>
<tbody>
  <tr>
    <td style="border:none;border-collapse:collapse;">
{% mermaiddiagram() %}
flowchart TD
    A[".md files"] -->|1. Parse and extract metadata| B["Index"]
    B -->|2. Build internal representation| C["Graph structure"]
    C -->|3. Calculate nodes positions| D["Nodes coordinates and edges"]
    D -->|4. Render using `reactflow`| E["Pretty graph ✨"]
{% end %}
    </td>
    <td style="border:none;border-collapse:collapse;">

1. `.md` files parsing. I already described that above, so we will not focus on it here.
2. Obviously, it's not possible to place family persons on the 2-dimensional plane.
  The entire family history can be a huge complicated graph.
  Also, the perfect nodes layout does not exist because in different cases the user wants to see different people in different positions.
  So, each layout algorithm has its own internal relationships representation.
  This representation usually contains only nodes that will be rendered on the view and their relationships (nodes edges).
  Optionally, the internal representation can contains an additional data to help it to build the resulting graph.
3. Each layout algorithm implements its own solution for calculating nodes positions (`x` and `y` coordinates).
  The third step is to calculate these coordinates.
4. And the last forth step is to create `Node[]` and `Edge[]` objects and pass them to `rectflow` view.
    </td>
  </tr>
</tbody>
</table>

At this point, the _internal representation_ can be a bit magical thing.
Let me explain it better on an example.
Let's take the Reingold-Tilford. It is the simplest layout I have.
It can render only direct ancestors and descendants of the selected person/marriage:

![](./nancy-demo.png)

Intuitively you can assume that internally it's basically a tree. 
And you will be right. In the code it looks like this ([src](https://github.com/TheBestTvarynka/grafily/blob/96b65e62e0ace284a3727ca7400cd795bd1cd02b/src/layout/tree/treeBuilder.ts#L105-L114)):

```ts
/**
 * Tree builder which allows creating and modifying family trees.
 * This tree builder can be used for parents (ancestors) and children (descendants)
 * trees generations. The implemented behaviour is abstract enough.
 */
export class TreeBuilder {
    private family: Index;
    private children: Map<string, TreeNode[]> = new Map();
    private root: TreeNode | null = null;
    private getChildNodes: (nodeId: TreeNode, family: Index) => TreeNode[];

    /* ... */
}
```

It contains a family `Index` (all family relationships), a map with connections from parent to children nodes, and the root node/
There is nothing complicated.

On the screenshot above, you could see parents tree (ancestors tree) and children tree (descendants tree) of _Nance Mordor_.
Internally, it's just two trees ([src](https://github.com/TheBestTvarynka/grafily/blob/96b65e62e0ace284a3727ca7400cd795bd1cd02b/src/layout/tree/index.ts#L30-L34)):

```ts
export class ReingoldTilford {
    private family: Index;
    private parentsTreeBuilder: TreeBuilder;
    private childrenTreeBuilder: TreeBuilder;
    private root: string | null = null;

    /* ... */

}
```

The implementation is abstracted over the `getChildNodes` function, which returns either node parents or node children.

## Persistence

The attentive reader will notice a one problem around `TreeBuilder`: it cannot be saved into the file easily because it uses complex types, like `Map`, inside.

And you will be right. To be able to correctly serialize the structure into the JSON, the object needs to be a _plain_ object and does not contain circular references.

To resolve this issue, every layout implementation implements two methods for serializing and deserializing ([src 1](https://github.com/TheBestTvarynka/grafily/blob/96b65e62e0ace284a3727ca7400cd795bd1cd02b/src/layout/tree/treeBuilder.ts#L100-L103) and [src 2](https://github.com/TheBestTvarynka/grafily/blob/96b65e62e0ace284a3727ca7400cd795bd1cd02b/src/layout/tree/index.ts#L334-L350)):

```ts
// Can be safely serialized using `JSON.stringify`.
export interface FamilyTree {
    children: Record<string, TreeNode[]>;
    root: TreeNode;
}

/**
 * Returns the layout state ready for serialization. Is it safe to stringify it to the JSON
 * and parse back again.
 * For the `ReingoldTilford`, the `data` field has `{ parentsTreeBuilder: FamilyTree, childrenTreeBuilder: FamilyTree }` type.
 *
 * @returns {SerializableLayout} - A object ready to be serialized.
 */
toSerializableObject(): SerializableLayout {
    return {
        name: REINGOLD_TILFORD,
        data: {
            parentsTreeBuilder: this.parentsTreeBuilder.familyTree(), // Returns a `FamilyTree` object.
            childrenTreeBuilder: this.childrenTreeBuilder.familyTree(),
        },
    };
}
```

Such approach allows the user to restore the layout state from the disk and continue working from the same place.

## Interactivity

Now let's talk about interactivity.
We do not want to do an extra job on every user click because otherwise the interface will be laggy.

<table style="border:none;border-collapse:collapse;table-layout:fixed;">
<colgroup>
    <col style="width: 35%;">
    <col style="width: 65%;">
  </colgroup>
<tbody>
  <tr>
    <td style="border:none;border-collapse:collapse;">
{% mermaiddiagram() %}
flowchart TD
    A["User action"] -->|1. Update internal relationships| B["Updated internal representation"]
    B -->|2. Recalculate nodes positions| C["Nodes coordinates and edges"]
    C -->|3. Rerender using `reactflow`| D["Pretty graph ✨"]
{% end %}
    </td>
    <td style="border:none;border-collapse:collapse;">

1. When the user decides to edit the graph, the corresponding action is propagates to the layout object which, in turn, edits the internal representation accordingly.
  For example, if the user expands parents of the person, new persons will be added and linked to the internal representation.
2. When the internal representation is altered, then it recalculates nodes coordinates.
  This stage will create new `Node[]` and `Edge[]` objects.
  I have two reasons for that:

    1. It's too hard to trace and edit existing `Node[]` and `Edge[]` objects.
    2. We need a new array object anyway to trigger the React component rerender.
  
  From the React component prospective, all interactivity boils down to:

  ```ts
  const newGraph = layout.action(parameters);
  setGraph(newGraph);

  // For example:
  const newGraph = layout.collapseChildren(nodeId);
  setGraph(newGraph);
  ```
3. And the last third step is to pass `Node[]` and `Edge[]` objects to the `rectflow` view.
    </td>
  </tr>
</tbody>
</table>

Any graph modifications does not require the full graph rebuild.
After any user action, only a small part (usually) of the graph is affected.

## Layout algorithms

Below are the high-lever overview.
I cannot put everything in one post.
I plan to write separate blog posts for all algorithms involved in Grafily.
Descriptions below aim to explain pros and cons of each layout type but now how they work.

### Reingold-Tilford

I already mentioned this layout type many time above.
It has such interesting name because I developed it based on the Reingold-Tilford algorithm for calculating tree nodes coordinates (tree drawing problem).
I have a separate blog post about it: [Drawing Genealogy Graphs. Part 1: Tree Drawing Using Reingold-Tilford Algorithm](https://tbt.qkation.com/posts/draw-tree-using-reingold-tilford-algorithm/).

**Pros:**

- It is a simple layout perfect for seeing one person/marriage direct ancestors or descendants.
- It is pretty looking because (almost) all nodes are centered over its children.

**Cons:**

- The simplicity is also a limitation. No way to render parallel families or join two different family trees.

**Supported interactivity:**

- The user can collapse or expand parents of any parent node starting from the selected person/marriage and up the bloodline (ancestors tree).
  ![](./tree-parents-interactivity.png)
- The user can collapse or expand children of any children node starting from the selected person/marriage and down the bloodline (descendants tree).
  ![](./tree-children-interactivity.png)
- The user can swap any marriage spouses vice versa.
  | Before swap | After swap |
  |-|-|
  | ![](./tree-before-marriage-swap.png) | ![](./tree-after-marriage-swap.png) |
- The user can rearrange sibling. It is called _to move the current node left/right_.
  | Before move left | After move left |
  |-|-|
  | ![](./tree-before-move-left.png) | ![](./tree-after-move-left.png) |

### Brandes-Köpf

The most powerful (at the moment of writing) layout type.
Internally it is a bidirectional layered graph.
Almost the most generic graph you can think of ([src](https://github.com/TheBestTvarynka/grafily/blob/96b65e62e0ace284a3727ca7400cd795bd1cd02b/src/layout/fullGraph/graphBuilder.ts#L91-L102)):

```ts
/**
 * A special class for all graph manipulations. It is used to build the initial graph and modify it when the user makes any changes.
 * This implementation does not calculate any nodes positions. Its only purpose is to modify the graph structure and layering.
 */
export class GraphBuilder {
    private nodes = new Map<string, GraphNode>();
    // string - node id.
    // string[] - list of parent node ids.
    private parents = new Map<string, string[]>();
    // string - node id.
    // string[] - list of child node ids.
    private children = new Map<string, string[]>();
    // number - layer index.
    // string[] - list of node ids in the layer.
    private layers: Map<number, string[]> = new Map<number, string[]>();
    private family: Index;

    /* */
}
```

Its name is taken from the nodes coordinates algorithm that was used: [Fast and Simple Horizontal Coordinate Assignment](https://scispace.com/pdf/fast-and-simple-horizontal-coordinate-assignment-2aawem94ts.pdf).
To be honest, I did not develop the coordinates assignment algorithm from scratch.
The implementation is highly inspired by the `dagre` project: [github/dagrejs/dagre/2595d05a0f/lib/position/bk.js](https://github.com/dagrejs/dagre/blob/2595d05a0fbb7721f35bfcaab5fbf40f5b3858ca/lib/position/bk.js).
Basically, that's the same algorithm, but rewritten in TypeScript and adapted to the current use case.
You can read about it more here:

- The [Dagre](https://github.com/dagrejs/dagre) project on GitHub.
- [github/dagrejs/dagre/wiki#recommended-reading](https://github.com/dagrejs/dagre/wiki#recommended-reading).

I do not have a separate article about it yet, but I plan to write it someday and hope this day will come :innocent:.

**Pros:**

- It is able of rendering parallel families: families of the sibling spouse, for example.
  ![](./graph.png)
- The use can edit aby part of the graph.

**Cons:**

- Not all nodes are perfect centered. Even more, almost all nodes are not perfectly centered.
  The resulting graph is not super aesthetic or perfectly rendered.
  It can annoy you a bit.
- Currently, not all possible graph modification are implemented but they are at least possible and planned to be implemented :wink:.

**Supported interactivity:**

- The user can collapse/expend any parents of any person.
- The user can collapse/expand children of any marriage.
- The user can swap spouses of any marriage.
- Just like in the tree case, the user can rearrange sibling (_to move the current node left or right_).

As you can see, this layout is extremely flexible and powerful.

![](./graph-interactivity.png)

In reality, the Brandes-Köpf layout usage is not that simple.
The user must follow the rules to be able to modify the graph. There are some of them.

The moving the current node left and right actions have limitations:

- The selected node and the neighbor node in the move direction must not have children nodes (children nodes must be collapsed).
- The selected node and the neighbor node in the move direction must have only one parent connection: the common parent node.
  Spouses parents (if any) must be collapsed.

Because it is allowed for children nodes to have parallel families, it is not easy to determine how to swap sub-graphs of two siblings and avoid edges crossing.
Of course, I am sure I could create some smart-ass graph walking algorithm which will remember the borders of both sub-graphs and then swap them, but I thought about that for a moment and decided that it is not worth the trouble.
I decided to add limitations above in favor of implementation simplicity (see [Philosophy#the-worse-is-better](#the-worse-is-better) section).
The same story with the spouses swap action limitations:

- Maximum one spouse of the target marriage is allowed to have parents expanded.

# What's next?

I have two big directions of development:

1. First, I need to implement more possible modifications in the Brandes-Köpf layout.
  My idea is simple: if you can think of any family graph on 2D plain without edges crossing, then this graph should be possible to recreate in Grafily.
2. Currently, only small subset of families are supported: only one marriage per person, only one mom and one dad per person.
  What about adoption, step-dad/mom and their previous families, remarriage?
  I do not know what is the best way to support all possible families types and peoples relationships.
  I have only blurred image in my head but I am sure I will figure something out :wink:.

# Conclusions

I said it before and I say it again: writing software for yourself and **using** it is an amazing feeling.
I like the result, I like how it grows, I like the fact that I am using it.

Rendering people relationships is a hard task and there is no way to implement it correctly or perfectly.
Every feature has its tradeoffs and limitations.

I learned a lot about graphs algorithms and coordinates assignment algorithms.
I significantly improved my skills.
The first steps during implementation were super slow and uncertain.
Besides its huge usefulness in my genealogy research, the Grafily is also a great exercise of my programming skills.

# References

1. GitHub release [github/TheBestTvarynka/grafily/v.0.3.0](https://github.com/TheBestTvarynka/grafily/releases/tag/v.0.3.0).
2. [Drawing Genealogy Graphs. Part 1: Tree Drawing Using Reingold-Tilford Algorithm](https://tbt.qkation.com/posts/draw-tree-using-reingold-tilford-algorithm/).
3. [github/dagrejs/dagre/wiki#recommended-reading](https://github.com/dagrejs/dagre/wiki#recommended-reading).
