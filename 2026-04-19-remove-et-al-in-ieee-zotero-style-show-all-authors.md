---
title: Remove “et al.” in IEEE Zotero Style (Show All Authors)
slug: remove-et-al-in-ieee-zotero-style-show-all-authors
date_published: 2026-04-19T09:30:18.000Z
date_updated: 2026-04-19T09:30:18.000Z
excerpt: This guide shows a simple configuration tweak in the CSL style editor to disable “et al.” and force full author display, without needing any programming knowledge.
---

I got a revision from my committee about reference style. I’m using Zotero with the default IEEE citation style configuration. By default, if authors are more than 7, it prints the first author followed by “et al.”. The committee requires avoiding “et al.” and showing all authors.

Here’s a simple trick. No programming needed. It’s just a small config change.

### Example (Default Output)

[1] A. Verbitski et al., “Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases,” in Proceedings of the 2017 ACM International Conference on Management of Data, Chicago Illinois USA: ACM, May 2017, pp. 1041–1052. doi: 10.1145/3035918.3056101.

### Steps

On Zotero, click Edit → Settings → Cite → IEEE reference guide → Style Editor.

In the Style Editor, change `et-al-min="7"` to `et-al-min="100"`.

There are 2 occurrences of `et-al-min`, usually around line 125 and 143. Use Ctrl + F to find them.

### Style Configuration

Update the metadata to avoid overwriting the default style.

    <title>IEEE Reference Guide version 11.29.2023</title>
    <title-short>Institute of Electrical and Electronics Engineers</title-short>
    <id>http://www.zotero.org/styles/ieee</id>
    

Change it to:

    <title>IEEE Full Author</title>
    <title-short>IEEE Full Author</title-short>
    <id>IEEE Full Author</id>
    

### Apply the Style

Save as a new file, for example IEEE Full Author.csl.

Then go back to Zotero, click the plus button, import the file, and select it as your citation style.

### Example (After Fix)

[1] A. Verbitski, A. Gupta, D. Saha, M. Brahmadesam, K. Gupta, R. Mittal, S. Krishnamurthy, S. Maurice, T. Kharatishvili, and X. Bao, “Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases,” in Proceedings of the 2017 ACM International Conference on Management of Data, Chicago Illinois USA: ACM, May 2017, pp. 1041–1052. doi: 10.1145/3035918.3056101.
