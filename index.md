# Roscoe MCP

Semantic-search infrastructure for historical primary sources, served over
the [Model Context Protocol](https://modelcontextprotocol.io/). Roscoe lets
researchers query court opinions, Congressional debates, founders'
correspondence, newspapers, and other archival material by *meaning* rather
than by keyword, and returns results with direct links to the authoritative
source — no fabricated content, no model-generated citations.

The public project lives at **[roscoeproject.org](https://www.roscoeproject.org/)**.

| Surface | URL | What it is |
|---|---|---|
| Website | [www.roscoeproject.org](https://www.roscoeproject.org/) | Marketing site, mission, team, demos |
| RID browser | [rid.roscoeproject.org](https://rid.roscoeproject.org/) | Web UI for browsing collections and chatting against them |
| MCP aggregator | `app.roscoeproject.org` | The multi-collection MCP endpoint (Bearer-token auth) |

This repository holds the implementation. The source is private; this
README is published via GitHub Pages as the only public-facing artifact of
the repo itself.

## What's inside

Roscoe is a collection of independent **MCP servers**, one per source
corpus, fronted by an **aggregator** that exposes them as a single MCP
endpoint. A separate FastAPI **RID browser** provides a web UI for humans.
Each collection is a self-contained unit with its own metadata, embeddings
index, and search semantics.

The retrieval stack is intentionally boring:

- **DuckDB** for structured metadata
- **hnswlib** for approximate nearest-neighbour vector search
- **sentence-transformers** for embedding generation
- **FastMCP** for the MCP protocol surface
- **Cloudflare Tunnel** for public exposure without opening ports

Collections currently shipped:

| Collection | Source | Coverage |
|---|---|---|
| `caselaw` | CourtListener | U.S. state appellate and early federal opinions, 1754–1865 |
| `congrecord` | congress.gov | Congressional Record proceedings and debates |
| `chronamer` | Library of Congress, *Chronicling America* | Pre-1865 American newspapers (182k+ pages, OCR) |
| `founders` | Founders Online (National Archives) | Correspondence and papers of the founding generation |
| `lawmaking` | LOC, *A Century of Lawmaking* | Continental Congress through the 43rd Congress, 1774–1875 |
| `taney` | UVA Law / FromThePage | Roger Brooke Taney's pre-bench legal papers, 1792–1820 |

Each collection declares its own capabilities (`passage_in_search`,
`full_text_available`, `has_passage_meta`) in `collection.yaml`, so clients
can negotiate behaviour instead of assuming it.

## Identifiers (RIDs)

Every retrievable object has a permanent **Roscoe Identifier** of the form
`<collection-id>/<object-id>` (e.g. `1trshidr/abc123xy`), with mappings to
a DOI form (`10.XXXX/1trshidr.abc123xy`) and to an MCP URI
(`mcp://roscoe/1trshidr`). RIDs are stable across re-indexing and never
re-issued.

## Notable improvements since inception

The repository was opened on **24 December 2025**. Selected milestones
since then:

### Architecture & retrieval

- Refactored from one-off scripts into a **base-class + factory** pattern
  (`BasicRetriever`, `BaseMCPServer`) so adding a collection is now a
  manifest plus a small subclass, not a fresh codebase.
- Introduced **declared capabilities** in `collection.yaml` and centralised
  collection discovery; no module in shared code knows the collection list
  ahead of time.
- Added a **HierarchicalMetadataStore** to model works ↔ passages for
  collections whose ingest unit is larger than a single search hit.
- Standardised the **passage API** across collections and made
  `MAX_PASSAGE_CHARS` and other token-sensitive values configurable per
  collection rather than hard-coded.
- Externalised embedding-model configuration from `roscoe.env` into each
  `collection.yaml`, so collections can ship with their own model choice.

### Aggregator, chat, and the RID browser

- Switched the aggregator to a **non-blocking, resilient** model that
  survives individual collection-server failures and adds startup
  validation.
- Added **streaming** to the chat surface and fixed the
  turn-continuation/tool-use loop that previously dropped responses
  mid-stream.
- Multi-user session management with auto-save.
- Scholar / Analysis / Brief mode selector for chat responses.
- Replaced Formspree with a self-hosted contact-form backend that issues
  signed tokens.

### Collection-specific work

- **`chronamer`**: Read OCR directly from local bulk LOC files instead of
  the (slow, rate-limited) LOC API; results now include the OCR text
  in-band, with a `get_page_text` tool for full pages. CSV-driven
  newspaper-name migration for the LOC 2025 URL changes.
- **`caselaw`**: Renamed from `case-law` and migrated onto the standard
  metadata schema; year-filter bug fixed.
- **`congrecord`**: Permalinks fixed; full speech text stored in
  metadata so `passage` is populated in search responses.

### Reliability & operations

- Pre-1865 RID browser used to take **20 seconds to start** because of the
  IPv6 "Happy Eyeballs" fallback; HTTP clients now force IPv4 via
  `httpx.HTTPTransport(local_address="0.0.0.0")`.
- Lazy imports for `sentence_transformers` and `hnswlib` so MCP server
  startup is sub-second instead of multi-second.
- Fixed an MPS/asyncio deadlock that occasionally hung the embedding
  pipeline on Apple Silicon; added an index-recovery tool.
- Centralised logging: every component writes through a single
  configuration, MCP server stderr is redirected to log files instead of
  the parent's stdout, and there's NFS-based log fetching for the deployed
  fleet.
- Usage logging for every MCP tool call.
- Test-result tracking across runs for trend analysis and flaky-test
  detection.

### Infrastructure & deployment

- **Cloudflare Tunnel** stood up for all three public hostnames; `make
  setup-tunnel` provisions a fresh tunnel idempotently.
- **Environment switching** (`dev`, `test`, `prod`) for tunnel config and
  paths, with portable `~`-style paths.
- Linux-server hosting spec and deployment guide so the stack can be
  reproduced outside the original developer's Mac.
- Cloudflare configuration exported into the repo and audited; security
  tests cover both the surface and the live hostnames.

### Testing

- Migrated from ad-hoc scripts to **pytest** with a generic
  semantic-search test framework that every collection plugs into.
- Per-collection semantic tests (`founders`, `lawmaking`, `chronamer`,
  `congrecord`, `taney`).
- Infrastructure smoke tests that exercise the live hosts.
- Negative quality tests for LLM prompts, and YAML ↔ HTML selector tests
  for the website.

### Skills & developer surface

- Skills framework following Claude best practices, with deployment and
  config tooling.
- Claude Code slash commands for the project, plus a `/precommit` skill
  that runs the gate (rebuild venv, tests, code review, copy to user's
  repo, draft commit message).

### Ingest & pipeline (early 2026)

- Self-rendering HTML diagram of the collection ingest/load pipeline
  added under `docs/`, with explicit task tracking for the unfinished
  pieces.

## Timeline of major milestones

A walk through the commit log, condensed to the changes that moved the
project rather than every fix.

**December 2025 — foundation**

- *24–25 Dec:* Repository opened; `uv` project initialised. First two
  collections stood up — `congrecord` (bulk-loaded into MetadataStore)
  and `case-law`. MCP-initialisation timeout in `congrecord` fixed.
- *26 Dec:* MCP aggregator added. Chronicling America ingester landed.
- *27 Dec:* **HierarchicalMetadataStore** for works ↔ passages.
  **Vault** for secrets. **Taney** collection. Lazy imports for
  `sentence_transformers` / `hnswlib` cut MCP startup from
  multi-second to sub-second. Parallel downloads for `chronamer`.
  `case-law` migrated onto the standard metadata schema. First base
  class for MCP servers.
- *28 Dec:* `lawmaking` collection added. **BaseRetriever** and MCP
  server factory introduced — the inheritance pattern that lets a new
  collection ship as a manifest plus a small subclass. Aggregator made
  non-blocking. AnnasArchive provenance ripped out across the codebase.
- *29 Dec:* MOBI support, file-format detection, `anna` directory
  renamed to `library`. `roscoe.py` becomes a static launcher.
- *30 Dec:* First chat server.
- *31 Dec:* Startup health-check; config system rewritten.

**January 2026 — hardening, tunnel, chat polish**

- *2 Jan:* `pytest` test suite and package-structure fix. `case-law` →
  `caselaw` rename. **Cloudflare Tunnel** wired up; aggregator made
  resilient with explicit startup validation. Chat gained streaming
  and a share button. `chronamer` switched from the LOC API to local
  bulk OCR files (in-band passage text, with `get_page_text` for
  full pages); CSV-driven newspaper-name migration for the LOC 2025
  URL change. `congrecord` permalinks fixed. HNSW
  small-index crash patched. Scholar / Analysis / **Brief** chat
  modes (Brief is default). Multi-user session management with
  auto-save.
- *3 Jan:* **Environment switching** (`dev` / `test` / `prod`) for
  tunnel config and paths. `make setup-tunnel` automates idempotent
  tunnel provisioning.

**January 2026 — public website**

- *10 Jan:* New website (`website/`) with build-and-push tooling.
  Roscoe Pound bio page. Two-level menu and onboarding flow. First RID
  documentation.
- *11 Jan:* Build-date stamp on the site. **Skills framework**
  following Claude best practices. **Self-hosted contact-form backend
  with signed tokens** — Formspree retired.
- *12 Jan:* Skills deployment + config docs + comprehensive tests.
  **Generic semantic-search test framework**, then per-collection
  semantic tests for `founders`, `lawmaking`, `chronamer`,
  `congrecord`, `taney`. Negative quality tests for LLM prompts.
  Infrastructure smoke tests against the live hosts. Cloudflare
  configuration exported into the repo with a security audit.
  Architecture diagram with expandable modal added to the website.
  Voting-rights demo page.
- *13 Jan:* NFS log fetching, centralised config. Gzip
  Content-Length bug fixed. Security tests. MetadataStore API
  consistency pass; RID documentation expanded.

**January 2026 — refactor and operational polish**

- *16–17 Jan:* Claude Code slash commands installed. Alex Waisman
  added as a contributor. `BaseRetriever` renamed to `BasicRetriever`
  with `_enrich()` / `_filter()` hook points. Collection MCP servers
  moved into `collections/`. Standardised passage API + type-safe
  config methods (`get_int`, `get_bool`, `get_path`). YAML ↔ HTML
  selector tests for the website.
- *18 Jan:* Contact forms moved to an NFS CGI script. **Usage
  logging** for every MCP tool call. **MCP docstrings improved for
  LLM usability**. Embedding-model configuration externalised from
  `roscoe.env` into each `collection.yaml`. Chat tool-use
  continuation fixed.
- *19 Jan:* Index-recovery tooling. **MPS / asyncio deadlock** on
  Apple Silicon fixed. Subprocess progress reporting for
  embeddings.
- *20 Jan:* **Centralised logging** rolled out across all components.
  Agent server infrastructure + Cloudflare config. Agent search with
  metrics and configurable `k`. **Test-result tracking** across runs
  for trend analysis and flaky-test detection. LM Studio default
  model bumped to Qwen2.5-32B. TODO list converted to CSV. **20-second
  RID startup delay** fixed (IPv4-only HTTP, Happy-Eyeballs bypass).
  Agent prompts externalised; chat defaults tuned.
- *21 Jan:* MCP streaming reliability + turn-continuation logic
  hardened.
- *22 Jan:* Collection ingest/load pipeline diagram (self-rendering
  HTML) added under `docs/`, with task tracking for the unfinished
  pieces.

**February–March 2026 — paper-test evaluation**

- A separate evaluation repository (`roscoe-paper-test`) was built to
  measure `caselaw` against a researcher-validated gold set drawn from
  Pasker (2025), *"Which History Has Condescended to Notice: Black
  Testimony in Antebellum Courts"* ([SSRN
  5416774](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=5416774)).
  See the **Evaluation** section below.
- *22 Mar:* Linux-server hosting specification and deployment guide
  added so the stack can be reproduced outside the original
  developer's laptop.

**April 2026**

- Brief experiment with an embedded email-workflow agent, reverted
  the same day.

## Evaluation against a gold set

In early 2026 the `caselaw` collection was put through a formal
evaluation in a separate, private repository:
[`rbpasker/roscoe-paper-test`](https://github.com/rbpasker/roscoe-paper-test).
The evaluation exists to answer one question: *does semantic search
recover the same antebellum opinions that a historian found through
months of manual work, and does it bridge the gap between modern and
period terminology that defeats keyword search?*

The harness is a Jupyter notebook (`roscoe_evaluation.ipynb`) calling
the deployed Roscoe MCP server directly over streamable HTTP — no LLM
in the loop, so the numbers measure the retriever, not an interpreter
on top of it.

**Inputs**

- **Gold set** — 20 cases programmatically extracted from the Table
  of Cases in the SSRN paper, with CourtListener opinion IDs (and
  alternate IDs where CourtListener has multiple records for the
  same opinion). The gold set is a single JSON file
  (`gold_set_caselaw.json`) and is the durable contract for
  comparing future runs.
- **Query battery** — 21 queries in three categories: 4 SSRN-baseline
  queries (the exact Boolean strings used in the paper), 10 concept
  queries (freedom suits, fugitive-slave law, witness competency,
  manumission, …), and 7 contested-category pairs (modern term vs.
  period term — *"Black testimony"* vs. *"negro witness competency"*,
  *"homosexual"* vs. *"crime against nature"*, and similar).
- **Baseline** — CourtListener REST v4 keyword search,
  date-restricted to 1790–1865.

**Metrics**

| Metric | What it answers |
|---|---|
| Precision @ 10 | Of the 10 top results, how many are in the gold set? |
| Aggregate recall | Across the whole battery, how much of the gold set surfaces? |
| Jaccard overlap | For modern ↔ period query pairs, how interchangeable are the result sets? |
| Permalink resolution | Does every cited opinion still exist at its URL? |
| Metadata match | Does the returned case name / court / date agree with CourtListener? |

**Error taxonomy** — every non-gold result is classified into one of
five buckets so failures are actionable rather than aggregate:
*relevant-but-uncited*, *semantic drift*, *near-neighbour noise*,
*unitisation artifact*, *period-language miss*. Classifications
persist to `error_taxonomy.json` and survive re-runs. Auto-classification
uses three independent signals — TF-IDF cosine similarity,
sentence-level co-occurrence of racial × testimony terms, and a
spaCy NER-context heuristic — combined and presented for human
override.

**What it surfaced**

The evaluation is the source of `roscoe_improvements.md`, a punch
list of platform changes flagged by the actual numbers rather than
speculation. Two examples that drove real work in the implementation
repo:

- **R1 — Topic-field artifacts.** Ingestion-time prefixes (`*28admissibility`,
  bullet markers) were embedded as-is, contaminating the vectors.
  Fix: strip and normalise in the loader, re-embed affected documents.
- **R2 — `passage: null` in 4 of 6 collections.** Only `chronamer`
  and `congrecord` populated the in-search passage field; the other
  four returned `null`, which blocks client-side rerankers, snippet
  display, and offline classification. Fix has two halves: store the
  embedded passage text in the metadata `data` field at ingest, and
  set `passage_in_search: true` with a matching `_enrich()` override
  on each collection.

This is the only place in the project where retrieval quality is
measured against an independent ground truth. Subsequent platform
work is sequenced behind it.

## Repository layout

```
collections/       # One sub-directory per collection (mcp_server.py + collection.yaml + data)
common/            # Shared retriever, aggregator, metadata, RID, config
agent/             # Agent server (chat, prompts, session management)
website/           # Source for www.roscoeproject.org
cloudflare/        # Tunnel configuration
docs/              # Internal architecture / testing / deployment docs (private)
scripts/           # Operational scripts (deploy, tunnel setup, etc.)
skills/            # Claude Code skills shipped with the project
.github/workflows/ # Sync this README to the public landing page on every change
```

The page you are reading lives in a separate **public** repository,
[`rbpasker/roscoe-mcp-site`](https://github.com/rbpasker/roscoe-mcp-site),
which holds nothing but a rendered copy of this README. A workflow
(`.github/workflows/sync-readme-to-site.yml`) keeps the two in sync via
an SSH deploy key scoped to that one repo. GitHub Pages on a private repo
is a paid-plan feature; the sibling-repo pattern is the standard
free-tier workaround.

## Status & licence

Roscoe is research infrastructure under active development. The
implementation repository is private; collections, the public website, and
the RID browser are open to the world. Licensing of the implementation is
not yet finalised — if you are interested in deploying Roscoe for an
archive or institution, the right starting point is the contact form on
[roscoeproject.org](https://www.roscoeproject.org/#contact).
