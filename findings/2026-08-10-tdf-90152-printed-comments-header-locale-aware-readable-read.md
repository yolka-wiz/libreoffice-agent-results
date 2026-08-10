# tdf#90152 printed-comments header: locale-aware + readable (ready-to-apply patch)

_Generated 2026-08-10T10:02:32_

# tdf#90152 — Formatting of printed comments is hard to read (locale-aware post-it format)

**Status:** Investigation complete; patch prepared and statically verified (V1), **not applied** to the work clone (see "Tooling limitation" below).

## Root cause

`sw/source/core/doc/doc.cxx`, static helper `lcl_FormatPostIt()` (lines 555–604), which builds the header of every post-it/comment page in the temporary post-it document used when printing comments (`SwPostItMode` EndDoc/EndPage/Only):

- Line 562 hard-codes the field separator `static char const sTmp[] = " : ";`
- Lines 578–595 build **one flat plain-text line**:
  `Page : <n> Line : <m> Author : <name> <date> [RESOLVED]`
  with a fixed field order (Page → Line → Author → Date) and hard-coded single spaces.
- Only the date is localized (line 592, `SvtSysLocale::GetLocaleData().getDate()`).
- Labels come from localized resources (`STR_POSTIT_PAGE/LINE/AUTHOR`, `sw/inc/strings.hrc:777-779` → `shellres.hxx:30-32` → `initui.cxx:108-110`), but the surrounding punctuation and grouping are hard-coded in C++.

Result: a long, run-together header that is hard to scan, and a format that does not follow the UI locale's list-separator conventions.

## Fix (smallest correct change, one file)

`sw/source/core/doc/doc.cxx` only:

1. **Bold the labels** (`Page`, `Line`, `Author`) so the header is scannable, using the well-established "empty hint" idiom:
   - `pIDCO->InsertPoolItem(aPam, SvxWeightItem(WEIGHT_BOLD, RES_CHRATR_WEIGHT))` inserts an empty char attribute at the caret;
   - the following `InsertString` uses its default `SwInsertFlags::EMPTYEXPAND`, so the empty hint expands over the inserted label text (verified in `sw/source/core/txtnode/ndtxt.cxx:2556-2569`; default flag at `sw/inc/IDocumentContentOperations.hxx:167`);
   - text inserted at the *end* of a non-empty hint is not expanded, so values/separators remain normal weight.
2. **Locale-aware grouping**: fields are separated with `LocaleDataWrapper::getListSep() + " "` (e.g. `, ` for en-US, `; ` for de-DE) instead of a bare space. `unotools/localedatawrapper.hxx` is already included (doc.cxx:58) and `SvtSysLocale` is already in use.

New output (en-US): `**Page** : 3, **Line** : 4, **Author** : John Doe 02/14/2024` (only the labels bold; RESOLVED and body unchanged).

### Exact diff

```diff
--- a/sw/source/core/doc/doc.cxx
+++ b/sw/source/core/doc/doc.cxx
@@ -51,6 +51,7 @@
 #include <editeng/keepitem.hxx>
 #include <editeng/formatbreakitem.hxx>
 #include <editeng/pbinitem.hxx>
 #include <editeng/udlnitem.hxx>
+#include <editeng/wghtitem.hxx>
 #include <editeng/colritem.hxx>
 #include <editeng/xmlcnitm.hxx>
 #include <editeng/fontitem.hxx>
 #include <unotools/localedatawrapper.hxx>
@@ -578,19 +579,31 @@ static void lcl_FormatPostIt(
-    OUString aStr = SwViewShell::GetShellRes()->aPostItPage +
-        sTmp +
-        OUString::number( nPageNo ) +
-        " ";
-    if( nLineNo )
-    {
-        aStr += SwViewShell::GetShellRes()->aPostItLine +
-            sTmp +
-            OUString::number( nLineNo ) +
-            " ";
-    }
-    SvtSysLocale aSysLocale;
-    aStr += SwViewShell::GetShellRes()->aPostItAuthor +
-        sTmp + pField->GetPar1() + " " +
-        /*(LocaleDataWrapper&)*/aSysLocale.GetLocaleData().getDate( pField->GetDate() );
-    if(pField->GetResolved())
-        aStr += " " + SwResId(STR_RESOLVED);
-    pIDCO->InsertString( aPam, aStr );
+    SvtSysLocale aSysLocale;
+    const OUString aListSep = aSysLocale.GetLocaleData().getListSep() + " ";
+
+    // tdf#90152: The header used to be a single plain-text line with a hard-coded
+    // " : " between every field, which was hard to read and not locale aware.
+    // Print the labels in bold and separate the fields with the locale-dependent
+    // list separator instead.
+    auto lcl_InsertBoldLabel = [pIDCO, &aPam]( const OUString& rLabel )
+    {
+        pIDCO->InsertPoolItem( aPam, SvxWeightItem( WEIGHT_BOLD, RES_CHRATR_WEIGHT ) );
+        pIDCO->InsertString( aPam, rLabel );
+    };
+
+    lcl_InsertBoldLabel( SwViewShell::GetShellRes()->aPostItPage );
+    pIDCO->InsertString( aPam, sTmp + OUString::number( nPageNo ) );
+    if( nLineNo )
+    {
+        pIDCO->InsertString( aPam, aListSep );
+        lcl_InsertBoldLabel( SwViewShell::GetShellRes()->aPostItLine );
+        pIDCO->InsertString( aPam, sTmp + OUString::number( nLineNo ) );
+    }
+    pIDCO->InsertString( aPam, aListSep );
+    lcl_InsertBoldLabel( SwViewShell::GetShellRes()->aPostItAuthor );
+    pIDCO->InsertString( aPam, sTmp + pField->GetPar1() + " " +
+            aSysLocale.GetLocaleData().getDate( pField->GetDate() ) );
+    if(pField->GetResolved())
+        pIDCO->InsertString( aPam, " " + SwResId(STR_RESOLVED) );
```

