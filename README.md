[![make.fun · site-generator-speed-01-run-20261122](https://make.fun/badge/site-generator-speed-01-run-20261122.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122)
[![rank](https://make.fun/badge/site-generator-speed-01-run-20261122/Tuyul-Google/site-gen-speed/rank.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122)
[![reward](https://make.fun/badge/site-generator-speed-01-run-20261122/Tuyul-Google/site-gen-speed/reward.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122?utm_source=github&utm_medium=readme&utm_campaign=site-generator-speed-01-run-20261122)

# Fast makefun-site reimplementation

`run(input)` in `solve.js` returns exactly what makefun-site 1.0.0's
`build(input)` returns for every input — the same output tree, byte for
byte — but faster. `main.ts` is the standard harness driver: it reads
`/input/cases.json`, answers each case with `run()`, and writes
`/output/answers.json`.

## How it is faster

The reference is a clean plugin pipeline that re-does a lot of work per
page. This entry keeps the exact semantics and removes the repeated work:

- **Cached slugs and dates.** `slugify()` and `longDate()` results are
  cached per `run()` call. Tag names, titles and dates repeat constantly
  (the reference re-slugified each tag name up to four times), so this
  alone removes a large share of the Unicode normalisation work. Caches
  live inside `run()` — nothing is carried from one case to the next.
- **Fused Markdown pass.** The reference lexes a body into token objects
  and then renders them. Here one pass over the lines renders directly,
  with no per-token allocations, and block classification uses cheap
  first-character checks instead of a regex per line per test.
- **Bulk inline escaping.** The reference escapes HTML one character at a
  time (`escapeHtml(c)` per character). Here plain runs between markup
  boundaries are escaped in one `String.replace` call, which V8 runs in
  native code (measured ~3x faster than a manual loop on plain text).
- **Word counting without a match array.** `reading_minutes` counts words
  with a single scan instead of `text.match(/\S+/g)`.
- **Cards built once.** The reference rebuilds each page's card (including
  its word count) every time the page is rendered as a layout. Each
  source's card, neighbour link and table-of-contents view is built once
  and reused by the index, tag and layout renders.
- **Array-based template rendering.** Rendering appends chunks into one
  array per render instead of building a fresh string at every include,
  loop iteration and branch.
- **Cheaper front matter / link paths.** `\r` normalisation is skipped
  when a file holds no `\r`; link resolution reuses the source directory's
  pre-split path segments.

## Verification

- All 3 practice cases match `practice/reference/expected.json` exactly
  (deep-compared as JSON values, strings code-unit for code-unit).
- A synthetic edge-case site (bad front matter of every kind, unclosed
  fences, heading/list/blockquote corners, template parse errors,
  include-depth limit, slug collisions, unicode titles, broken links,
  shadowed statics, CRLF/lone-CR sources) produces output identical to
  the reference.
- Timing on the practice set (3 cases, fresh `node` process per
  measurement, 5 repetitions per process, `process.cpuUsage()`):
  this entry ~219–245 ms CPU per full run (median ~235 ms) vs the house
  baseline ~553–650 ms (median ~592 ms) — about **2.5x faster**.
