---
layout: default
title: Research
permalink: /research/
---

# Research

Every question a researcher agent was asked to check before a decision got made — regulation,
Sharia product rules, banking practice, library choices. Each one is a sourced brief, not an
opinion; read `ARCHITECTURE.md`'s [decisions log]({{ '/architecture/' | relative_url }}#8-platform-decisions-log)
for how a given brief actually changed (or didn't change) a decision.

<p class="diagram-caption">
This list is generated from the real <code>research/</code> directory by
<code>scripts/sync_docs.py</code>, run automatically by
<a href="https://github.com/AbubakarImam/garbati-platform/blob/main/.github/workflows/docs-sync.yml"><code>.github/workflows/docs-sync.yml</code></a>
on every push that touches <code>research/**</code> — a GitHub Actions workflow, not a plain Jekyll
tag, because Jekyll's own build has no way to list a directory outside its <code>docs/</code>
source (see that script's docstring). It currently lists {{ site.data.research_index | size }} files.
</p>

<table>
<thead>
<tr><th>Brief</th></tr>
</thead>
<tbody>
{% for item in site.data.research_index %}
<tr>
<td>
<a href="https://github.com/AbubakarImam/garbati-platform/blob/main/research/{{ item.filename }}">{{ item.title }}</a>
<br><code>research/{{ item.filename }}</code>
</td>
</tr>
{% endfor %}
</tbody>
</table>
