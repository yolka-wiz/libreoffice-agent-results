# Where is ScDocShell defined

_Generated 2026-08-08T17:40:44_

`ScDocShell` is the document shell class for **Calc (the `sc` module)** in LibreOffice.

## Definition

**Class declaration:** `sc/source/ui/inc/docsh.hxx` (line 79)

```cpp
class SAL_DLLPUBLIC_RTTI ScDocShell final: public SfxObjectShell, public SfxListener
```

It's a 544-line header that:
- Inherits from `SfxObjectShell` (the generic document/shell base from `sfx2`) and `SfxListener`
- Includes `sfx2/objsh.hxx`, `sfx2/sfxmodelfactory.hxx`, `sfx2/viewsh.hxx`, `scdllapi.h`, `document.hxx`, `docuno.hxx`, etc.
- Declares the `ScDocShell`-related helpers like `TestImportDBF`

## Implementation

`sc/source/ui/docshell/docsh.cxx` (≈2900+ lines). The two constructors:

- Line 2926: `ScDocShell::ScDocShell(const SfxModelFlags i_nSfxCreationFlags, ScDocumentMode docMode)`
- Line 2931: `ScDocShell::ScDocShell(const SfxModelFlags i_nSfxCreationFlags, const std::shared_ptr<ScDocument>& pDoc)`

This file owns the shell's `SfxObjectShell` virtuals (`DoInitNew`, `DoLoad`, `DoSave`, `GetMedium`, etc.) and Calc-specific logic (document model access via `GetDocument()`, undo, charts, import/export).

## Ownership / usage

- **Module:** `sc` (Calc). `ScDocShell` is the central object that binds the Calc document model (`ScDocument`) to the SFX framework, bridging UI views (`ScTabViewShell`, `ScViewData`) and the model.
- Forward declarations of `ScDocShell` appear widely across `sc` (e.g., `sc/source/ui/inc/undotab.hxx`, `sc/source/ui/vba/vbarange.hxx`, `sc/source/ui/vba/vbaeventshelper.hxx`) and in `compilerplugins` results as well.
- `ScDocument` (in `sc/inc/document.hxx`) holds a back-reference to `ScDocShell` (`class ScDocShell * pDocShell`) so the model can reach the shell.

Note: `ScDocShell` is only a *declaration* (in its header); the actual class definition of `ScDocShell` is finalized at the header — the `final` keyword and `SAL_DLLPUBLIC_RTTI` are in `docsh.hxx`, while all member function bodies live in `docsh.cxx`.
