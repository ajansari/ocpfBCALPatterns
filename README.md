# OnlyCopilotFans Business Central AL Patterns

**Created by:** AJ Ansari
**Last Updated:** October 10, 2026

A standalone reference library of reusable Business Central AL coding patterns — each one
extracted from a real bug found and fixed on an actual project, then generalized so it can be
recognized and applied quickly the next time the same class of problem shows up on a different
extension. This is its own project, meant to travel with AJ Ansari across engagements, independent
of any single Business Central extension it happens to have been extracted from.

Each pattern lives in its own self-contained Markdown file: what the pattern is for, the versions
it applies to, the symptom, the verified root cause, the generalized fix, a minimal worked
reproduction that stands on its own without the originating project, where else the same shape
shows up, and the caveats worth checking before applying it blind. Every platform claim is
verified against official Microsoft Learn documentation or the shipped symbol files (quoted, with
a source link or object reference) rather than asserted from memory.

## Patterns

| Pattern | Description | File |
|---|---|---|
| Reading a parent link set by the platform — filter group 4 for `SubPageLink`, group 0 for `RunPageLink` | A child record created inside its parent's page comes out with the link field blank, or vanishes with "the entry is outside the filter," because code read the parent link from the wrong filter group — or from only one of the two groups a page can be opened with. Covers a two-group read that restores the filter group on every path, a hidden link field on the child page, a `TestField` guard so the table fails loudly, and `DelayedInsert` on the child list. | [`Pattern-SubPageLink-FilterGroup4.md`](Pattern-SubPageLink-FilterGroup4.md) |
| `Record.Init()` does not clear the primary key — and it *does* clear everything else | Two opposite failures from one method. Reusing a record variable across several inserts fails on the second with "already exists" because `Init()` leaves the primary key alone — it looks like a No. Series bug but isn't. Calling `Init()` on a record the platform already pre-filled (a page trigger) wipes the pre-filled values because `Init()` resets every non-key field. Covers `Clear()` or a fresh variable for loops, never calling `Init()` in page triggers, and a discriminating test to tell the two apart. | [`Pattern-Init-Does-Not-Clear-Primary-Key.md`](Pattern-Init-Does-Not-Clear-Primary-Key.md) |
| A number-series field gets its lookup from `TableRelation`, and numbering follows Business Foundation | The number-series field on a setup card or assisted setup wizard opens no lookup, opens the wrong page, or the picked value never reaches the setup record — so the extension cannot number its first record. Covers the `Code[20]` field with `TableRelation = "No. Series"` on the setup table, binding a wizard to a temporary copy of the setup table rather than page variables, offering to create a series safely, and numbering via the Business Foundation `"No. Series"` codeunit (BC 24+) instead of the obsolete `NoSeriesManagement`. | [`Pattern-NoSeries-Setup-Field-And-Numbering.md`](Pattern-NoSeries-Setup-Field-And-Numbering.md) |
| A `PromptDialog` closes with `Action::OK`, not `LookupOK` — and the save belongs to whichever side owns the error | The "Keep it" button on a Copilot prompt dialog does nothing: `OnQueryClosePage` compared `CloseAction` to `Action::LookupOK`, which a `PromptDialog` never sends; `systemaction(OK)` closes with `Action::OK` and takes no `OnAction`. Covers the correct case branch, returning `false` to keep the dialog open when there is nothing to keep, and the two Microsoft-documented places to perform the save: inside `OnQueryClosePage`, or in the caller from the `Action` that `RunModal()` returns. Corrects the folk claim that errors inside `OnQueryClosePage` are swallowed. | [`Pattern-PromptDialog-OK-Closes-With-Action-OK.md`](Pattern-PromptDialog-OK-Closes-With-Action-OK.md) |
| Capturing an API key in a page — `SecretText` cannot be a page field, and `IsolatedStorage.Set` is the only setter | A wizard must take an API key from the user and store it without a table field or debugger exposure. Binding a `SecretText` to a page field fails (AL0156); `IsolatedStorage.SetSecret` does not exist (AL0132). Covers the `Text` page variable with `ExtendedDatatype = Masked` copied to a `SecretText` in `OnValidate`, `IsolatedStorage.Set(Key, SecretText, DataScope)` (runtime 13+) and the matching `Get`/`Contains`/`Delete`, a configured-flag on the setup table, a Test Connection gate before Finish, and the write-transaction rule that forbids a modal page after `Set`. | [`Pattern-SecretText-Page-Capture-And-IsolatedStorage.md`](Pattern-SecretText-Page-Capture-And-IsolatedStorage.md) |
| A page extension can `addlast(Prompting)` even when the base page declares no Prompting area | A Copilot action was parked in the Actions menu because "the base page has no Prompting area". The anchor for `addlast` is an area name; the extension creates the area, and only the misspelling (`Prompts`) fails. Covers the `area(Prompting)` syntax from Microsoft Learn, the runtime 13/14 supported page-type matrix, `Sparkle` imagery and verb-first captions, hiding the action with `IsCapabilityRegistered` where Copilot cannot run, and the one legitimate reason to fall back to a standard action. Compiler-verified on page 39 General Journal in BC 27.5. | [`Pattern-Prompting-Area-On-Page-Extension.md`](Pattern-Prompting-Area-On-Page-Extension.md) |
