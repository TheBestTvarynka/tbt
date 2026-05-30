+++
title = "Announcing Grafily v.0.3.0"
date = 2026-06-04
draft = false
template = "post.html"
description = "Obsidian plugin for rendering pretty family graphs (family trees)"

[taxonomies]
tags = ["js", "ts", "project", "react", "algorithms", "grafily"]

[extra]
keywords = "TypeScript, Graphs, Algoritjms"
toc = true
mermaid = true
# thumbnail = "dataans-thumbnail.png"
+++

[github/TheBestTvarynka/grafily/v.0.3.0](https://github.com/TheBestTvarynka/grafily/releases/tag/v.0.3.0)

# Intro

Around a month ago I released Grafily v.0.3.0. I was really happy to reach this mailstone.
Grafily is the most algorithmically challenging project so far and I had a lot of enjoyment working on it.
The fact that I use this tool personally in the scope of my own family research makes me happy.

Let me recall what is Grafily and what its purpose.
Grafily is an Obsidian plugin for rendering family relationships graphs and trees.
It scans person's pages inside the vault and builds the tree/graph based on it.

> For the sake of convenience: When I say _graph_ assume that I mean both _graph_ and _tree_ cases.
Graphs include trees.

The resulting graph is interactive.
It means that the user is able to change the graph structure, expand relationships (add nodes), or collapse them (hide).

This article explains what have been implemented, how it works, includes a showcase, and describes plans for the future.
You can use the page index to jump to any section you are interested in.

# Philosophy

The Graphily has one congreate goal, purpose: render pretty family relationships graphs.
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

# Features

- **Start-up menu.** The start-up menu shows when the user opens the plugin.
  It allows the user to either load a saved graph or set parameters and generate a new one.
  ![](./start-up_menu.png)
- **Persistance.** The user can save generated graphs and reopen them in the next session.
  It allows the user to have a different relationships graphs on hand without the need to rebuild the whole graph every time.
  The saved graphs are listed on the right side of the start-up menu.
- **Data directory configuring.** The user can configure the directory that the plugin will scan for people's data.
- **Graphs layout building and rendering:**
  * Tree-baed layout called [Reingold-Tilford](#reingold-tilford).
    ![](./tree.png)
  * Graph-based layout called [Brandes-Köpf](#brandes-kopf).
    ![](./graph.png)
- **Interactivity.** The user is able to collapse/expand nodes, rearrange nodes (swap siblings, swap spouse nodes in the marriage, etc).
  UI buttons for the interactiviry are located at the top right corner of the graph view and called the _side panel_:

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
It's not neccessary to scan all `.md` files inside the vault.
The user can configure the target directory and the app will scan files only inside this dir.

// TODO: target dir settings

To be successfully accepted, the `.md` page must have predefined metadata at the beggining of the page:

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

Based on the specified relationships in each person metadata, the app is able to build the full relashionships graph.
You can type any information you want after the `---`. The `# <surname> <name>` line is required. All other key-value pairs are optional.
Moreover, you do not need to specify the spouse link for both; only one link is sufficient.
For example, if you specified in the metadata that Bob's spouse is Emma, then it is not required to specify Bob in Emma's metadata.

The most interesting part is how the app build the graph. It is explained in the next section :relaxed:.

## Architechture

<table style="border:none;border-collapse:collapse;">
  <tr>
    <td style="border:none;border-collapse:collapse;">
{% mermaiddiagram() %}
flowchart TD
    A[".md files"] -->|Parse and extract metadata| B["Index"]
    B -->|Build internal representation| C["Graph structure"]
    C -->|Calculate nodes positions| D["Nodes coordinates and edges"]
    D -->|Render using `reactflow`| E["Pretty graph ✨"]
{% end %}
    </td>
    <td style="border:none;border-collapse:collapse;">
So, the app works in 4 main stages:

1. `.md` files parsing. I already described that above, so we will not focus on it here.
    </td>
  </tr>
</table>



## Layout algorithms

### Reingold-Tilford

### Brandes-Köpf

# Showcase

# Conclusions

# References
