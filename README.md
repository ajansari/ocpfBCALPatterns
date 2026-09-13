# OnlyCopilotFans Business Central AL Patterns

**Created by:** AJ Ansari
**Last Updated:** September 13, 2026

A standalone reference library of reusable Business Central AL coding patterns — each one
extracted from a real bug found and fixed on an actual project, then generalized so it can be
recognized and applied quickly the next time the same class of problem shows up on a different
extension. This is its own project, meant to travel with AJ Ansari across engagements, independent
of any single Business Central extension it happens to have been extracted from.

Each pattern lives in its own self-contained Markdown file: what the pattern is for, when to use
it, the symptom, the verified root cause, the generalized fix, a worked example from the project
it was extracted from, and the caveats worth checking before applying it blind. Every platform
claim is verified against official Microsoft Learn documentation (quoted verbatim, with a source
link) rather than asserted from memory.

## Patterns

| Pattern | Description | File |
|---|---|---|
| Reading a `SubPageLink` value — filter group 4, not group 0 | A newly created child record (a subform line, a related record created in a filtered context) comes out blank or "outside the filter" because the parent-link filter was read from the wrong filter group. Covers the fix, a hidden-field defensive measure, and a `TestField` guard to fail loudly instead of silently. | [`Pattern-SubPageLink-FilterGroup4.md`](Pattern-SubPageLink-FilterGroup4.md) |
| `Record.Init()` does not clear the primary key | Reusing one record variable across multiple `Insert()` calls (wizards, sample-data seeding, import loops) causes a duplicate-key error on the second insert that looks like a No. Series bug but isn't — `Init()` never clears the primary key. Covers two fixes (with a clear recommendation), a documented anti-pattern to avoid, and a diagnostic checklist. | [`Pattern-Init-Does-Not-Clear-Primary-Key.md`](Pattern-Init-Does-Not-Clear-Primary-Key.md) |
