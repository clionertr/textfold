# A 423-page book's page: where the time and the memory go

Measured on one real page — a 423-page Chinese book, 342 sections, 1,961 paragraphs, 2,289 KaTeX formulas,
114 tables, 96 figures, a document 426,019 px tall — in Chrome 155, over the DevTools Protocol, at a fixed level
and with the browser's `immutable` cache bypassed.

| | before | after |
|---|---:|---:|
| DOM elements | 188,060 | 136,270 |
| … with the virtualization switch on | | **3,917** |
| MathML nodes (KaTeX's copy for screen readers) | 38,905 | 0 |
| Layout | 859 ms | ~550–625 ms |
| Script | 852 ms | ~730 ms |
| Worst long task at load | 1,452 ms | ~1,100 ms (957 ms in one paired run) |
| Page bytes (HTML + inline data) | 7,217,853 | **5,522,999** |
| Scroll height | 426,019 px | 426,019 px (unchanged) |

Timings carry ±10% run-to-run noise on this machine; the node counts and byte counts do not.

## What actually costs

A CPU profile of the load, not a guess:

```
 6.5%   904 ms  getBoundingClientRect     ← 49,407 geometry reads to find the dozen units on screen
 3.9%   552 ms  (program)
 0.5%    74 ms  set innerHTML
```

Three functions — `unitsNear`, `visibleCands`, `captureHere` — walked all ~1,961 units and measured each one, every
time they asked "what is near this band". Sections are 342 boxes, so measuring those first and looking inside the ones a
band can touch costs an order of magnitude less.

The second cost was KaTeX: for every formula it emits a full HTML tree *and* a hidden MathML tree for screen readers.
On this book that is 31,287 MathML nodes of pure duplication, plus spans that carry only the twelve classes KaTeX uses
while it builds its box tree (`mord`, `mbin`, `mrel`, … — none of them referenced by `katex.min.css`; see
[KaTeX#3344](https://github.com/KaTeX/KaTeX/issues/3344) and [KaTeX#4164](https://github.com/KaTeX/KaTeX/pull/4164)).

The third cost was the data itself: 6.4 MB of inline JSON, 290,081 tokens of which 282,478 (97.4%) were objects with a
single field, `{"t": "字"}` — one per Chinese character.

## The changes

1. **KaTeX draws HTML only** (`output: 'html'`). Nothing on screen changes; a screen reader loses the formula's MathML.
2. **`katexSlim()`** strips the twelve build-time-only classes (`nobreak` among them), unwraps spans that carry nothing but those classes and
   hold only text, and drops the empty ones. Done on `renderToString`'s string, innermost first, eight passes.
   117,412 spans instead of 130,302 on this book.
3. **`<img decoding="async">`** — decode off the main thread.
   (`loading="lazy"` was tried and rejected: the figures carry no width/height, so the page's height shrank by 40,000 px
   as pictures failed to reserve space. Correctness of the scrollbar beats the decode saving.)
4. **Bands measured by section** — `secsNear()` + `unitsIn()` before any per-unit `getBoundingClientRect()`.
5. **Packed tokens** (`server.py: pack_tokens`, `?pack=1`) — a token that carries nothing but its text is sent as that
   text; the client reads both forms (`tTok()`). Page: −23.5% bytes, and fewer objects to parse.
6. **Virtualization** (a switch, `Aa › Performance`) — a section four screens away hands its children to a detached
   `DocumentFragment` and keeps its measured height inline; it comes back within 2.2 screens. A section whose stored
   level or language no longer matches the reader's is mounted, measured and released again, so heights follow a level
   change. Opening the browser's find bar or making a selection restores everything first: detached text is out of the
   browser's reach until it is back.

## The switches

`localStorage['tf-perf'] = {"pack": 1, "virt": 1}` — both off by default; both in `Aa › Performance`.

- **pack** needs a reload: the data is inside the page's HTML. The switch sets the flag and reloads with `?pack=1`.
- **virt** applies at once.

## Tried and rejected

- **`content-visibility: auto`** with `contain-intrinsic-size`. Halved the worst long task (1,152 → 612 ms) but the
  scroll height collapsed from 475,626 to 206,356 px with a constant estimate, and to 221,353 px with a per-section
  estimate computed from the DOM at boot — because the reader renders progressively and only a third of the text was
  there yet. A model-based estimate would need a per-section height cache keyed by level, language, column width, font
  and KaTeX version; worth doing, but it is not a CSS one-liner.
- **`loading="lazy"`** — see above.
- **`warm()` on `requestIdleCallback` instead of a timer** — measurably worse (557 → 653/711 ms worst long task): the
  warmed keyframes are what make the first zoom instant, and idle time is where the reader is most likely to be touched.

## What I would do next, in this order

1. **Cache real section heights** (level + language + column width + font + KaTeX version in the key), then turn on
   `content-visibility: auto` with those heights. It keeps the DOM but skips style, layout and paint for what is far
   away — the only way to get the win *and* an honest scrollbar.
2. **Pack the ladder too.** `lad.brief` and `lad.topic` are still arrays of one-character objects with a marker index;
   runs of the same marker could be a string plus a marker, and `units` would shrink again.
3. **Slim the token model itself.** 370,000 characters, one object each, is a lot of objects for a reader that mostly
   concatenates them. A test over this book would say whether a compact format pays for the format version bump.

## Limits of the two switches (read before turning them on)

The packed page is a lossless change of representation — the same text, fewer objects — and needs nothing from the
reader beyond a tolerance for a token that is a string. Virtualization is the one with edges:

- **The browser's own find cannot see a released section.** The app's find bar and a text selection put everything back
  first (`beforeprint` too, and a link, a note or the saved position brings back the section it names), but Ctrl+F from
  the menu, or the mobile find UI, may miss text that is currently far away. Ctrl+A also misses it.
- **A released section is out of the document**, so a live `Range` whose endpoints sat inside one does not survive the
  release. Selections pause the sweep; a drag that starts in a released section is not possible until it is back.
- **Heights are only as good as the layout they were measured in.** The record is keyed by language, viewport and the
  reader's own `capEpoch` (fonts loaded, pictures arrived, type changed) plus the level; when any of them moves, the
  page is drawn afresh and the far sections are released again — a full render on a level or type change, exactly as
  without the switch.
- **Printing** is covered by `beforeprint`; an export that copies the live DOM rather than the document's data would not
  be — the app's own save and export build from the data, so they are unaffected.

If any of that matters more than the memory, leave the switch off: the four changes above it stand on their own.
