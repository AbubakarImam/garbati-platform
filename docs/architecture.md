---
layout: default
title: Architecture
permalink: /architecture/
---

<p class="diagram-caption" style="margin-bottom:2rem;">
This page mirrors the real <code>ARCHITECTURE.md</code> from the repository root — kept in sync
automatically by <a href="https://github.com/AbubakarImam/garbati-platform/blob/main/.github/workflows/docs-sync.yml"><code>.github/workflows/docs-sync.yml</code></a>
on every push that changes it, not a hand-maintained copy (see that workflow and
<code>scripts/sync_docs.py</code> for exactly why a GitHub Action does this instead of a plain
Jekyll tag — Jekyll's own <code>include</code>/<code>include_relative</code> tags refuse any path
containing <code>..</code>, so neither can reach a file one directory above <code>docs/</code>
directly). Prefer reading the real file itself?
<a href="https://github.com/AbubakarImam/garbati-platform/blob/main/ARCHITECTURE.md">It's directly on GitHub</a>.
</p>

{% include architecture.md %}
