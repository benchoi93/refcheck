# refcheck

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

**Academic reference verification for AI assistants.**

A [Claude Code](https://claude.com/claude-code) skill plus a small CLI that checks academic citations against real publication databases. It catches hallucinated references, finds real papers on a topic, and exports publication-ready BibTeX.

> **v0.2 drops the MCP server.** refcheck is now a skill that calls the `refcheck` CLI through Bash. That means no long-running server, no MCP config, and the CLI works the same way in a plain terminal. If you used `refcheck-mcp`, remove its `mcpServers` entry and follow the steps below.

## The Problem

Large language models frequently hallucinate academic citations: plausible-looking references with fabricated titles, authors, DOIs, or venues. A single hallucinated citation in a paper or grant proposal can undermine credibility.

refcheck gives your assistant direct access to Crossref, Semantic Scholar, arXiv, Scopus, and IEEE Xplore. Every reference it returns is backed by a real database record.

## What It Does

| Command | Purpose |
|---------|---------|
| `refcheck verify` | Check if a citation is real. Returns `verified`, `partial_match`, or `not_found` with a confidence score and corrected BibTeX. |
| `refcheck search` | Find real papers on a topic. Never fabricates; only returns database-backed results. |
| `refcheck bibtex` | Export publication-ready BibTeX by DOI, title, or Semantic Scholar ID. Supports batch export. |

## Quick Start

### 1. Install the CLI

```bash
pip install git+https://github.com/benchoi93/refcheck.git
```

Or from source: `git clone https://github.com/benchoi93/refcheck.git && pip install -e refcheck`. Requires Python 3.10+.

### 2. Install the skill

In Claude Code:

```
/plugin marketplace add https://github.com/benchoi93/refcheck.git
/plugin install refcheck@refcheck
```

Or, without the plugin system, copy the skill directory:

```bash
mkdir -p ~/.claude/skills && cp -r skills/refcheck ~/.claude/skills/
```

### 3. (Optional) Add API keys

```bash
mkdir -p ~/.config/refcheck && cp .env.example ~/.config/refcheck/.env   # then edit
```

See [Configuration](#configuration).

### 4. Use it

Ask Claude Code naturally. The skill triggers on citation, BibTeX, and paper-finding requests:

```
> Verify this reference: Li, Y., Yu, R., Shahabi, C., & Liu, Y. (2018).
  Diffusion convolutional recurrent neural network: Data-driven traffic
  forecasting. ICLR 2018.

> Find me 5 recent papers on physics-informed neural networks for traffic
  state estimation and get their BibTeX.

> Verify all references in my refs.bib and replace any incorrect entries
  with the corrected BibTeX.
```

Or call the CLI directly. Each command takes one JSON argument and prints JSON:

```bash
refcheck verify '{"title": "Attention is all you need", "year": 2017}'
refcheck search '{"query": "traffic state estimation", "max_results": 5, "year_from": 2020}'
refcheck bibtex '{"doi": "10.1016/j.trc.2024.104873"}'
```

## Commands

### `refcheck verify`

Checks whether a citation is real by querying multiple databases and fuzzy-matching the results.

**Parameters** (JSON keys):

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `title` | `str` | At least one of `title` or `doi` | Paper title |
| `authors` | `list[str]` | No | Author names (last names sufficient) |
| `year` | `int` | No | Publication year |
| `doi` | `str` | At least one of `title` or `doi` | DOI string |
| `venue` | `str` | No | Journal or conference name |

**Verification pipeline:**

1. **DOI lookup** (if provided) — Resolves via Crossref. A DOI match is near-certain verification.
2. **Title search** — Queries all available databases in parallel.
3. **Fuzzy matching** — Scores each candidate on title similarity (token set ratio), author overlap (Jaccard on last names), year match (+/-1 tolerance), and venue similarity.
4. **Verdict** — Weighted composite score produces a classification:

| Verdict | Score | Meaning |
|---------|-------|---------|
| `verified` | >= 0.85 | High confidence the reference is real and accurate |
| `partial_match` | 0.50 - 0.85 | A similar paper exists but some fields differ |
| `not_found` | < 0.50 | No match in any database. Likely hallucinated. |

**Automatic BibTeX correction:** When the verdict is `verified` or `partial_match`, the response includes a `corrected_bibtex` field with the correct BibTeX entry from the matched paper. This means a single call both detects errors and provides the fix.

**Example response (partial match):**

```json
{
  "verdict": "partial_match",
  "confidence": 0.72,
  "matched_reference": {
    "title": "Attention is All You Need",
    "authors": ["Ashish Vaswani", "Noam Shazeer", "..."],
    "year": 2017,
    "doi": "10.48550/arXiv.1706.03762",
    "venue": "Advances in Neural Information Processing Systems",
    "url": "https://doi.org/10.48550/arXiv.1706.03762"
  },
  "discrepancies": ["Year mismatch: claimed 2018, found 2017"],
  "sources_checked": ["crossref", "semantic_scholar", "arxiv"],
  "corrected_bibtex": "@article{vaswani2017attention,\n  author = {Vaswani, Ashish and ...},\n  title = {Attention is All You Need},\n  year = {2017},\n  doi = {10.48550/arXiv.1706.03762},\n}"
}
```

### `refcheck search`

Searches for real papers across all configured databases. Results are deduplicated by DOI and title similarity.

**Parameters** (JSON keys):

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `query` | `str` | *required* | Search topic or keywords |
| `max_results` | `int` | 10 | Number of results (1-30) |
| `year_from` | `int` | None | Earliest publication year |
| `year_to` | `int` | None | Latest publication year |
| `databases` | `list[str]` | All available | Which databases to query |

**Each result includes:** title, authors, year, DOI, venue, abstract (truncated), URL, and source database.

### `refcheck bibtex`

Generates verified BibTeX entries. Prefers Crossref content negotiation (publisher-quality BibTeX) and falls back to metadata construction when needed.

**Parameters** (JSON keys, provide one):

| Parameter | Type | Description |
|-----------|------|-------------|
| `doi` | `str` | Single DOI |
| `dois` | `list[str]` | Multiple DOIs (batch mode) |
| `title` | `str` | Paper title (searches and verifies first) |
| `semantic_scholar_id` | `str` | Semantic Scholar paper ID |

**Example — batch DOI export:**

```bash
refcheck bibtex '{"dois": ["10.1016/j.trc.2024.104873", "10.48550/arXiv.1706.03762"]}'
```

The `bibtex` field holds concatenated entries ready to append to a `.bib` file:

```bibtex
@article{vaswani2017attention,
  author        = {Vaswani, Ashish and Shazeer, Noam and Parmar, Niki and ...},
  title         = {Attention is All You Need},
  journal       = {Advances in Neural Information Processing Systems},
  year          = {2017},
  doi           = {10.48550/arXiv.1706.03762},
  url           = {https://doi.org/10.48550/arXiv.1706.03762},
}
```

**BibTeX conventions:**
- Citation keys follow `{lastname}{year}{keyword}` format (e.g., `vaswani2017attention`)
- Acronyms and proper nouns are wrapped in `{}` to preserve capitalization
- Entry types are inferred from metadata (`@article`, `@inproceedings`, `@misc`, etc.)
- Fields are never fabricated. Missing data (e.g., page numbers) is omitted rather than guessed.

## Databases

refcheck queries multiple academic databases in parallel. It works out of the box with no API keys — Crossref and arXiv are free and unauthenticated. Adding optional keys unlocks additional sources and better rate limits.

| Database | Key Required | Coverage | Notes |
|----------|:---:|----------|-------|
| **Crossref** | No | 156M+ DOI-registered publications | DOI backbone. Set `CROSSREF_EMAIL` for faster rate limits. |
| **Semantic Scholar** | No | 200M+ papers | Broadest coverage. Free API key recommended. |
| **arXiv** | No | 2.5M+ preprints | Essential for recent ML/CS work. 3s rate limit enforced. |
| **Scopus** | Yes | 90M+ records | Elsevier API key + optional institutional token. |
| **IEEE Xplore** | Yes | IEEE/IET publications | Free tier: 200 requests/day. |

refcheck runs in **degraded mode** when keys are missing — it skips unavailable APIs and reports which sources were checked.

## Configuration

All keys are optional. Precedence, highest first: the process environment, `./.env`, `~/.config/refcheck/.env`, then the repo-root `.env` (editable installs only).

```bash
# Strongly recommended — enables Crossref "polite pool" (~50 req/sec)
CROSSREF_EMAIL=you@university.edu

# Recommended — free key from https://www.semanticscholar.org/product/api
S2_API_KEY=your-s2-key

# Optional — Elsevier API key from https://dev.elsevier.com/
ELSEVIER_KEY=
# Optional — Elsevier institutional token (higher rate limits)
ELSEVIER_INSTTOKEN=

# Optional — free key from https://developer.ieee.org/
IEEE_API_KEY=
```

## Architecture

```
refcheck/
├── .claude-plugin/
│   ├── plugin.json            # Claude Code plugin manifest
│   └── marketplace.json       # Single-plugin marketplace (for /plugin marketplace add)
├── skills/refcheck/SKILL.md   # The skill: when and how to call the CLI
├── src/refcheck/
│   ├── cli.py                 # `refcheck` entry point: verify / search / bibtex
│   ├── clients/               # Crossref, Semantic Scholar, arXiv, Scopus, IEEE
│   ├── matching.py            # Fuzzy matching: title, author, year, venue scoring
│   ├── bibtex.py              # BibTeX generation, formatting, citation keys
│   ├── models.py              # Pydantic v2 output models
│   └── config.py              # API key loading
└── pyproject.toml
```

**Key design decisions:**
- **Async throughout** — All API calls use `httpx.AsyncClient` with 10-second timeouts.
- **Parallel queries** — Databases are queried simultaneously via `asyncio.gather`.
- **Graceful degradation** — One failing API never blocks the others. Errors go to stderr; stdout is always valid JSON and lists which sources were checked.
- **Never fabricates** — If no match is found, refcheck says so. BibTeX fields are left out rather than guessed.

## Use Cases

- **Writing papers** — Verify every citation before submission. Catch wrong years, misspelled authors, or entirely fabricated references. Partial matches return corrected BibTeX automatically.
- **Fixing .bib files** — Verify references in bulk and replace incorrect entries with database-backed corrected BibTeX in a single pass.
- **Literature reviews** — Search for real papers on a topic and get BibTeX in one step.
- **Grant proposals** — Ensure all referenced prior work actually exists.
- **Teaching** — Help students verify citations in their assignments.
- **Batch BibTeX export** — Collect DOIs from a draft and export a clean `.bib` file.

## Limitations

- **Coverage gaps** — Some niche publications may not appear in the queried databases. A `not_found` result does not guarantee the paper is fake; it may exist in databases not covered (e.g., PubMed for biomedical, DBLP for CS).
- **Metadata quality** — BibTeX constructed from metadata (when Crossref content negotiation is unavailable) may be missing fields like page numbers or volume.
- **Rate limits** — arXiv enforces a 3-second delay between requests. IEEE free tier is limited to 200 requests/day.
- **No Google Scholar** — There is no official Google Scholar API. The existing coverage from Crossref + Semantic Scholar + arXiv is broader and more reliable.

## License

MIT

## Citation

If you use refcheck in your research workflow, a mention is appreciated:

```
refcheck: Academic Reference Verification for AI Assistants
https://github.com/benchoi93/refcheck
```