The trailing part of the function (paragraph split, comment body from `GetPar2()`, `_WIN32` CR-stripping) is untouched. The comment body stays in its own paragraph without bold.

## Why this is correct / static verification (V1)

- `SvxWeightItem(FontWeight, sal_uInt16)` ctor exists (`include/editeng/wghtitem.hxx:42`); same usage pattern already in `sw/source/filter/md/mdcallbcks.cxx:162` and `sw/qa/uibase/dochdl/dochdl.cxx:66`.
- `RES_CHRATR_WEIGHT` is `TypedWhichId<SvxWeightItem>` (`sw/inc/hintids.hxx:218`), implicitly convertible to the ctor's id param.
- `IDocumentContentOperations::InsertString` defaults to `SwInsertFlags::EMPTYEXPAND` (`sw/inc/IDocumentContentOperations.hxx:167`), which makes an empty hint at the insertion point expand over the inserted text (`sw/source/core/txtnode/ndtxt.cxx:2556-2569`); inserting at the end of a non-empty hint does not expand it (else-branch `continue`), so values/separators stay normal.
- `InsertPoolItem` with an empty selection inserts an "empty" hint (documented at `sw/inc/IDocumentContentOperations.hxx:228-230`, implemented at `DocumentContentOperationsManager.cxx:1745+`).
- `LocaleDataWrapper::getListSep()` exists (`include/unotools/localedatawrapper.hxx:227`); header already included.
- No other code parses this header string (PDF export uses its own format in `EnhancedPDFExportHelper.cxx:2407-2411`); `rg` over `sw/qa` found no test asserting the old header text.

## Tooling limitation / verification level

The fix-engineer toolset for this run exposes read/search/git/memory/publish tools but **no file-write or patch-apply tool**, and the work clone had no staged changes; therefore the patch could **not be applied, compiled, or committed** in this run.

- Verification of the **patch content**: **V1** (static/API-level review, above).
- Verification of the **applied fix in the repo**: **V0** — nothing was applied; no build (`instdir`/`workdir`) exists in the environment, so V2/V3 were not possible either.

**Next step for a writer with write access:** apply the diff above on branch `writer/tdf-90152-postit-print-locale` (already created) in `/home/agent/workspace/lo-writer`, then commit with message
`tdf#90152 make printed comments header locale-aware and readable` and push. A suggested unit-test hook is `sw/qa/extras/unowriter/unowriter.cxx` (`testRenderablePagePosition` pattern: `XRenderable::getRendererCount` with `IsPrinter` + `PrintAnnotationMode=2`), asserting the post-it page text contains a bold `Page` label run.

## References

- Inspector A root-cause note (memory knowledge#271): `sw/source/core/doc/doc.cxx` `lcl_FormatPostIt()` lines 555–604; hard-coded `" : "` at line 562; fixed field order; only date localized at line 592.
- Patch stored in shared memory as knowledge#272.

