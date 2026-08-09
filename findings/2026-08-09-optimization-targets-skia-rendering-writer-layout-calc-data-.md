# Optimization targets: Skia rendering, Writer layout, Calc data, startup

_Generated 2026-08-09T21:11:32_

# LibreOffice Optimization Analysis (HEAD `88771d0a6`)

Static, code-level analysis of four areas: `vcl/skia` + `vcl/source/gdi` (rendering), `sw/source/core/text` (Writer layout), `sc/source/core/data` (Calc), `desktop/source/app` + `configmgr` (startup). No build performed (no build dir; LibreOffice takes hours to build). All claims are grounded in paths + line numbers at HEAD.

---

## F1 — Writer: constant-string text measurements re-shape on every call (no layout-context cache)

**Files:** `sw/source/core/text/frmform.cxx`, `txttab.cxx`, `porfly.cxx`, `porfld.cxx`, `inftxt.hxx/cxx`, `sw/source/core/txtnode/swfont.cxx`, `fntcache.cxx`, `vcl/source/gdi/impglyphitem.cxx`

The layout hot path measures constant strings repeatedly, and each measurement rebuilds a full text layout:
- `frmform.cxx:1695` `rInf.GetTextSize("          ")` (10 spaces), `:1716` `GetTextSize(sLastWord)`, `:1725` `GetTextSize("-")` — in `SwTextFrame::ParagraphComposer`, per candidate line.
- `txttab.cxx:631` and `:648` `GetTextSize(OUString(' '))` / `GetTextSize(OUString(m_cFill))` — per tab stop.
- `porfly.cxx:93`, `porfld.cxx:132` — single-space measurement per fly/field portion.

Chain: `SwTextSizeInfo::GetTextSize(const OUString&)` (`inftxt.hxx:805`) → `inftxt.cxx:415` constructs a `SwDrawTextInfo` with **`pSI = nullptr` and layout context `std::nullopt`** → `SwSubFont::GetTextSize_` (`swfont.cxx:1055`) → `SwFntObj::GetTextSize` (`fntcache.cxx:1737`) → `OutputDevice::GetTextWidth` → `vcl/source/gdi/impglyphitem.cxx:444-449`: because `layoutCache == nullptr`, it allocates a fresh `vcl::text::TextLayoutCache::Create(text)` on **every call** — i.e. script itemization + shaping, per line, per tab stop.

Space width is constant within a paragraph (same font), and the "10 spaces" figure is identical for every line of the paragraph. The codebase already has a context-threading overload (`inftxt.cxx:431-450`, `GetTextSize(std::optional<SwLinePortionLayoutContext>)`) that reuses the paragraph layout context — the `const OUString&` overload bypasses it.

**Fix:** memoize space/char width per `(SwFont, OutputDevice)` or per paragraph; or add a layout-context-aware overload for these call sites. Saves a `TextLayoutCache::Create` + shaping allocation per line/tab stop.

---

## F2 — Calc: redundant per-row mdds block binary search in cell access and optimal-width

**Files:** `sc/source/core/data/column.cxx`, `column2.cxx`

