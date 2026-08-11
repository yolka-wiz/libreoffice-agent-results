# opt-f1: Writer constant-string text measurement memoized (per-line layout-cache rebuild)

_Generated 2026-08-11T06:36:59_

## opt-f1 — Writer: constant-string text measurement rebuilds layout cache per line (memoize)

**Branch:** `writer/opt-f1-const-text-memoize` (pushed) — commit `6e5b774a9`
**Module:** sw (Writer core text)
**Status:** IMPLEMENTED + PUSHED

### Problem (verified root cause)

Writer measures constant strings — `" "` (space), `"-"` (hyphen), the tab fill char, `"          "` (10 spaces) — once **per line / per tab stop / per paint** during formatting:

- `frmform.cxx:1695/1716/1725` — `ParagraphComposer` (per line)
- `portxt.cxx:1100` — `FormatEOL` (every line ending in blanks)
- `txttab.cxx:631/648` — tab paint (`' '` and fill char)
- `txthyph.cxx:422` — soft-hyphen width
- `porfly.cxx:93`, `porfld.cxx:132`, `portox.cxx:54`, `porrst.cxx:1037`, `porref.cxx:52`, `portxt.cxx:1315` — portion view widths

All go through `SwTextSizeInfo::GetTextSize(const OUString&)` → `GetTextSize(OutputDevice*, pSI=nullptr, …)` (`inftxt.cxx:415`), which builds a `SwDrawTextInfo` with `pSI=nullptr`, no layout context and **no VCL text-layout cache**. On a glyph-cache miss this runs `TextLayoutCache::Create` + a full HarfBuzz shape of the constant string per call (`impglyphitem.cxx:444-452`), and pollutes the bounded VCL glyph LRU with constant-string entries.

The width of a space/hyphen/fill string is constant per (physical font, output device, map mode, layout state).

### Fix (smallest correct change)

1-entry memo in `SwTextSizeInfo` for the constant-string path, keyed by everything that can change the result:
- physical font identity — new `SwFont::GetFntCacheId()/GetFntIndex()` accessors returning the font-cache id/index of the active sub-font (every `SwSubFont` attribute setter resets the id to `nullptr`, so any font change forces a miss — verified `swfont.hxx:485/498/511/553/566`),
- output device + reference device, both map modes,
- layout mode, RTL flag, `SnapToGrid()` (ruby formatting toggles it via `pormulti.cxx`),
- the string itself.

The cache lives in the per-format/per-paint `SwTextSizeInfo` instance, so zoom/layout-mode changes across operations start from a fresh entry. The key is stored **after** the measurement (the font-cache id is only valid once `ChgFnt` inside `GetTextSize_` has resolved it). Font-cache eviction is harmless: the width depends on the font attributes (captured by the id), not the `SwFntObj` instance.

**Files:** `sw/source/core/inc/swfont.hxx` (2 accessors), `sw/source/core/text/inftxt.hxx` (mutable cache key + `#include <vcl/mapmod.hxx>`), `sw/source/core/text/inftxt.cxx` (memo check/store). 78 insertions, 1 deletion.

### Verification

- **V1 (static review, manual):** every new expression type-checked by inspection (`GetFntCacheId/GetFntIndex` const accessors in a friend class, `MapMode` by value + `operator==` (cow-wrapper impl-pointer compare — exactly "did the device map mode change"), `static_cast<sal_uInt8>` of `ComplexTextLayoutFlags`, `mutable` members in a const method, default member initializers so copy ctors reset the cache). Cache-key soundness traced through `SwFntAccess`/`ChgFnt`/font-cache-id invalidation; all 13 call sites confirmed to pass `pSI==nullptr` + full-string.
- **V0 (build):** no build environment in this workspace — `clang-format`/compile were **not** run. This is stated honestly; the change is not compile-verified.
- Suggested follow-up (V2/V3): `make check` of sw layout suites (`layoutwriter`, `core_layout`) and a manual scroll/repaint check on a large paragraph-heavy document; optional perf trace confirming `GetLayoutGlyphs` no longer shapes `" "` per line.
