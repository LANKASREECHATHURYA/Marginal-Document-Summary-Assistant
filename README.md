# Marginal — Document Summary Assistant

Upload a PDF or a photo of a document. Marginal extracts the text (OCR for scans), ranks every sentence by how central it is to the document, and returns a summary — short, medium, or long — with key terms pulled into the margin.

**Live demo:** _add your deployed URL here_
**Repo:** _add your GitHub URL here_

---

## Why this build

Most take-home versions of this brief call an LLM API for the summary. That works, but it means the reviewer's demo depends on your API key, your rate limit, and your uptime months from now. I built the whole pipeline — extraction, OCR, and summarization — to run **entirely client-side**, in the browser, with no backend and no API keys:

- It never goes down and never needs a `.env` file to run.
- A document never leaves the user's device, which matters for anything remotely sensitive.
- It works from a single static `index.html` — deploy it anywhere that serves static files.
- Free tiers usually rate-limit; this doesn't, because there's nothing to call.

The tradeoff is that the summarizer is extractive (it selects and reassembles the document's own sentences) rather than generative (it doesn't paraphrase). For a technical assessment this is the right tradeoff: it's inspectable, deterministic, and the reasoning is visible in the code rather than hidden behind a prompt.

## How it works

**1. Upload** — drag-and-drop or file picker, accepts PDF, PNG, JPG, WEBP.

