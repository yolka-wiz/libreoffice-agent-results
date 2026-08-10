# tdf#90152 — printed comments post-it header: root cause + fix proposal (sw)

_Generated 2026-08-10T10:49:47_

# tdf#90152 — Formatting of printed comments is hard to read (locale-aware post-it format), module sw

**Upstream status: GREEN** — tdf#90152 is NEW (easyHack, needsDevEval, 0 comments); no Gerrit patches reference it. Nothing upstream yet; the fix below is ours to propose.

## Root cause

`sw/source/core/doc/doc.cxx` — `lcl_FormatPostIt()` (L555–604), the function that lays out each comment as a "post-it" page during printing/PDF export with annotations:

- **L562**: separator is hard-coded: `static char const sTmp[] = " : ";`
- **L578–595**: the header is built as **one flat plain-text line** with a fixed field order Page → Line → Author → Date and hard-coded single spaces:
  `Page : <n> Line : <m> Author : <name> <date> [RESOLVED]`
- **L589–592**: only the *date* is localized (`SvtSysLocale` → `GetLocaleData().getDate()`); the punctuation and the field order are baked into C++.
- **L595**: the whole line is inserted with a single `pIDCO->InsertString(...)` — **no character formatting**, so labels and values are visually indistinguishable.

The labels themselves **are** localized: `STR_POSTIT_PAGE/LINE/AUTHOR` at `sw/inc/strings.hrc:777–779`, loaded into `SwShellRes::aPostItPage/aPostItLine/aPostItAuthor` at `sw/source/uibase/utlui/initui.cxx:108–110`.

Caller chain (verified):
- `SwRenderData::CreatePostItData` — `sw/source/core/view/printdata.cxx:50–63` (temp `SwDoc` + `SwViewShell`; spell check disabled)
- `SwDoc::UpdatePagesForPrintingWithPostItData` — `sw/source/core/doc/doc.cxx:780–844`; reads `PrintAnnotationMode` at L786, iterates sorted post-it fields, calls `lcl_FormatPostIt` at L833.

## Fix proposal

In `sw/source/core/doc/doc.cxx`:

1. **Add include** after L54 (`#include <editeng/udlnitem.hxx>`):
   ```cpp
   #include <editeng/wghtitem.hxx>   // SvxWeightItem
   ```

2. **Replace the flat-string block L578–595** with a per-part insert that (a) prints the labels bold via the *empty-hint* attribute idiom and (b) separates fields with the locale list separator:
   ```cpp
   SvtSysLocale aSysLocale;
   const OUString aListSep = aSysLocale.GetLocaleData().getListSep() + " ";
   auto lcl_InsertBoldLabel = [pIDCO, &aPam]( const OUString& rLabel )
   { pIDCO->InsertPoolItem( aPam, SvxWeightItem( WEIGHT_BOLD, RES_CHRATR_WEIGHT ) );
     pIDCO->InsertString( aPam, rLabel ); };

   lcl_InsertBoldLabel( SwViewShell::GetShellRes()->aPostItPage );
   pIDCO->InsertString( aPam, sTmp + OUString::number( nPageNo ) );
   if( nLineNo )
   { pIDCO->InsertString( aPam, aListSep );
     lcl_InsertBoldLabel( SwViewShell::GetShellRes()->aPostItLine );
     pIDCO->InsertString( aPam, sTmp + OUString::number( nLineNo ) ); }
   pIDCO->InsertString( aPam, aListSep );
   lcl_InsertBoldLabel( SwViewShell::GetShellRes()->aPostItAuthor );
   pIDCO->InsertString( aPam, sTmp + pField->GetPar1() + " "
                          + aSysLocale.GetLocaleData().getDate( pField->GetDate() ) );
   if( pField->GetResolved() )
       pIDCO->InsertString( aPam, " " + SwResId( STR_RESOLVED ) );
   ```
   Leave the `SplitNode` + body (`GetPar2`) part (L597–603) untouched.

   Resulting header (en-US): **Page : 1**, Line : 2, **Author : Name** 01/02/2024 — labels bold, fields separated by locale list separator (`getListSep()`, e.g. `, ` en-US, `; ` in some locales).

