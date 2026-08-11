# sw: memoize constant-string text measurement (space/hyphen width) — opt-f1

_Generated 2026-08-11T06:14:03_

# Investigation: Writer constant-string text measurement rebuilds layout cache per line (memoize)

**Module:** sw (primary), touches vcl only as read-only context. **Type:** performance optimization (opt-f1).
**Upstream check:** GREEN — no bug id provided; web/Gerrit/Bugzilla searches found no matching upstream change (APIs bot-blocked, no related patch surfaced).

---

## 1. Root cause (verified code path)

All of these call sites measure a **constant string** (space, hyphen, fill char, 10 spaces) and get the width via `SwTextSizeInfo::GetTextSize(const OUString&)`:

| Call site | String | Frequency |
|---|---|---|
| `sw/source/core/text/frmform.cxx:1695` | `"          "` (10 spaces) | per line — `ParagraphComposer` invoked per line at `frmform.cxx:2011-2012` |
| `frmform.cxx:1716` | last word | per line |
| `frmform.cxx:1725` | `"-"` | per line |
| `sw/source/core/text/portxt.cxx:1100` | `' '` | `FormatEOL` — every line ending with blanks |
| `sw/source/core/text/porfly.cxx:93` | `' '` | trailing blank of fly portion, per line |
| `sw/source/core/text/txthyph.cxx:422` | `'-'` | per soft-hyphen view width |
| `sw/source/core/text/txttab.cxx:631` / `:648` | `' '` / fill char | every tab-stop paint |
| `porfld.cxx:132`, `portox.cxx:54`, `porrst.cxx:1037`, `porref.cxx:52`, `portxt.cxx:1315` | `' '` | field/outline/ref view widths, per paint |

**Path:** `inftxt.hxx:805-808` (inline) → `GetTextSize(m_pOut, nullptr, rText, 0, len)` `inftxt.cxx:415-428` builds a `SwDrawTextInfo` with
- `pSI = nullptr`, `layoutContext = std::nullopt` (inftxt.cxx:421-422),
- **no `SetVclCache(...)`** — the paragraph `TextLayoutCache` (`m_pCachedVclData`, set for format at `inftxt.cxx:1807`) is *not* threaded into these standalone-string measurements.

→ `SwSubFont::GetTextSize_` (`sw/source/core/txtnode/swfont.cxx:1022` → `:1055`) → `SwFntObj::GetTextSize` (`fntcache.cxx:1737` → `:1843`) → `GetTextArray` (`fntcache.cxx:861` → `:795` → `:818`):

```cpp
// fntcache.cxx:816-821  (no layout-context branch)
const SalLayoutGlyphs* pLayoutCache = SalLayoutGlyphsCache::self()->GetLayoutGlyphs(
    &rDevice, rStr, nIndex, nLen, 0, layoutCache);   // layoutCache == nullptr here
```

→ `SalLayoutGlyphsCache::GetLayoutGlyphs` (`vcl/source/gdi/impglyphitem.cxx:325`): on **glyph-cache miss** with `layoutCache == nullptr`:

```cpp
// impglyphitem.cxx:444-452
std::shared_ptr<const vcl::text::TextLayoutCache> tmpLayoutCache;
if (layoutCache == nullptr)
{
    tmpLayoutCache = vcl::text::TextLayoutCache::Create(text);  // script itemization
    layoutCache = tmpLayoutCache.get();
}
std::unique_ptr<SalLayout> layout = outputDevice->ImplLayout(text, nIndex, nLen, ...); // HarfBuzz shaping
```

### Why the existing caches do not absorb this

- `vcl::text::TextLayoutCache::Create` *is* globally memoized (LRU keyed by string, `vcl/source/text/TextLayoutCache.cxx:80-93`), so itemization is not the dominant cost.
- The dominant cost is the **HarfBuzz shape** executed on every glyph-cache miss. The glyph cache key (`impglyphitem.cxx:512-564`) contains `fontMetric`, `fontScale`, `mapMode`, `layoutMode`, `digitLanguage`, `rtl`, `disabledLigatures` — the space is measured with the *current* font at each call site, so mixed-format paragraphs (bold/italic/script runs) create a fresh entry per distinct font state, and the bounded LRU (`officecfg …GlyphsCacheSize`) evicts these single-glyph entries under document-wide churn — they get re-shaped over and over.
- These measurements also **bypass the paragraph layout context** (`SwLinePortionLayoutContext`, created at `itrform2.cxx:1578-1584`) that normal text portions use to reuse a whole-run layout.

**Conclusion:** the width of `' '`, `'-'`, and fill chars is a **constant per (physical font, output device)**. Re-measuring it per line / per tab stop / per paint re-runs the whole VCL layout pipeline (`KernArray` allocation + glyph-cache lookup + shape on miss + width/space-count loop `fntcache.cxx:1857-1877`) for a value that never changes.

---

## 2. Fix proposal (module sw, minimal diff)

