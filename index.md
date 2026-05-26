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
| `books` | Various (with Google Scholar metadata) | Academic monographs chunked into passages, Chicago 18 footnotes |
| `jstor` | JSTOR | Academic journal articles, full-text passages with citations |

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
- Added a **HierarchicalMetadataStore** to model works ↔ passages (used by
  `books`).
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
- **`books`**: Permalinks added to search results, citation instructions
  cleaned up, incomplete books excluded from listings, MOBI / multi-format
  support.
- **`congrecord`**: Permalinks fixed.

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

- Ingest/load pipeline diagram (self-rendering HTML) added under `docs/`,
  with explicit task tracking for the un-finished pieces.

For the full history, browse the commit log in this repository.

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
