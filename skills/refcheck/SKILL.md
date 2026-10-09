---
name: refcheck
description: Verify academic references, search for real papers, and generate BibTeX entries using Crossref, Semantic Scholar, arXiv, Scopus, and IEEE. Use when verifying citations, checking a .bib file for hallucinated or wrong entries, finding real papers on a topic, or generating BibTeX from a DOI or title. Not for literature reviews or topic surveys.
---

# refcheck — Academic Reference Verification

Never write a citation from memory. Verify it, or find a real one, with the `refcheck` CLI via Bash.

```bash
refcheck <verify|search|bibtex> '<json_args>'
```

stdout is always valid JSON. Per-database errors (rate limits, 403s) go to stderr and do not abort the run.

## Setup check

If `refcheck` is not on `PATH`, install it, then retry:

```bash
pip install git+https://github.com/benchoi93/refcheck.git
```

API keys are optional and are read from `./.env`, `~/.config/refcheck/.env`, or the environment: `CROSSREF_EMAIL` (recommended), `S2_API_KEY`, `ELSEVIER_KEY`, `ELSEVIER_INSTTOKEN`, `IEEE_API_KEY`. Without keys, Scopus and IEEE are skipped.

## `verify` — is this citation real?

```bash
refcheck verify '{"title": "Attention is all you need", "authors": ["Vaswani", "Shazeer"], "year": 2017, "venue": "NeurIPS"}'
```

| Param | Type | Notes |
|-------|------|-------|
| `title` | str | one of `title`/`doi` required |
| `doi` | str | fastest, near-certain match |
| `authors` | list[str] | last names are enough |
| `year` | int | ±1 tolerance |
| `venue` | str | journal or conference |

Returns `verdict` (`verified` ≥0.85, `partial_match` 0.50–0.85, `not_found` <0.50), `confidence`, `matched_reference`, `discrepancies`, `sources_checked`, and `corrected_bibtex` (for verified or partial matches).

`not_found` means "not in these databases", not proof that the paper is fake. Report it as unverified; don't silently drop or invent a replacement.

## `search` — find real papers

```bash
refcheck search '{"query": "physics-informed traffic state estimation", "max_results": 5, "year_from": 2020}'
```

| Param | Type | Notes |
|-------|------|-------|
| `query` | str | required |
| `max_results` | int | default 10 |
| `year_from`, `year_to` | int | filters |
| `databases` | list[str] | subset of `crossref`, `semantic_scholar`, `arxiv`, `scopus`, `ieee` |

Returns `results` (title, authors, year, doi, venue, abstract, url, source), deduplicated by DOI.

## `bibtex` — export BibTeX

```bash
refcheck bibtex '{"doi": "10.1016/j.trc.2021.103091"}'
refcheck bibtex '{"dois": ["10.1016/j.trc.2021.103091", "10.1287/trsc.2024.0525"]}'
```

Provide one of `doi`, `dois`, `title`, or `semantic_scholar_id`. Returns `bibtex` (paste-ready), `entries`, and `warnings`. Crossref-native BibTeX is used when there's a DOI; otherwise the entry is built from metadata, and a warning says so.

## Workflows

**Check a .bib file:** read it → for each entry, `verify` with title + DOI (+ authors/year) → report each discrepancy → offer to swap in `corrected_bibtex`. For long files, leave out `ieee` (200 requests/day limit).

**Cite something on a topic:** `search` → pick the best match → `bibtex` with its DOI.

## Choosing databases

| Need | Use |
|------|-----|
| DOI lookup | `crossref` alone |
| Transportation / engineering / Elsevier journals | `crossref`, `scopus`, `semantic_scholar` |
| IEEE conferences (ITSC, IV, T-ITS) | `ieee` (sparingly) |
| Recent ML preprints | `arxiv`, `semantic_scholar` |
| General | omit `databases` (query all) |