**Primary — memoize the space (and hyphen) width in `SwTextSizeInfo`**, keyed by the physical font instance on the output device:

1. `sw/source/core/text/inftxt.hxx`
   - add forward declaration near line 45: `class LogicalFontInstance;` (declared in `include/vcl/outdev.hxx:67`, returned by `OutputDevice::GetFontInstance()` at `outdev.hxx:1240`);
   - in `SwTextSizeInfo` (protected section, ~line 175) add mutable members:
     ```cpp
     mutable LogicalFontInstance const* m_pSpaceSizeFont = nullptr;
     mutable SwTwips m_nSpaceSizeMapMode = 0;      // safety: key includes map mode
     mutable SwPositiveSize m_aSpaceSize;          // full size (height preserved)
     ```
2. Rewrite the inline `GetTextSize(const OUString&)` at `inftxt.hxx:805-808`:
   ```cpp
   inline SwPositiveSize SwTextSizeInfo::GetTextSize( const OUString &rText ) const
   {
       if (rText.getLength() == 1 && (rText[0] == ' ' || rText[0] == '-'))
       {
           // ensure the physical font is current on the device (idempotent; cf. SelectFont)
           m_pFnt->ChgPhysFnt( m_pVsh, *m_pOut );
           LogicalFontInstance const* pFI = m_pOut->GetFontInstance();
           if (pFI != m_pSpaceSizeFont || m_pOut->GetMapMode().GetHashValue() != m_nSpaceSizeMapMode)
           {
               m_pSpaceSizeFont = pFI;
               m_nSpaceSizeMapMode = m_pOut->GetMapMode().GetHashValue();
               m_aSpaceSize = GetTextSize(m_pOut, nullptr, rText, TextFrameIndex(0), TextFrameIndex(1));
           }
           return m_aSpaceSize;
       }
       return GetTextSize(m_pOut, nullptr, rText, TextFrameIndex(0), TextFrameIndex(rText.getLength()));
   }
   ```
   `m_pFnt` is a `SwFont*` (pointee mutable through a const object), so `ChgPhysFnt` (non-const, `swfont.cxx:935`) is callable; it is a cheap no-op when the font is unchanged (`m_bFontChg` guard) and mirrors `SelectFont()` (`inftxt.cxx:397-403`). All 11 call sites use `.Width()` only; caching the full `SwPositiveSize` keeps `Height()` correct.

**Alternative (more durable, bigger diff):** cache blank width in `SwFntObj` (`fntcache.hxx:60-80`) beside `m_nScrAscent/m_nPrtAscent/m_nScrHeight/m_nPrtHeight`, with a lazy `GetFontBlankWidth(SwViewShell const*, const OutputDevice&)` accessor. This survives across paragraphs and paints (SwTextSizeInfo instances are transient); recommended if profiling shows the per-paragraph memo insufficient.

**Explicitly NOT changed:** `frmform.cxx:1695` `GetTextSize("          ")` — leave it measured normally (the code divides by 10.0 at `:1698`, assuming 10 spaces = 10×single space; memoizing it via `10 * spaceWidth` would alter behavior if that assumption is imperfect). It is far less frequent than the `' '` call sites.

---

## 3. Verification level

- **V0:** build `sw` module (header change in `inftxt.hxx` recompiles dependent sw TUs).
- **V1 (recommended baseline):** run existing sw layout suites unchanged — `CppunitTest_sw_layoutwriter*`, `CppunitTest_sw_core_layout` (portion/XML dumps must be byte-identical), `CppunitTest_sw_uiwriter*`. No visible-output change is expected.
- **V2:** add a targeted layout test exercising (a) justified paragraph with paragraph-composer (≥2 spaces in previous line) and (b) tab stops with fill char, asserting exported layout XML is unchanged; optionally assert the memo hit via the existing `SV_STAT(nGetTextSize)` counters (`swfont.cxx:999/1050`) if stats are enabled.
- **V3:** manual — open a multi-line justified document with prop word-spacing + paragraph composer, scroll/repaint/print-preview, compare rendering before/after.

## 4. Risks

- **Stale cache:** mitigated by keying on `GetFontInstance()` + map-mode hash and by the idempotent `ChgPhysFnt` before the key check; the key is re-evaluated on every call.
- **Behavior change:** none for the `' '`/`'-'` widths (same measurement result, just memoized). `"          "` at frmform.cxx:1695 intentionally untouched.
- **Memory:** two pointers + two scalars per `SwTextSizeInfo` — negligible.
- **Multi-device:** key is per actual output device (`m_pOut`), so no cross-device (screen/printer) staleness.
- **CJK/grid paths:** the grid branch (`fntcache.cxx:1762-1809`) is only entered for CJK+grid and is unaffected; kana-compression is disabled in this overload (`inftxt.cxx:426`), and kerning is a per-font property already covered by the font-instance key (`fntcache.cxx:1875-1876`).

Shared-memory record: knowledge#276 (opt-f1, verified by inspector).
