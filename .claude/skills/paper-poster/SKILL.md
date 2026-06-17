---
name: paper-poster
description: Use when asked to make a poster, illustrated summary, explainer page, or visual walkthrough of ONE research/academic paper — given as an arXiv ID/URL, a PDF, an image/screenshot of the paper, a LaTeX source, or just a title. Produces a self-contained HTML page with figures pulled from the paper. (To process a whole inbox of mixed sources, use the paper-notes skill, which dispatches this per paper.)
---

# Paper Poster

## Overview

Turn one research paper into a single self-contained **HTML poster** — figures + text, organized into the sections readers care about: **动机 / 方法 / 实验 / 局限性** (motivation / method / experiments / limitations). Output is a folder with `index.html` and a local `img/` directory, openable in any browser with no server.

Core principle: **get the real figures from the paper, quote the paper's own wording, and never invent results.**

**Language: Chinese (zh) only.** Body text is written in **中文**, but **keep academic terminology, proper nouns, model/method names, metric names, and quoted phrases in English** (e.g. cross-attention、point map、AUC@3°、"register attention"). Do NOT produce an English version and do NOT add a language toggle. The page is `lang="zh"`.

**Model:** Sonnet is sufficient — this is a mechanical fetch → convert → fill-template workflow, not heavy reasoning. When running it as a subagent, dispatch with `model: "sonnet"` (cheaper/faster, no quality loss). Reserve Opus only if the paper text needs unusually deep synthesis.

## Workflow

1. **Acquire the source** (see below).
2. **Extract content** — read each section, pull the key claims/numbers verbatim.
3. **Extract & prep figures** — convert/download to web PNGs, downscale the heavy ones.
4. **Generate the poster** — fill `assets/poster-template.html`, reference images via `img/`.
5. **Verify** — open it, confirm every `<img>` resolves and no result was fabricated.

## Acquiring the source

You may be handed any of: an **arXiv id/url**, a **PDF**, an **image/screenshot** of the paper, a
**LaTeX source**, or just a **title**. Whatever it is, first **identify the paper and get its arXiv
id** — `Read` a PDF/image to lift the title, or `WebSearch` a title, then verify against
`arxiv.org/abs/<id>`. Once you have the id, get the cleanest source for figures, in order of
preference: **LaTeX source** (`/e-print/<id>`, vector figures) → **arXiv HTML** (`/html/<id>`) →
the **PDF** (`pdftoppm`). If there's no arXiv version, work from the PDF/image you were given.

- **arXiv HTML exists for only some papers — and don't trust the HTTP status alone.** All variants (`/html/<id>`, `…v1/v2`) can 404; worse, arXiv serves a *soft-404* (HTTP **200** with a "no HTML available" placeholder). Verify the body actually contains the paper before scraping — grep for a distinctive chunk of the paper **title**: `curl -s -L "https://arxiv.org/html/<id>" | grep -qi 'a few words of the title'`. If absent, fall back to e-print/PDF.
- **arXiv URLs:** abstract `arxiv.org/abs/<id>`, HTML `arxiv.org/html/<id>`, LaTeX source `arxiv.org/e-print/<id>`, PDF `arxiv.org/pdf/<id>`. A filename that *looks* like an id (`2601.06338v2.pdf`) is NOT proof — verify by fetching the abstract.
- LaTeX source gives the cleanest figures (original PDFs/PNGs) and full text in `sec/*.tex`, `tables/*.tex` — prefer it when available.

## Extracting figures

**For arXiv HTML, scrape the actual `<img src>` URLs from the body — do NOT guess `xN.png` names.** The naming is inconsistent (`x1.png`, `imgs/teaser.jpg`, `img/arch.jpg` all coexist) and `x1.png` is NOT reliably Figure 1 — guessing assigns wrong captions. Pull the real `src` paths and the figure they belong to from the page, then download those:
```bash
# 1) list the real figure URLs from the HTML body (resolve relative paths against the page)
curl -s -L "https://arxiv.org/html/<id>" | grep -oE '<img[^>]+src="[^"]+"' | sed -E 's/.*src="//;s/"//'
# 2) download a chosen one (name the local file by the figure's ACTUAL format/role, not the source name)
curl -s -L -o img/architecture.png "https://arxiv.org/html/<id>/<scraped-path>"

# LaTeX/zip figures that are PDFs -> PNG (raster at ~130 dpi)
pdftoppm -png -r 130 -singlefile figures/architecture.pdf img/architecture

# PDF-only paper: rasterize specific pages, then crop visually
pdftoppm -png -r 150 -f 3 -l 3 -singlefile paper.pdf img/page3

# Downscale anything heavy (>~1MB or >1600px wide) so the page stays light
sips -Z 1600 img/qualitative.png
```

- **Confirm what each figure actually depicts** before captioning it (read its nearby `<figcaption>`/surrounding text); the source filename is not a reliable hint.
- **Don't trust the file extension.** A `.png` URL may serve a JPEG (and vice versa). Always `file img/*` and rename to the true format; broken/mismatched files surface here.
- **Pick a small, purposeful set** (~3–5): teaser + architecture + one results figure + one ablation. Download only what you reference; don't leave unreferenced files in `img/`.

Tools: `pdftoppm` (poppler), `sips` (macOS built-in), or `gs`/ImageMagick as fallback.

## Building the poster