- `ScColumn::GetCellValue(SCROW)` (`column.cxx:570-577`) and `GetCellTextAttr(SCROW)` (`column.cxx:620-639`) call `maCells.position(nRow)` from scratch — an O(log #blocks) binary search — every time, throwing away positional state even in tight per-row loops.
- `GetOptimalColWidth` (`column2.cxx:736-864`): inner loop at `:833-850` calls `GetNeededSize(nRow, …)` per non-empty cell; `GetNeededSize` (`column2.cxx:86-99`) re-runs `maCells.position(nRow)` even though the loop at `column2.cxx:825` already positioned the iterator via `maCells.position(itPos, nRow)` (amortized O(1)).
- Net effect for a full column (e.g. 1M rows, many blocks): N × O(log B) searches plus per-row pattern/script work, where the block-iteration style already used by the simple-text path (`column2.cxx:800`, `sc::ParseAllNonEmpty`) shows the cheaper pattern.

**Fix:** add iterator/offset-aware overloads of `GetNeededSize`/`GetCellValue` — the `ColumnBlockPosition` API (`sc/inc/mtvelements.hxx:169-177`) exists precisely for this and is already threaded through the copy/paste and listener paths.

---

## F3 — Calc: mutex + two-level hash lookup on every `ColumnBlockPositionSet::getBlockPosition` call

**Files:** `sc/source/core/data/mtvelements.cxx` (+ `sc/inc/mtvelements.hxx`)

`ColumnBlockPositionSet::getBlockPosition` (`mtvelements.cxx:77-113`) takes `std::scoped_lock aGuard(maMtxTables)` and then does **two** `std::unordered_map` lookups (tab → col) on **every** call — including hits, where the entry was already fetched. It is called per formula reference / per cell in hot paths:
- `column2.cxx:3527` `ScColumn::StartListening`, `:3539` `EndListening`
- `column4.cxx:102,251`, `column3.cxx:1536` (delete/copy-before-clip)
- `column.cxx:883,1574` (copy/paste handlers), `documentstreamaccess.cxx:45,66,101`
- `refupdatecontext.cxx:94-96`, `clipcontext.cxx:32`

During start-listening of a large document every formula reference pays a mutex acquisition plus two hash finds. The single-table variant `TableColumnBlockPositionSet` (`mtvelements.cxx:149-172`) is lock-free — the lock is only needed for insertion, not for the read-mostly hit path.

**Fix:** double-checked lookup (read without lock, lock only on insert), or per-table sets / `std::shared_mutex` read locks.

---

## F4 — Skia: per-call allocations in bitmap readback and N50 selection inversion

**File:** `vcl/skia/gdiimpl.cxx`

- `getBitmap` (`:1374-1415`): `makeCheckedImageSnapshot(mSurface, rect)` may copy the region — the in-source TODO at `:1383-1385` admits the waste when the image is only consumed by `blendAlphaBitmap()`. On HiDPI it then immediately `Scale(1.0/mScaling, …)` (`:1393-1412`), i.e. downscale now, upscale again later.
- `invert` with `SalInvert::N50` (`:1466-1486`): allocates a fresh 2×2 `SkBitmap`, writes 4 pixels, and builds a shader on **every** call (in-source TODO at `:1469` “Use createSkSurface() and cache the image”). N50 is selection inversion — runs on every selection repaint in Writer/Calc.
- `getPixel` (`:1417-1430`): allocates a **full-surface** `SkBitmap` per pixel read (comment admits “presumably slow”).

**Fix:** statically cache the immutable 2×2 checker image + shader (or reuse `createSkSurface()`); read a 1×1 rect in `getPixel`; let the alpha-blend path use the rect snapshot directly instead of a full copy.

---

## F5 — Startup: configmgr parses every registry layer strictly sequentially on the critical path

**Files:** `configmgr/source/components.cxx`, `nodemap.cxx/hxx`, `config_map.hxx`, `desktop/source/app/appinit.cxx`

- `parseXcsXcuLayer` (`components.cxx:825-829`) = `parseXcdFiles` + **all** `.xcs` + **all** `.xcu` files, parsed one file at a time; each file gets a brand-new `ParseManager(url, new XcsParser/XcuParser)` (`components.cxx:97-120`) with its own `xmlreader::XmlReader` — full re-parse per file, single-threaded.
- Runs from the `Components` constructor (`components.cxx:488-556`, instrumented with `osl_getGlobalTimer()` — the timing logs show this is a tracked startup cost), which executes inside `cppu::defaultBootstrap_InitialComponentContext()` called by `Desktop::InitApplicationServiceManager` (`desktop/source/app/appinit.cxx:83-84`) — the first thing `soffice_main` does (`sofficemain.cxx:66,105`).
- `NodeMap::findNode` (`nodemap.cxx:43-49`) has a **single-entry** cache (`nodemap.hxx:60-63`) that is cleared by every `insert`/`operator[]` (`nodemap.hxx:52-55`). During the parse phase each node is inserted once, so the cache never hits; at read time every tree walk does `std::map` `OUString` comparisons (`config_map.hxx`, `LengthContentsCompare`).

**Fix:** parallelize per-file parsing within a layer (the files are independent; the tree merge is the only serial part) and/or cache the parsed configuration in a binary form; give `NodeMap` a small multi-entry cache that is not invalidated by inserts of unrelated keys.

---

## Ranking / expected impact

| # | Area | Cost today | Effort | Payoff |
|---|------|-----------|--------|--------|
| F5 | Startup (configmgr) | hundreds of XML files re-parsed serially on every launch | medium (parallelize/cache) | high — visible startup latency |
| F1 | Writer layout | layout cache rebuild per constant-string measure per line | low (memoize/thread context) | high — text-heavy docs |
| F4 | Skia | shader/bitmap alloc per invert(N50) call; full-copy snapshots | low | medium — selection/scroll repaint |
| F3 | Calc | mutex + 2 hash finds per formula ref/cell op | low (double-checked lookup) | medium — large-sheet load/paste |
| F2 | Calc | N×O(log B) block searches in width calc & cell access | medium (iterator overloads) | medium — big-sheet resize/filter |

*Environment note: this checkout is a single squashed commit (`88771d0a6`), so no per-line git history was available; findings are grounded in current HEAD paths/line numbers, and cross-referenced against the in-source TODO comments that independently corroborate F1/F4.*
