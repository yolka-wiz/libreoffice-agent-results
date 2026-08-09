# LibreOffice optimization targets: VCL text layout, Writer formatting, Calc column sizing, startup/configmgr

_Generated 2026-08-09T21:24:31_

# LibreOffice Code-Level Optimization Targets

Inspected actual code in `vcl/source/gdi/CommonSalLayout.cxx`, `vcl/source/outdev/font.cxx`, `vcl/source/gdi/sallayout.cxx`, `sw/source/core/text/{itrtxt,itrpaint,itrform2,porlay,portxt,itradj}.cxx`, `sc/source/core/data/{column,column2,column3}.cxx`, `configmgr/source/*`, and `desktop/source/app/*`. All line numbers verified against the working tree (HEAD 88771d0a6).

---

## 1. [vcl] Quadratic kashida-position validation — `IsKashidaPosValid` does 2 linear scans per query

**Files/lines:**
- `vcl/source/gdi/CommonSalLayout.cxx:1113-1133` — `GenericSalLayout::IsKashidaPosValid` runs two `std::find_if` scans over the whole `m_GlyphItems` vector (lines 1116-1119) for every `(nCharPos, nNextCharPos)` pair.
- `vcl/source/outdev/font.cxx:1336-1348` — `OutputDevice::GetWordKashidaPositions` calls it inside a per-character loop (`for i in 0..nEnd`), so one word = **O(n²)** glyph scans. `pOutMap->at(i)` (line 1347) is also bounds-checked in a loop whose size is already known.
- `vcl/source/gdi/sallayout.cxx:1304-1326` — `MultiSalLayout::IsKashidaPosValid` may re-run the same scan on every fallback layout.
- Caller: `sw/source/core/text/itradj.cxx:199` (`lcl_ComputeKashidaPositions`, per word per line during Arabic/justified block layout).

**Reasoning:** For a word of length *n* with *g* glyphs, every query is O(g); *n* queries → O(n·g). Long Arabic words with many fallback runs make this visibly slow during justification. The glyph list is only mutated by `LayoutText`/`ApplyJustificationData`, so a one-time `charPos → glyph index` lookup (single pass, e.g. `std::unordered_map<sal_Int32,size_t>` or a sorted index + binary search) turns each query into O(1)/O(log g). Also switch `pOutMap->at(i)` → `operator[]` (or resize + assignment).

---

## 2. [sw] `SwScriptInfo::ScriptType` / `NextScriptChg` are linear scans called per text portion

**Files/lines:**
- `sw/source/core/text/porlay.cxx:1617-1628` — `ScriptType(nPos)` loops from index 0 over all script-change entries on **every** call.
- `sw/source/core/text/porlay.cxx:1604-1614` — `NextScriptChg(nPos)` is the same linear scan.
- Hot callers in the line-formatting loop `SwTextFormatter::BuildPortions` (`sw/source/core/text/itrform2.cxx`): lines 491-492 (`fnRequireKerningAtPosition` → `ScriptType` ×2), line 714 (`NextScriptChg`), plus `sw/source/core/text/portxt.cxx:82,169` (justification `lcl_AddSpace`, called per text portion when block align is on) and `sw/source/core/text/redlnitr.cxx:695`.

**Reasoning:** `m_ScriptChanges` (declared `sw/source/core/inc/scriptinfo.hxx:120-130`) is sorted by position, but both queries rescan from the beginning. A paragraph with *S* script boundaries and *P* portions costs O(P·S) instead of O(P·log S) — quadratic for mixed CJK/Latin/Complex paragraphs. Fix: `std::lower_bound` over the sorted vector, or — better — an incremental cursor, since `BuildPortions`/`lcl_AddSpace` query positions monotonically. Same treatment for `NextDirChg`/`DirType` (porlay.cxx:1630-1655).

---

## 3. [sw] `SwTextIter::GetPrev_` re-walks the line list from the paragraph root on every backward step

**Files/lines:**
- `sw/source/core/text/itrtxt.cxx:79-90` — `GetPrev_` does `while (pLay->GetNext() != m_pCurr) pLay = pLay->GetNext();` starting from `GetParaPortion()`.
- `sw/source/core/text/itrtxt.cxx:161-185` — `GetPrevLine` repeats the same root-walk (plus dummy skipping).
- Consumers: `Prev()` (99-115), `CharToLine` (210-216), cursor/repaint code.

