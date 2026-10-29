---
layout: default
title: Home
permalink: /
---

<section class="hero" markdown="1">

# Open-source core banking, built for Sharia-compliant microfinance.

<p class="thesis">GARBATI is six real services behind a real double-entry ledger — account opening,
transfers, fees, reversal — not a mockup, not a pitch deck. Every number on this page is a real,
current count from the project's own working files; every design decision behind the platform is
written down and public.</p>

<div class="cta-row">
  <a class="btn btn-primary" href="{{ '/replicate/' | relative_url }}">Run it yourself &rarr;</a>
  <a class="btn" href="{{ '/architecture/' | relative_url }}">Read the architecture</a>
</div>

<div class="stat-strip">
  <div class="stat stat-highlight">
    <span class="stat-value">6</span>
    <span class="stat-label">Services shipped</span>
  </div>
  <div class="stat">
    <span class="stat-value">50 / 52</span>
    <span class="stat-label">Tasks closed</span>
  </div>
  <div class="stat">
    <span class="stat-value">28</span>
    <span class="stat-label">Decisions logged</span>
  </div>
  <div class="stat">
    <span class="stat-value">13</span>
    <span class="stat-label">Research briefs</span>
  </div>
</div>
<p class="stat-asof">As of 29 Oct 2026 — counted directly from <code>TASKS.md</code> and <code>research/</code> at the time this page was written. Not a live counter; see <a href="https://github.com/AbubakarImam/garbati-platform/blob/main/TASKS.md">TASKS.md on GitHub</a> for the current numbers.</p>

</section>

<div class="ledger-rule"><span>Architecture at a glance</span></div>

Six services, each the single owner of one kind of truth. The orchestrator decides the order of
every customer operation and is the only caller that posts money to the ledger; the ops console
never talks to a service directly — every staff action goes through `OpsApi`, which holds the real
credentials. Solid arrows are the customer path; dashed arrows are the staff (ops console) path.