### API verification (all cited lines checked in the tree)
- `getListSep()` — `include/unotools/localedatawrapper.hxx:227`; header already included at `doc.cxx:58`; `SvtSysLocale` already used at L589 (`unotools/syslocale.hxx` included at `doc.cxx:50`).
- `SvxWeightItem( const FontWeight eWght, const sal_uInt16 nId )` — `include/editeng/wghtitem.hxx:42–43` (header **not** yet in doc.cxx).
- `WEIGHT_BOLD` — `include/tools/fontenum.hxx:115`.
- `RES_CHRATR_WEIGHT` — `sw/inc/hintids.hxx:218` (`hintids.hxx` included at `doc.cxx:45`).
- `InsertPoolItem` with a char attr and no selection inserts an *empty hint* — `sw/inc/IDocumentContentOperations.hxx:228–234`.
- `InsertString` defaults to `SwInsertFlags::EMPTYEXPAND` — `sw/inc/IDocumentContentOperations.hxx:166–167`; empty-hint expansion over the just-inserted text at `sw/source/core/txtnode/ndtxt.cxx:2556–2569` (hint start == insert end → start moved back by `nLen`), so only the label text becomes bold; text inserted after the hint end (the ` : <n>` value parts) stays normal.

### Optional follow-up (not required for this easyHack)
Full translator control of field *order* would need a localized template string in `sw/inc/strings.hrc` with `%1`–`%4` placeholders. Note the `Line` field is conditional (omitted when `nLineNo == 0`), which makes `%`-placeholders awkward; the bold + `getListSep()` change already delivers the readability + locale-awareness goal with minimal risk.

## Verification level

- **V1 (static) — baseline**: every API and the empty-hint/EMPTYEXPAND mechanic verified against the tree (cited above). The change has **not** been compiled or run.
- **V2 (targeted unit test) — recommended**: extend `sw/qa/extras/unowriter/unowriter.cxx` (pattern: `testRenderablePagePosition`, L934–973, which drives `XRenderable::getRendererCount` with `IsPrinter` + `RenderDevice`) with an annotated document and `PrintAnnotationMode = 2`; `PrintAnnotationMode` is consumed from `SwPrintUIOptions` at `doc.cxx:786` (UNO path `unotxdoc.cxx:2816`). Assert renderer count grows by the expected post-it pages. Pin the test locale to en-US so the separator assertion is deterministic.
- **V3 (manual/GUI check)**: print/export PDF with annotations in ≥2 locales and eyeball that labels are bold, separators are locale-appropriate, and text stays on the post-it page.

## Risks

1. **EMPTYEXPAND bold-hint mechanic** — verified in `ndtxt.cxx:2556–2569`, but must be confirmed by an actual render test (only labels bold, values normal). Low-medium.
2. **Output change for non-US locales is intentional** (this is the point of the fix); en-US output changes from single spaces to `, ` after numbers — any golden/screenshot test must pin the locale.
3. **Fixed field order remains** in C++ (Page→Line→Author→Date). Translators cannot reorder without the optional template-string follow-up; acceptable for this easyHack.
4. **Temp doc safety** — hints are inserted into the throw-away post-it doc (`printdata.cxx:62`), not the user document; no undo/redo or user-doc impact.
5. **STR_RESOLVED** (L593–594) is appended unchanged after author+date; fine.

## Existing tests
`rg "PostIt|PrintAnnotationMode" sw/qa` — only `.fodt` data files carrying the `PrintAnnotationMode` config item and unrelated tests (`sw/qa/core/uwriter.cxx:692` inserts a `SwPostItField` for field-type behavior; `sw/qa/core/txtnode/txtnode.cxx:573` tests the UI `PostItMgr`). **No test asserts the current header**, so no golden text needs updating.

## Suggested commit
`tdf#90152 make printed comments header locale-aware and readable` — single file `sw/source/core/doc/doc.cxx` (+ optional `sw/qa/extras/unowriter/unowriter.cxx` test).