**Reasoning:** `SwLineLayout` is a singly-linked list with no back-pointer, so each `Prev()` is O(lines in paragraph) and scanning a paragraph backwards is O(lines²). Bounded by lines-per-paragraph, but it is pure pointer-chasing with an easy fix: cache the previous line in the iterator (`m_pPrev` is already a member — it is only filled by `GetPrev_`), or add a `GetPrevious()` back-pointer maintained by `Insert`/`TruncLines` (itrform2.cxx:132-142).

---

## 4. [sc] `GetOptimalColWidth` inner loop triggers a full string-format + i18n script scan per cell

**Files/lines:**
- `sc/source/core/data/column2.cxx:833-850` — per-cell loop: `rDocument.GetScriptType(nCol, nRow, nTab)` (line 835) → `GetPattern(nRow)` (839) → `GetNeededSize` (843).
- `sc/source/core/data/documen6.cxx:132-152` — `ScDocument::GetScriptType`: when the cached script type is `UNKNOWN` it fetches pattern + conditional format + number format, then `GetCellScriptType` formats the whole cell value (`ScCellFormat::GetString`) and runs the i18n script detector, caching afterwards.
- Same cost appears in `GetOptimalHeight` (`column2.cxx:1030-1032` → `GetRangeScriptType`) → `UpdateScriptType` (`column3.cxx:843-883`).

**Reasoning:** On the first optimal-width/height pass over a large sheet (load, layout, or after edits invalidate script types), every non-empty cell pays one full number-format + string conversion + script scan just to select a font (`bGetFont`). Consecutive cells in a block usually share pattern/format, so this can be batched: resolve the script type once per (block, pattern-change) instead of per cell, or precompute it at set-time as `column4.cxx:1134 updateScriptType` already does for edit cells.

---

## 5. [vcl] `LayoutText` hot loop: 2 glyph passes, per-subrun language conversion, per-glyph code-point decode

**File:** `vcl/source/gdi/CommonSalLayout.cxx` (the function every text render goes through, lines 360-856)
- Lines 673-686 vs 691-840: glyphs are walked twice (cluster out-of-order detection pass, then the main item-building pass) — could be merged into one pass.
- Line 618 (+ `ShapeSubRun` → `hb_shape_full` at 207): when `SalLayoutFlags::UnclusteredGlyphs` is set, HarfBuzz shapes the *same* text a second time (documented workaround for tdf#61444/71956/124116, but it doubles shaping cost for those layouts).
- Lines 589-599: `hb_language_from_string` re-converts the BCP47 string on every sub-run — hoist/cache the `hb_language_t` once per layout (`msLanguage` is already parsed in `ParseFeatures`).
- Line 783: `rArgs.mrStr.iterateCodePoints(&o3tl::temporary(sal_Int32(nCharPos)), 0)` — a full code-point decode call per glyph; for the common BMP case a direct `pStr[nCharPos]` read (with a surrogate check only when needed) avoids the call + temporary machinery on every glyph of every text run.

---

## Bonus (startup/configmgr)

- `configmgr/source/components.cxx:258-293` — `Components::initGlobalBroadcaster` walks *all* roots with a full path→`Modifications` map descent on every broadcast (in-code `TODO` at line 262: "Iterate only over roots w/ listeners"); called from `rootaccess.cxx:167` and `update.cxx:109-141` on every config commit, which happens repeatedly during startup (extension sync, settings writes).
- `configmgr/source/components.cxx:488-...` — all XCS/XCU layers are parsed strictly sequentially in the `Components` constructor; layers are independent after schema parsing and could be parsed in parallel (there is already timing instrumentation, e.g. line 532).

## Method
- Consulted shared memory first (no prior findings on these exact spots), then `rg_search`/`read_repo_file` on each target, verified line numbers, and recorded each finding via `memory_add` (kinds `opt`, ids #265-#270). No test/lint run performed (full LibreOffice build not required for static analysis; `detect_toolchain` shows a heavyweight build).