<div class="diagram-wrap">
<svg viewBox="0 0 900 600" role="img" aria-labelledby="arch-title arch-desc" xmlns="http://www.w3.org/2000/svg">
  <title id="arch-title">GARBATI service map</title>
  <desc id="arch-desc">Ops console calls OpsApi, which calls StaffIdentityApi, IdentityApi, OrchestratorApi and LedgerApi. OrchestratorApi calls IdentityApi, ProductApi and LedgerApi. Only OrchestratorApi posts money to LedgerApi.</desc>
  <style>
    .node rect { fill: var(--surface); stroke: currentColor; stroke-width: 1.5; }
    .node text { fill: currentColor; font-family: "Work Sans", sans-serif; font-size: 14px; font-weight: 600; }
    .node .sub { font-family: "Work Sans", sans-serif; font-size: 10.5px; font-weight: 400; opacity: 0.75; }
    .edge { stroke: currentColor; stroke-width: 1.4; fill: none; }
    .edge.staff { stroke-dasharray: 5 4; opacity: 0.8; }
    .edge-label-bg { fill: var(--ground); }
    .edge-label { font-family: "IBM Plex Mono", monospace; font-size: 10.5px; fill: var(--ink); }
    .legend text { font-family: "Work Sans", sans-serif; font-size: 11px; fill: var(--ink); }
  </style>
  <defs>
    <marker id="arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="currentColor"></path>
    </marker>
  </defs>

  <!-- customer path -->
  <path class="edge" marker-end="url(#arrow)" d="M155,186 L145,270"></path>
  <path class="edge" marker-end="url(#arrow)" d="M250,175 Q320,220 355,270"></path>
  <path class="edge" marker-end="url(#arrow)" d="M200,186 Q320,330 400,420"></path>

  <!-- staff (ops console) path -->
  <path class="edge staff" marker-end="url(#arrow)" d="M745,76 L745,140"></path>
  <path class="edge staff" marker-end="url(#arrow)" d="M745,186 L745,270"></path>
  <path class="edge staff" marker-end="url(#arrow)" d="M650,153 Q400,90 150,270"></path>
  <path class="edge staff" marker-end="url(#arrow)" d="M650,170 L250,170"></path>
  <path class="edge staff" marker-end="url(#arrow)" d="M700,186 Q600,330 500,420"></path>

  <!-- edge labels (with a ground-colored backing rect so crossing lines don't obscure them) -->
  <g>
    <rect class="edge-label-bg" x="98" y="218" width="112" height="15"></rect>
    <text class="edge-label" x="104" y="229">who may do this?</text>

    <rect class="edge-label-bg" x="255" y="216" width="150" height="15"></rect>
    <text class="edge-label" x="261" y="227">rules, fees, tax, limits?</text>

    <rect class="edge-label-bg" x="245" y="330" width="170" height="15"></rect>
    <text class="edge-label" x="251" y="341">post the legs (money moves only here)</text>

    <rect class="edge-label-bg" x="610" y="100" width="150" height="15"></rect>
    <text class="edge-label" x="616" y="111">session cookie, every request</text>

    <rect class="edge-label-bg" x="610" y="218" width="178" height="15"></rect>
    <text class="edge-label" x="616" y="229">who is this staff member, what role?</text>

    <rect class="edge-label-bg" x="330" y="128" width="190" height="15"></rect>
    <text class="edge-label" x="336" y="139">customer / KYC / mandate lookups</text>

    <rect class="edge-label-bg" x="390" y="150" width="200" height="15"></rect>
    <text class="edge-label" x="396" y="161">account opening, transfers, reversals</text>

    <rect class="edge-label-bg" x="545" y="300" width="160" height="15"></rect>
    <text class="edge-label" x="551" y="311">balance / transaction lookups</text>
  </g>

  <!-- nodes -->
  <g class="node">
    <rect x="650" y="30" width="190" height="46" rx="5"></rect>
    <text x="745" y="49" text-anchor="middle">Ops console</text>
    <text class="sub" x="745" y="63" text-anchor="middle">browser, React + TS</text>
  </g>

  <g class="node">
    <rect x="60" y="140" width="190" height="46" rx="5"></rect>
    <text x="155" y="159" text-anchor="middle">OrchestratorApi</text>
    <text class="sub" x="155" y="173" text-anchor="middle">step order, decisionId</text>
  </g>

  <g class="node">
    <rect x="650" y="140" width="190" height="46" rx="5"></rect>
    <text x="745" y="159" text-anchor="middle">OpsApi</text>
    <text class="sub" x="745" y="173" text-anchor="middle">staff BFF, holds real keys</text>
  </g>

  <g class="node">
    <rect x="60" y="270" width="170" height="46" rx="5"></rect>
    <text x="145" y="289" text-anchor="middle">IdentityApi</text>
    <text class="sub" x="145" y="303" text-anchor="middle">KYC, mandates, auth</text>
  </g>

  <g class="node">
    <rect x="270" y="270" width="170" height="46" rx="5"></rect>
    <text x="355" y="289" text-anchor="middle">ProductApi</text>
    <text class="sub" x="355" y="303" text-anchor="middle">rules, fees, tax, limits</text>
  </g>

  <g class="node">
    <rect x="650" y="270" width="190" height="46" rx="5"></rect>
    <text x="745" y="289" text-anchor="middle">StaffIdentityApi</text>
    <text class="sub" x="745" y="303" text-anchor="middle">staff roles</text>
  </g>

  <g class="node">
    <rect x="330" y="420" width="220" height="50" rx="5"></rect>
    <text x="440" y="441" text-anchor="middle">LedgerApi</text>
    <text class="sub" x="440" y="456" text-anchor="middle">accounts, entries, balances</text>
  </g>

  <!-- legend -->
  <g class="legend">
    <line x1="60" y1="530" x2="100" y2="530" stroke="currentColor" stroke-width="1.4"></line>
    <text x="108" y="534">customer path</text>
    <line x1="60" y1="555" x2="100" y2="555" stroke="currentColor" stroke-width="1.4" stroke-dasharray="5 4" opacity="0.8"></line>
    <text x="108" y="559">staff / ops console path</text>
  </g>
</svg>
</div>
<p class="diagram-caption">Matches <a href="{{ '/architecture/' | relative_url }}">ARCHITECTURE.md</a> sections 2–3 at the time this page was written. This diagram is hand-drawn, not generated — if the real service map changes later, it needs a manual update to match (unlike the Architecture and Replicate pages below, which stay in sync with the real files automatically, no manual update needed).</p>

<div class="ledger-rule"><span>How to replicate it yourself</span></div>

Clone every service repo as a sibling of `garbati-platform`, then from this repo:

```bash
docker compose up --build
```

One command builds all five services' images, starts five PostgreSQL databases, runs every
service's migrations, and brings up five real APIs wired to call each other — no manual API-key
setup, no `localhost` port juggling. The full runbook — the sibling-repo layout, the port table, the
seeded admin bootstrap step — lives on the <a href="{{ '/replicate/' | relative_url }}">Replicate It</a> page.

<div class="ledger-rule"><span>Want the full history?</span></div>

<div class="further-reading">
<ul>
  <li><a href="{{ '/research/' | relative_url }}">Research</a> — every question a researcher agent was asked to check, with sources.</li>
  <li><a href="https://github.com/AbubakarImam/garbati-platform/blob/main/TASKS.md">TASKS.md on GitHub</a> — the full task board and every decision in the owner's own words. It's a working log, not introductory reading, so it lives on GitHub rather than as a page here.</li>
  <li><a href="https://github.com/AbubakarImam/garbati-platform/blob/main/LEARNING_LOG.md">LEARNING_LOG.md on GitHub</a> — a plain-language explanation of every merged feature, with interview questions and answers.</li>
</ul>
</div>
