---
name: paper-notes
description: Use when asked to process the paper inbox, generate paper notes, ingest dropped papers, or update/rebuild the paper-notes collection. Handles a mix of arXiv ids/urls, PDFs, paper screenshots, and .bib/.txt reference lists — builds one illustrated 中文 poster per paper and rebuilds the collection page.
---

# Paper Notes — inbox intake (orchestration)

Turn whatever is sitting in the paper inbox into 中文 illustrated posters, **one per paper**, then
rebuild the collection landing page. You (the main agent) orchestrate; **one Sonnet subagent builds
each poster** by following the **paper-poster** skill.

- Working dir: `_src/` (git-ignored). Inbox: `_src/_inbox/`.
- Published posters: `<slug>/`. Generated landing page: `index.html`.
- Per-paper build spec: the **paper-poster** skill.

## Principle
**Do not branch on file type.** Read whatever is in the inbox — PDFs, images/screenshots,
`.txt`/`.bib` reference lists, `queue.txt` lines, or ids/urls/titles the user pasted — and work out
which paper(s) each points to. Interpreting the modality is your job, not a fixed set of rules.

## Steps

1. **Gather inputs.** List everything in `_src/_inbox/` and read `queue.txt` (skip `#`
   comment lines). Include any ids/urls/titles the user gave directly in the request.

2. **Extract a paper list.** `Read` each item (Read handles PDF / image / text natively) and build a
   deduped list. Each entry = the best handle you can get — an **arXiv id** (preferred), else a
   title or url.
   - A `.bib`/`.txt` reference list → possibly many papers (one per entry).
   - A PDF or a screenshot → one paper (lift its title; resolve the arXiv id if it has one).
   - `queue.txt` lines / pasted text → one paper each.

3. **Dedup against the collection.** Skip papers that are already posters:
   `grep -rlE '"arxiv_id"[: ]+"<id>"' */meta.json _src/*/meta.json`
   (fall back to a title match when there's no id). Report the skipped ones; do not regenerate them.

4. **Build one poster per NEW paper — in parallel, via Sonnet subagents.** Dispatch the Agent tool
   once per paper **in a single message** (so they run concurrently), each with `model: "sonnet"`
   and a prompt like:
   > Follow the **paper-poster** skill to build EXACTLY ONE 中文 poster for `<arxiv-id|url|title>`
   > into `_src/<slug>/` (`index.html` + `img/` + `meta.json`), using date `<YYYY-MM-DD>`.
   > Pick a short lowercase slug. Handle ONLY this one paper — do NOT read the inbox or generate any
   > other paper. Print the slug when done.

   Pass today's date explicitly (subagents have no clock). Collect the slug each returns.
   For a long list, dispatch in batches rather than hundreds at once.

4b. **Validate every new `meta.json`.** Subagents sometimes emit invalid JSON (most often a raw
   ASCII `"` inside the Chinese `summary_zh`), which the index builder silently drops. Check each:
   `for m in _src/*/meta.json; do python3 -c "import json,sys;json.load(open(sys.argv[1]))" "$m" || echo "BAD: $m"; done`
   Fix any BAD one (convert interior `"` in the string value to fullwidth `“”`) or re-dispatch that
   paper. Do not publish until all new `meta.json` parse.

5. **Publish + rebuild the index.** Run `pipeline/publish_to_site.sh` — it WebP-compresses
   each new poster's figures into `<slug>/` and rebuilds `index.html`.

6. **Archive consumed inputs.** Move the inbox files you processed into `_src/_processed/`
   so a later run won't reprocess them.

7. **Report.** New posters (slug + arXiv id), skipped duplicates, and any failures.

## Notes
- Raw figure sources and the inbox live in `_src/` (git-ignored); only the published WebP
  under `<slug>/` is committed. Finish with `git add . && git commit && git push`
  (GitHub Pages redeploys).
- For a single paper handed to you directly (not via the inbox), just follow the **paper-poster**
  skill — you don't need this orchestration.
