# tdf#90152 — locale-aware, readable printed comments header (sw)

_Generated 2026-08-10T11:14:38_

# tdf#90152 — Formatting of printed comments is hard to read

## Status
- **Fix implemented** on branch `writer/tdf-90152-postit-print-locale`, commit `76898c2bf` (`tdf#90152 make printed comments header locale-aware and readable`), one file changed: `sw/source/core/doc/doc.cxx` (+39/−11).
- **Verification level: V1 (static).** Not compiled — no build environment available in this workspace (shallow snapshot, no workdir/instdir). All referenced APIs and signatures were verified against the tree.
- **Push to origin: NOT completed** — `git push` timed out after 60 s on every attempt (5 tries). The branch + commit exist locally; needs a manual push: `git push -u origin writer/tdf-90152-postit-print-locale`.

## Root cause
`lcl_FormatPostIt()` (`sw/source/core/doc/doc.cxx:555-604`) printed each comment as a post-it page during print/PDF export with:
- hard-coded separator `static char const sTmp[] = " : ";` (L562);
- one flat plain-text line `Page : <n> Line : <m> Author : <name> <date> [RESOLVED]` with fixed order, hard-coded single spaces between fields, and only the date localized;
- a single `InsertString` with no character formatting → labels indistinguishable from values.

Labels themselves are localized (`STR_POSTIT_PAGE/LINE/AUTHOR`); only the punctuation/order/formatting were hard-coded.

## Fix (what/why)
1. **Locale-aware field separation** — fields are now separated by `LocaleDataWrapper::getListSep() + " "` (`SvtSysLocale` was already in use for the date). en-US output: `Page : 1, Line : 2, Author : Name 01/02/2024`; other locales get their own separator.
2. **Bold labels** — the localized labels (Page / Line / Author) are printed bold so they stand out from the values.

### Implementation note (important correction to earlier analysis)
The previously proposed "empty-hint" idiom — `InsertPoolItem(bold)` then `InsertString(label)`, then `InsertString(value)` — does **not** keep the values normal: a character-attribute hint whose end coincides with the insert position *extends* over the newly inserted text (`ndtxt.cxx` `Update()`, L1401–1458; `DontExpand` is false by default; interactive typing only stops this via `SwDoc::DontExpandFormat`). Everything after the first label would have become bold.

Instead, the header is built as one string, inserted with a single `InsertString`, and then each label range is bolded explicitly:
- record label ranges while building the string (`std::vector<std::pair<sal_Int32,sal_Int32>>`),
- capture `nInsPos` **before** `InsertString` (the caret moves past the inserted text),
- for each range create `SwPaM aLabel(SwPosition(*pTextNode, nInsPos+off1), SwPosition(*pTextNode, nInsPos+off2))` and call `pIDCO->InsertPoolItem(aLabel, SvxWeightItem(WEIGHT_BOLD, RES_CHRATR_WEIGHT), SetAttrMode::DONTEXPAND)`.

This follows the proven selection path in `lcl_InsAttr` (`DocumentContentOperationsManager.cxx:1853-1883`, hints created over the exact range; word-boundary expansion only applies to empty selections, L1745) and the pattern in `doctxm.cxx:2154-2188`.

## API verification (static)
- `SvxWeightItem(FontWeight, sal_uInt16)` — `include/editeng/wghtitem.hxx:42` (include added after `udlnitem.hxx`; header pulls in `tools/fontenum.hxx` for `WEIGHT_BOLD`).
- `RES_CHRATR_WEIGHT` — `sw/inc/hintids.hxx:218` (already included).
- `getListSep()` — `include/unotools/localedatawrapper.hxx:227` (already included).
- `InsertPoolItem(const SwPaM&, const SfxPoolItem&, SetAttrMode, ...)` — `sw/inc/IDocumentContentOperations.hxx:234`.
- `SwPosition(const SwContentNode&, sal_Int32)` — `sw/inc/pam.hxx:47`; `SwPaM(const SwPosition&, const SwPosition&)` — `pam.hxx:198`; `SwPosition::GetContentIndex()` — `pam.hxx:85`; `SwNode::GetTextNode()` — `node.hxx:179`.
- `SetAttrMode` — `sw/inc/swtypes.hxx:134` (transitively available via `ndtxt.hxx`).

## V2 test design (recommended, not run)
`sw/qa/extras/unowriter/unowriter.cxx`, modeled on `testRenderablePagePosition` (L934–973): annotated doc + `PrintAnnotationMode=2` (post-it pages) via `XRenderable::getRendererCount` with `IsPrinter`/`RenderDevice`, locale pinned to en-US; assert the post-it pages render and the header labels carry the bold `CharWeight`. No existing test asserts the old header, so nothing else needs updating.

## Risks / follow-ups
- Output for non-en-US locales changes intentionally (locale list separator); a V2 test must pin the locale.
- Field order remains fixed in C++ (translators cannot reorder without an optional `%1–%4` template string; the conditional Line field complicates that) — noted as follow-up.
- Hints are written only to the throw-away post-it document; no user-document/undo impact.