Copy `assets/poster-template.html` to `<out>/index.html` and fill the placeholders. The template is a dark, responsive, self-contained page (inline CSS, no dependencies) with: hero/TL;DR, sticky nav, and collapsible sections for the four content blocks. Reference figures as `img/<name>.png`.

Content rules:
- **Always link the original (required).** Put a clickable link to the source paper in BOTH the header and the footer so readers reach it in one click. Prefer the arXiv abstract page `https://arxiv.org/abs/<id>`; otherwise the project page, OpenReview, or DOI. A poster without a working link back to the paper is incomplete.
- **Keywords (required).** Every poster MUST carry 6–10 search keywords (academic terms, mixed 中英 — task, method, domain, key technique, e.g. `4D reconstruction, dynamic scene, point tracking, feed-forward, transformer, 视频深度`). Put them in two places: a `<meta name="keywords" content="...">` in `<head>`, and a visible keyword-chip row in the hero. These drive retrieval in the collection index.
- **Quote the paper's own phrasing** for claims and headline numbers; mark quotes. Summarize the connective tissue in your own words. Figure captions may be paraphrased, but any **metric, baseline, or comparison must be verbatim** from the paper.
- **Don't hide unfavorable numbers.** If a baseline beats the paper on some benchmark, show it — fidelity over flattery.
- **Never fabricate** metrics, baselines, or a "Limitations" section. If the paper has no explicit limitations section, say so and label each point as *stated* vs *inferred from the design*.
- One figure per major idea: architecture under Method, qualitative/teaser up top, results tables/plots under Experiments.

## Output structure

```
<out>/<slug>/
  index.html        # the poster (中文)
  img/              # all figures, local
  meta.json         # REQUIRED sidecar — the machine-readable record of this poster
```
For a portable single file, base64-inline the images instead of an `img/` dir (larger file, but emailable).

**`meta.json` (required).** Every poster writes a sidecar so the collection index can be rebuilt deterministically (category/keywords/summary live here, NOT only in rendered HTML). Exact fields:
```json
{
  "slug": "g3t",
  "title": "G3T Up! Gravity Aligned Coordinate Frames Simplify Pointmap Processing",
  "arxiv_id": "2605.27372",
  "url": "https://arxiv.org/abs/2605.27372",
  "category": "计算机视觉 · Computer Vision",
  "keywords": ["gravity-aligned frame", "pointmap", "前馈式三维重建", "..."],
  "summary_zh": "一句话中文摘要。",
  "date": "2026-06-10",
  "source": "arxiv-html"
}
```
`category` must be exactly one of the four **broad** taxonomy strings below (大类 = coarse bucket only; fine-grained topic is expressed via `keywords`/tags, NOT via the category). `arxiv_id` = "" if none; `url` = canonical paper link (arxiv abs, else project/OpenReview). `date` = the day it was added (passed in when run head-less, since the agent has no clock).

**meta.json MUST be valid JSON** — the collection index does `json.load` on it and silently drops files that don't parse. The #1 break is a raw ASCII `"` inside a Chinese string value (e.g. `"summary_zh": "...作为"世界知识"..."` ends the string early). Inside `summary_zh`/`title`, use fullwidth quotes `“…”` (or escape as `\"`), never a bare `"`. After writing, verify: `python3 -c "import json; json.load(open('<slug>/meta.json'))"`.

## Common mistakes

| Mistake | Fix |
|---|---|
| Trusting a PDF filename or an HTTP 200 as proof of the arXiv id / HTML | Confirm the title via `arxiv.org/abs/<id>`; a soft-404 returns 200 — verify the body, else fall back to `/e-print/` or PDF |
| Inventing numbers or a Limitations section | Quote the paper verbatim; label inferred points as *inferred* |
| Embedding 8 MB figures | `sips -Z 1600` / re-raster to keep the page light |

The positive rules above already cover the rest — 中文 body, header+footer link, 6–10 keywords, relative `img/` paths — follow them.

## Category taxonomy — 4 broad buckets (大类)

`meta.json`'s `category` must be exactly one of these. Pick the single best-fitting bucket; it is
intentionally coarse. Express the paper's specific sub-topic through `keywords` (6–10 terms) — those
become the clickable filter tags on the collection page, so that is where fine-grained distinctions
live (e.g. a Gaussian-splatting paper and a segmentation paper both sit under 计算机视觉, told apart by
their keywords). Add a new bucket only if a paper fits none.

| 大类 | 涵盖（仅粗分，细分看 keywords） |
|---|---|
| 机器人 · Robotics | VLA, imitation/RL policies, manipulation, visual servoing, action tokenization, world models for control |
| 计算机视觉 · Computer Vision | 3D/4D reconstruction, pointmap, depth, SLAM, Gaussian splatting, segmentation, human/scene reconstruction |
| 生成模型 · Generative Models | diffusion, flow matching, one-step/few-step generation, GAN, physical-prior generation, video generation |
| 理论与优化 · Theory & Optimization | self-supervised representation, world models, identifiability, optimization, mechanistic interpretability, 纯理论分析 |

**Dedup before generating.** A poster is identified by its arXiv id. If one already exists for this
paper, don't duplicate it: `grep -lE '"arxiv_id"[: ]+"<id>"' <collection-root>/*/meta.json` — if it
matches, stop. (The `paper-notes` orchestration also dedups, but check when run standalone.)

The collection landing page is **generated** from all the `meta.json` sidecars — never hand-edit it.
Building/publishing the collection is the `paper-notes` skill's job (it runs the publish + index
rebuild after posters are written).
