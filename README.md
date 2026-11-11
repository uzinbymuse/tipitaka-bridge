# Tipitaka Bridge — တိပိဋက တံတား

A tiny static site that connects the two Tipitaka readers, page by page:

- **Text reader** — https://tipitaka-web-three.vercel.app (192 Pali books, Flask + Turso)
- **Image reader** — https://tipitakapali-zin.vercel.app (117 scanned print books, static HTML + Cloudflare R2)

## Routes

- `/to-image?book=<text slug>&page=<n>` — from a text page to the same page's scanned image.
  Builds the canonical viewer URL (`viewer.html?book=&total=&seg=&img=&title=&home=`, `#pdf=`).
- `/to-text?book=<image id>&page=<n>[&pdf=<pdf>]` — from a scanned printed page to the same page's text.
  Redirects to the text reader with `?book=<slug>&page=<n>` (deep-link support in the text app).
  The optional `&pdf=` disambiguates volumes where printed numbers restart
  between two works (see below); the viewer link should pass it.

Both routes show a short Burmese message instead of redirecting when no
counterpart exists (e.g. grammar books have no scanned version).

## Files

- `mapping.json` — the single source of truth: 148 rows, one per
  (image book, text book, printed-page range). `text_page = image_printed + offset`.
  All offsets are currently 0 (both corpora share the same edition pagination).
  Fields: `image_id`, `image_title`, `text_id`, `text_title`, `offset`,
  `image_printed_first`, `image_printed_last`, `shared_toc`, `confidence`
  (`high`/`medium`/`low`), `method`, `note`.
- `image-books.json` — baked copy of the image books' data (title, total PDF
  pages, print segments) from the live R2 `books.json`, 2026-11-11.
  `to-image` tries the live R2 `books.json` first and falls back to this copy.
- `to-image/index.html`, `to-text/index.html` — the two redirectors (plain JS, no build).
- `index.html` — landing page.

## How the mapping was built (2026-11-11)

Title-first, table-of-contents-confirmed matching between the image books'
`toc.json` and the text app's `/api/book` data:

1. Candidate text books by normalized book title (volume markers stripped;
   volume splits disambiguated by volume number, e.g. ပဌမော ↔ (ပ)).
2. Combined print volumes segmented into constituents (shared-suffix tiling
   and full-title substring rules).
3. Each candidate confirmed by monotonic in-order alignment of TOC titles
   (numbers kept — `၁-` vs `၁။` and spelling variants `ဋ္ဌကထာ`/`အဋ္ဌကထာ`,
   `နိဒေသ`/`နိဒ္ဒေသ`, `ဓမ္ဓ`/`ဓမ္မ`, `သံယုတ်တ`/`သံယုတ္တ` unified);
   the winning offset needs ≥5 shared titles and at least half of the
   aligned pairs (a real cross-match attha_sa_03 once passed 26 vs 13, so
   the share rule replaced an earlier "2× second-best" rule).
4. Competing candidates claiming the same printed pages were rejected
   (the stronger TOC match wins).

Coverage: 117/117 image books, 148 text books of 192 (the other 60 text
books — mostly grammar and miscellaneous works — have no scanned version).
Confidence: 140 high / 3 medium / 5 low.

## Known caveats

- **Low-confidence rows (5):** `A1MMGB001849` (Kankhavitarani commentary —
  the text corpus covers printed 83–357 only, no TOC to confirm the offset),
  `A1MMGB003718` (Purana/Abhinava Tika — 2 rows, same reason), and
  `A1MMGB002673` (2 rows, see below). Offsets assumed 0.
- **Medium-confidence rows (3):** `A1MMGB002243` and `A1MMGB002248`
  (Jataka volumes 6–7, 6 shared TOC titles each) and `A1MMGB011810` (below).
- **`A1MMGB021245`:** the title names both Dhammasangani Tikas, but the
  217-page volume holds only the Mula-Tika (printed 1–203); the Anu-Tika
  rows were dropped as spurious cross-matches (the two Tikas share TOC titles).
- **`A1MMGB011810` (Mahaniddesa commentary):** medium confidence — only 2 TOC
  pairs confirm offset 0; the tail beyond printed 386 is sparse in the TOC.
- **`A1MMGB002673` (Vibhanga Mula/Anu-Tika):** printed numbers restart
  between the two Tikas (both start at printed 1), so the two rows overlap
  by design and carry `pdf_first`/`pdf_last` ranges; `/to-text` uses `&pdf=`
  to pick the right one. Same pattern may apply to other restart volumes —
  only this one needed it because its text books have no TOC to confirm with.
- **Print segments:** the viewer URLs use the R2 `books.json` segments live
  (baked copy as fallback). The 2026-11-07 audit found segment errors in 42
  books (`PROPOSED-FIXES-42.md`); those fixes are NOT applied here yet —
  reconciling them against the live R2 segments is still open work, and the
  bridge will pick the corrected segments up automatically once R2 is fixed.
- The text app's `?book=&page=` deep-link support is a separate change in
  the text app repo; `/to-text` links to that URL shape already.

## Deploy

Static site on Vercel, Git-connected to this repo. No build step, no
`vercel.json` (the `to-image/` and `to-text/` directories serve their own
`index.html`; Vercel's directory redirect handles `/to-image` → `/to-image/`).