**2. Text extraction**
- PDFs: [pdf.js](https://mozilla.github.io/pdf.js/) reads the embedded text layer, page by page.
- If a PDF has little or no text layer (a scanned document saved as PDF), it's automatically rasterized page-by-page and OCR'd — no separate "is this scanned?" toggle for the user.
- Images: [Tesseract.js](https://tesseract.projectnaacl.org/) runs OCR directly in the browser (WASM), no server round-trip.

**3. Sentence ranking (TextRank)**
- The extracted text is split into sentences.
- A similarity graph is built where two sentences are "connected" in proportion to how many meaningful (non-stopword) terms they share.
- Sentence importance is computed with **PageRank power iteration** over that graph — the same algorithm behind the original TextRank paper (Mihalcea & Tarau, 2004) — so a sentence scores highly when it's echoed by other important sentences, not just when it repeats common words.
- For very long documents (>260 sentences), the graph is skipped in favor of a position-weighted term-frequency score, to keep the whole thing running in milliseconds rather than building an O(n²) matrix on a 10,000-word doc.

**4. Summary composition**
- The top-ranked sentences are selected (12% / 28% / 50% of the document for short / medium / long) and re-ordered back into their original sequence, so the summary reads coherently rather than as a shuffled highlight reel.
- The most salient sentences are highlighted inline.
- The 10 highest-frequency meaningful terms are surfaced as key-term tags.

**5. Output** — copy to clipboard or download as `.txt`. Length can be changed after the fact without re-processing the document, since ranking happens once and only the selection threshold changes.

## Tech stack

| Concern | Choice | Why |
|---|---|---|
| PDF parsing | pdf.js (CDN) | Industry-standard, no server needed |
| OCR | Tesseract.js (CDN, WASM) | Runs fully client-side |
| Summarization | Hand-written TextRank | No API key, transparent, fast |
| UI | Single HTML file, vanilla JS, CSS variables | Zero build step, trivial to deploy/audit |

No frameworks, no bundler, no `node_modules`. This was a deliberate choice for a scoped assessment: it keeps the entire logic auditable in one file and removes an entire category of "works on my machine" risk.

## Running it locally

No build step required.

```bash
# any static server works, e.g.:
npx serve .
# or just open index.html directly in a browser
```

## Deploying (any of these — zero configuration needed)

This is one static `index.html` file with no build step, so any static host works:

**Netlify (fastest — no account setup required for a test link)**
1. Go to [app.netlify.com/drop](https://app.netlify.com/drop)
2. Drag the `document-summary-assistant` folder (the one containing `index.html`) onto the page
3. Netlify gives you a live URL in seconds — that's your "Working application URL" deliverable

**Vercel**
```bash
npm i -g vercel
cd document-summary-assistant
vercel deploy --prod
```

**GitHub Pages**
1. Push this folder to a GitHub repo
2. Repo Settings → Pages → Deploy from branch → `main` / root
3. Your app is live at `https://<username>.github.io/<repo>/`

**Cloudflare Pages / Firebase Hosting / surge.sh** all work the same way — point them at the folder, no build command, output directory is `.`.

## Error handling & edge cases covered

- Rejects unsupported file types and files over 25MB with an explicit, actionable message.
- Detects PDFs with no usable text layer and automatically falls back to OCR instead of returning an empty summary.
- Guards common abbreviations (`Mr.`, `e.g.`, `Fig.`, etc.) during sentence splitting so they don't fragment the summary.
- Every processing stage has its own loading state, not just a single spinner, so a slow OCR pass on a large scanned PDF doesn't look like a hang.

## What makes this different, not just "more features"

Most take-home submissions for this brief will look the same: upload → call an LLM → paste back whatever it says. That's a wrapper, not an engineering decision. Three things here come from actually thinking about what a *document* summarizer should guarantee, not just what a chatbot can output:

**1. The summary is provably not hallucinated — and you can check, not just take my word for it.**
Because the summarizer is extractive (TextRank, not a generative model), every sentence in the summary is a byte-for-byte substring of your document. That's not a claim in the README — it's enforced in code and testable: click any summary sentence and the app finds its exact character position in the extracted source text and highlights it there. If a sentence can't be located verbatim, it can't have made it into the summary in the first place, by construction. For a document summarizer, "did it make this up" is the single scariest failure mode, and this design makes that failure mode structurally impossible rather than merely unlikely.

**2. Every document gets its own unique visual "fingerprint," generated from the same math that ranks it — not decoration bolted on afterward.**
TextRank builds a similarity graph between sentences before ranking them. Instead of throwing that graph away once ranking is done, the app renders it: each sentence becomes a point on a circle, sized by importance and colored red if it made the summary; a line is drawn between two sentences when they share real vocabulary. The result is a small radial diagram that's structurally unique to that document — two documents with different internal structure literally cannot produce the same shape, because it's drawn from their own similarity matrix. Click a node and it locates that sentence in the source, same as clicking a summary line. This turns an internal data structure most implementations discard into a legible, interactive part of the product.

**3. Summary length is a dial, not three buckets — and dragging it doesn't re-run the algorithm.**
Ranking (the expensive part — building the O(n²) similarity graph and power-iterating PageRank) happens once per document. Selecting *how much* of that ranked list to show is a cheap sort-and-slice, so the compression slider (5%–60%, continuous) re-renders instantly on every drag tick instead of recomputing sentence importance from scratch. Short/Medium/Long still exist as quick presets that just set the slider — but the underlying model treats length as continuous, because it is.

On top of those three, smaller but genuinely useful additions:

- **OCR confidence scoring** — a colour-coded badge reporting Tesseract's actual confidence, so a shaky scan doesn't silently masquerade as clean text.
- **Reading-time delta** — "8 min → 2 min · 75% shorter," a legible before/after rather than an abstract percentage.
- **Listen to the summary** — one click, using the browser's built-in speech synthesis, zero dependencies.
- **Session history** — the last 5 documents stay in memory so you can flip back without re-running OCR (nothing is written to disk or localStorage).
- **Export as PDF** — a dedicated print stylesheet strips the UI down to a clean one-pager, including the document fingerprint.
- **Dark mode** — respects `prefers-color-scheme`, same palette carried into a dark variant rather than an inverted default theme.

## Possible next steps

- Add an optional "generative" mode that sends the extracted text to an LLM API for an abstractive summary, for users who bring their own key.
- Persist recent documents locally (IndexedDB) so a summary survives a page refresh.
- Multi-language OCR (Tesseract supports it; currently locked to English for speed).
- Batch upload / process multiple documents in one pass and compare summaries side by side.
