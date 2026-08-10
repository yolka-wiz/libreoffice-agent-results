# tdf#90152 — printed comments post-it header not readable / not locale-aware (sw)

_Generated 2026-08-10T10:13:29_

## Investigation: tdf#90152 — Formatting of printed comments is hard to read (locale-aware post-it format)

**Module:** sw — **Upstream:** GREEN (tdf#90152 NEW, easyHack/needsDevEval, unassigned, last touched 2026-03-31; no Gerrit changes reference it)

---

### Root cause

`sw/source/core/doc/doc.cxx` — `lcl_FormatPostIt()` (lines 555–604), called from `doc.cxx:833` when building the temporary post-it document (`SwRenderData::CreatePostItData`, `sw/source/core/view/printdata.cxx:50-63`).

Evidence:

| Line | Code | Problem |
|---|---|---|
| `doc.cxx:562` | `static char const sTmp[] = " : ";` | label↔value separator hard-coded in C++, not localized |
| `doc.cxx:578-595` | one flat `OUString aStr` = `Page : <n> Line : <m> Author : <name> <date> [RESOLVED]` | fixed field order, hard-coded single spaces between fields |
| `doc.cxx:589-592` | `SvtSysLocale aSysLocale; ... aSysLocale.GetLocaleData().getDate(...)` | only the **date** is locale-formatted |
| `doc.cxx:595` | single `pIDCO->InsertString(aPam, aStr)` | whole header inserted as one plain-text run → labels visually indistinguishable from values (hard to read) |

The label *texts* are already localized (`sw/inc/strings.hrc:777-779` → `shellres` → `sw/source/uibase/utlui/initui.cxx:108-110`), but the surrounding punctuation, field order and lack of emphasis are hard-coded in C++.

No existing test asserts the current header (`rg` over `sw/qa` finds only `PrintAnnotationMode` config items in `.fodt` test data).

---

### Fix proposal (exact)

**File:** `sw/source/core/doc/doc.cxx`

**1) Add include** after line 54 (`#include <editeng/udlnitem.hxx>`):
```cpp
#include <editeng/wghtitem.hxx>
```
(`unotools/localedatawrapper.hxx` is already included at `doc.cxx:58`.)

**2) Replace the string-build block, lines 578–595:**
```cpp
    SvtSysLocale aSysLocale;
    const OUString aListSep = aSysLocale.GetLocaleData().getListSep() + " ";
    auto lcl_InsertBoldLabel = [pIDCO, &aPam]( const OUString& rLabel )
    {
        pIDCO->InsertPoolItem( aPam, SvxWeightItem( WEIGHT_BOLD, RES_CHRATR_WEIGHT ) );
        pIDCO->InsertString( aPam, rLabel );
    };
    lcl_InsertBoldLabel( SwViewShell::GetShellRes()->aPostItPage );
    pIDCO->InsertString( aPam, sTmp + OUString::number( nPageNo ) );
    if( nLineNo )
    {
        pIDCO->InsertString( aPam, aListSep );
        lcl_InsertBoldLabel( SwViewShell::GetShellRes()->aPostItLine );
        pIDCO->InsertString( aPam, sTmp + OUString::number( nLineNo ) );
    }
    pIDCO->InsertString( aPam, aListSep );
    lcl_InsertBoldLabel( SwViewShell::GetShellRes()->aPostItAuthor );
    pIDCO->InsertString( aPam, sTmp + pField->GetPar1() + " " +
                         aSysLocale.GetLocaleData().getDate( pField->GetDate() ) );
    if( pField->GetResolved() )
        pIDCO->InsertString( aPam, " " + SwResId( STR_RESOLVED ) );
```
Do **not** touch the body part after it (line 597 `SplitNode` + `GetPar2()` insert, lines 597-603).

**Why this works (verified against headers):**
- **Bold labels** — `InsertPoolItem` with no selection inserts an *empty hint* (`IDocumentContentOperations.hxx:228-230`); `InsertString` defaults to `SwInsertFlags::EMPTYEXPAND` (`IDocumentContentOperations.hxx:166-167`), and `ndtxt.cxx:2556-2569` expands empty hints at the insert position over the inserted text → the label run becomes bold. Text appended afterwards (the value) sits at the end of the hint and stays regular weight → labels pop, values stay normal.
- **Locale-aware separators** — `LocaleDataWrapper::getListSep()` (`include/unotools/localedatawrapper.hxx:227`) yields e.g. `, ` (en-US) vs `; ` (de-DE) between the Page/Line/Author groups, replacing the hard-coded single spaces.
- **API signatures confirmed:** `SvxWeightItem(const FontWeight, const sal_uInt16)` (`include/editeng/wghtitem.hxx:42`); `WEIGHT_BOLD` (`include/tools/fontenum.hxx:115`); `RES_CHRATR_WEIGHT` (`sw/inc/hintids.hxx:218`); `pField->GetPar1()/GetDate()/GetResolved()` (`sw/inc/docufld.hxx:545,557,575`).

**Optional (more thorough l10n, larger change):** replace the whole header with a single localized template in `sw/inc/strings.hrc` using `%1`–`%4` placeholders (`STR_POSTIT_FORMAT`), so translators can reorder fields; `sTmp` `" : "` would then also move into the template. Recommended only if maintainers want full field-order l10n; the bold+list-sep fix already resolves the *readability* and *locale punctuation* aspects of the bug.

---

### Verification level

- **Baseline for this proposal: V1** (static): all APIs, includes, enum values and the empty-hint/EMPTYEXPAND mechanism verified against headers/sources as cited above. (V0 = not applied/compiled — the fix is a proposal for the Fix Engineer.)
- **Target: V2** — build `sw` and add a unit test in `sw/qa/extras/unowriter/unowriter.cxx` following the existing `testRenderablePagePosition` renderer-count pattern (lines 934-973, `XRenderable::getRendererCount` + `PrintAnnotationMode=2`), asserting: (a) header text contains the label strings and (b) `SvxFont`/`SwTextAttr` weight on the label ranges is `WEIGHT_BOLD`.
- **V3** (full): manual visual check of File → Print → Comments (post-it mode) output across 2-3 locales (e.g. en-US vs de-DE) for both readability and separator change.

### Risks

1. **Empty-hint expansion semantics** — relies on `InsertString` default `EMPTYEXPAND`; if a caller ever changes the default flag, labels silently lose bold. Mitigation: keep default, or pass `SwInsertFlags::EMPTYEXPAND` explicitly.
2. **`getListSep()` vs colon** — only the *between-field* separator is localized; `" : "` (label↔value) stays as today. This is the minimal-change scope; full l10n needs the template-string option.
3. **Localization of labels** — `aPostItPage/aPostItLine/aPostItAuthor` are `SwResId` strings loaded once (static `ShellRes`); translations are fine, but the bold attribute is applied per-insert, so no stale-resource risk.
4. **Existing golden/visual tests** — none assert the header (verified via rg in sw/qa), so no expected test breakage; the only risk is snapshot-based layout tests if any render post-it mode (none found).
5. **Windows CR-stripping / body path** untouched — no interaction with lines 597-603.

