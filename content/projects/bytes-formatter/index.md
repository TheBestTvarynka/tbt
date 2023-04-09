+++
title = "bytes-formatter"
date = 2023-04-09
draft = false

[taxonomies]
tags = ["js", "tools"]
[extra]
toc = true
+++

Visit this tool at [bf.qkation.com](https://bf.qkation.com).

{{ img(src="simple_demo.png" alt="byte-formatter demo screenshot") }}

### Motivation

During my work, I often need to convert bytes between different representations (format) like from *hex* to *decimal* and vice versa. From *hex/base64* to *ASCII*. And so on. Before I just have one small rust project on my laptop with a few dependencies and every time I inserted my new data, run this project, and copy the output. After some time it became inconvenient: I was forced to keep this project on every machine, compile it after every code change, etc. After some suffering, I wrote this user-friendly tool in pure html/css/js. The code is clear-cut and perfectly does the job.

Features:

* supported formats: `decimal`, `hex`, `base64` (both: usual and url-encoded), `ascii`, `binary`, `utf-8`, and `utf-16`.
* ability to copy only selected range of the bytes. It automatically counts the amount of the selected bytes.
* share by the link.
* integrated [asn1 parser](https://asn1.qkation.com) (available for `hex` and `base64` output types).

### How to use it

Just follow [the link](https://bf.qkation.com/) and paste your data. The interface is pretty intuitive. Anyway, it also has examples of the different inputs. Click the `Show input examples` button to see them. You'll see that you can insert even unfiltered data and this tool will do basic filtering and formatting.

### Moving forward

I already implemented the needed functionality for me. If you found the bug or have the feature request, then create a [new issue](https://github.com/TheBestTvarynka/bytes-formatter/issues/new) with the proper description.