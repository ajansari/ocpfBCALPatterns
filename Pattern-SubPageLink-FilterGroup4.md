# Pattern: Reading a `SubPageLink` value — the parent link lives in filter group 4, not group 0

**A newly created child record comes out blank / orphaned, or "falls outside the filter," because code tried to read the parent link with a bare `GetFilter()`.**

When a child record is created from inside a view that is *implicitly* filtered to a parent — a `ListPart` subform bound by `SubPageLink`, a report `dataitem` bound by `DataItemLink` — the filter that establishes "this child belongs to that parent" is **not** in the default filter group that `Rec.GetFilter(...)` reads. It lives in filter group **4 ("Link")**. Code that reads the link value without switching filter groups first reads an empty string, silently assigns nothing, and produces a child row whose linking field is blank: invisible in every filtered view, orphaned in the data, and often accompanied by the runtime complaining that the record it just created is outside the current filter. This pattern is the correct read, plus the two defensive measures that make the failure loud instead of silent if it ever happens anyway.

---

## Applicability and version

This is a **Business Central AL platform pattern**, not tied to any specific extension, table, or vertical. It applies to any AL object that reads a filter which the platform set on its behalf.

**Version:** core, long-standing AL platform behavior — **no minimum BC version applies.** Microsoft Learn's `Record.FilterGroup([Integer])` reference states its availability as:

> **Version**: *Available or changed with runtime version 1.0.*

Runtime version 1.0 is the first AL runtime, so the filter-group numbering — including group 4 being the `SubPageLink` / `DataItemLink` group — has been present for the entire life of the AL language. It was not introduced at, or changed by, any later BC release, and **no "designed for version XX+" cutoff should be claimed for it.**

**Verified/tested against, in the project this was extracted from:** BC v28.4 symbol files (Base Application `28.4.53241.54183`, System Application `28.4.53241.54101`, Business Foundation `28.4.53241.53312`, Application `28.4.53241.53312`, System `28.0.53984.0`), AL Runtime `17.0`, BC Application minimum `28.0.0.0`, recommended `28.4+`. Confirmed working on a live BC sandbox.

---

## Symptom

What a developer actually sees, in concrete terms:

- A user opens a parent document card (e.g. a new Bootcamp, a new order-like header), types into the first empty row of the embedded lines subform, and gets a runtime message along the lines of **"the view is filtered, and the entry is outside the filter."**
- The child record is nonetheless **created** — but with the linking field (`"Bootcamp No."`, `"Document No."`, `"Parent No."`, whatever it is) **blank**.
- That orphan row is then **invisible from every normal entry point**, because every normal entry point is filtered to a parent. It only shows up in an unfiltered list of the child table, which most users never open. Rows quietly accumulate.
- If the child table has a `FlowField`/statistics rollup on the parent (counts, sums, "seats remaining"), the parent's numbers are silently wrong, because the orphan rows are not counted against any parent.
- Adding a guard that reads the link from the filter **appears to fix it in code review and changes nothing at runtime.** This is the most expensive part of the symptom: the wrong fix is invisible. There is no compile error, no warning, no exception — `GetFilter` on a field with no filter in group 0 legitimately returns `''`, and assigning `''` to an already-blank field is a perfectly valid no-op. The code reads as defensive and correct, and does nothing at all.

---

## Root cause

AL maintains **multiple simultaneous filter groups** on a record. `SetFilter`/`SetRange`/`GetFilter` all operate on *whichever group is currently selected*, and the selected group defaults to 0. The platform puts the filters it sets on your behalf into other, reserved groups — so a plain `Rec.GetFilter(...)` is only ever looking at the group the *end user's* filter pane writes to, never at the group the *platform's* parent-link writes to.

Verified against Microsoft Learn, `Record.FilterGroup([Integer]) Method` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/record/record-filtergroup-method>

The documentation's own table of internally-used filter groups, quoted verbatim for the two rows that matter here:

> | Number | Name | Description |
> | --- | --- | --- |
> | 0 | Std | The default group where filters are placed when no other group has been selected explicitly. This group is used for filters that can be set from the filter dialogs by the end user. |
> | 4 | Link | Used for the filtering actions that result from the following: - [DataItemLink Property (Reports)](../../properties/devenv-dataitemlink-reports-property)- [SubPageLink Property](../../properties/devenv-subpagelink-property) |

That is the whole mechanism. The `SubPageLink` filter is in group **4**; `GetFilter` with no group switch reads group **0**; the two never meet.

The same page also documents, verbatim, that all groups apply at once — which is why the *view* is correctly filtered even though your *code* can't see the filter:

> All groups are active at all times. The only way to turn off a group is to remove the filters set in that group.

> Filters in different groups are all effective simultaneously.

And it warns — verbatim — about the difference between reading such a group and writing to it (see Caveats):

> It's possible to use one of the internally used groups. If you do this, you replace the filter that Dynamics 365 Business Central assumes is in this group. If, for example, you use filter group 4 in a page, you will replace the filtering that is actually the result of applying the [SubPageLink Property](../../properties/devenv-subpagelink-property). This could seriously alter the way pages and subpages interact.

For orientation, the other groups a developer is likely to want to read (same verbatim table):

> | 2 | Form | Used for the filtering actions that result from the following: - SetTableView Method (XMLport), SetTableView Method (Page) - SourceTableView Property - DataItemTableView Property. |
> | 3 | Exec | Used for the filtering actions that result from the following: - SubPageView Property - RunPageView Property |
> | 6 | Security | Used for applying security filters for user permissions. |

So: `SourceTableView` → group 2. `SubPageView` / `RunPageView` → group 3. `SubPageLink` / `DataItemLink` → group 4. Reaching for group 0 to find any of them is the bug.

---

## The pattern / fix

Three parts. Part 1 is the actual fix; parts 2 and 3 are the defensive measures that convert a silent orphan into a loud, actionable error if the link is ever genuinely absent for some other reason.

### Part 1 — read the link from filter group 4, and restore the previous group

```al
page 50101 "My Child Lines Subform"
{
    PageType = ListPart;
    SourceTable = "My Child Table";
    DelayedInsert = true;
    AutoSplitKey = false;

    // ... layout ...

    trigger OnNewRecord(BelowxRec: Boolean)
    var
        PrevFilterGroup: Integer;
        ParentNoFilter: Text;
    begin
        // Inert when the platform's own propagation already worked — only fires on blank.
        if Rec."Parent No." <> '' then
            exit;

        // SubPageLink filters live in filter group 4 ("Link"), not the default group 0 —
        // a plain Rec.GetFilter() reads group 0 and misses it entirely.
        PrevFilterGroup := Rec.FilterGroup();
        Rec.FilterGroup(4);
        ParentNoFilter := Rec.GetFilter("Parent No.");
        Rec.FilterGroup(PrevFilterGroup);

        Rec."Parent No." := CopyStr(ParentNoFilter, 1, MaxStrLen(Rec."Parent No."));
    end;
}
```

Three things are load-bearing and should survive adaptation:

1. **Save and restore the group.** `Rec.FilterGroup()` with no argument *gets* the current group; `Rec.FilterGroup(4)` *sets* it and returns the previous value. Leaving the record parked on group 4 means every subsequent `SetRange`/`SetFilter` in the same code path silently writes into the platform's link group — the exact thing the documentation warns about above. Restore it on every path out.
2. **Guard on blank first.** The platform's own propagation of `SubPageLink` into a new subform row usually works. This trigger is a fallback, and should be a no-op whenever the normal path succeeded — never an unconditional overwrite.
3. **`CopyStr` to the field's own `MaxStrLen`.** `GetFilter` returns `Text`, not the field's type. See Caveats for when `CopyStr` is the wrong tool.

**Typed alternative, often better** — when the link is a single value (which `SubPageLink = "Parent No." = field("No.")` always produces), `GetRangeMin` returns the value already converted to the field's own type, with no text handling and no truncation risk:

```al
    trigger OnNewRecord(BelowxRec: Boolean)
    var
        PrevFilterGroup: Integer;
    begin
        if Rec."Parent No." <> '' then
            exit;

        PrevFilterGroup := Rec.FilterGroup();
        Rec.FilterGroup(4);
        if Rec.GetFilter("Parent No.") <> '' then
            Rec."Parent No." := Rec.GetRangeMin("Parent No.");
        Rec.FilterGroup(PrevFilterGroup);
    end;
```

`GetRangeMin` errors if the filter is not a simple range/equality, so it fails loudly on an assumption violation rather than writing a wildcard string into a key field. That is usually what you want. Use the `CopyStr` form only where a non-trivial filter expression is genuinely possible and you intend to handle it.

### Part 2 — expose the link field on the subpage, hidden

```al
            repeater(Group)
            {
                field("Parent No."; Rec."Parent No.")
                {
                    ApplicationArea = All;
                    Visible = false;
                    ToolTip = 'Specifies the parent record this line belongs to.';
                }
                // ... the visible fields ...
            }
```

This matches the standard BC convention: subforms bound by `SubPageLink` generally expose their link field as a control even when the user never sees it, rather than relying purely on the record buffer. A field the page knows about is a field the page can propagate into; it also makes the value inspectable when debugging, which a purely-implicit link is not.

### Part 3 — make the failure loud at the table level

```al
    trigger OnInsert()
    begin
        Rec.TestField("Parent No.");
        // ... numbering, seeding, other OnInsert work ...
    end;
```

This is the part worth keeping permanently even after the subform is fixed. It is a table-level invariant — *a child of this table is never valid without a parent* — and it belongs on the table, where it holds for every caller: this subform, a different page added later, an API insert, a data-migration codeunit, a test. If the link is ever blank again for any reason, the insert fails immediately with a clear, actionable error naming the field, instead of writing an orphan row that nobody will find for weeks.

---

## Worked example from this project

**Project:** Bootcamp Registration Tracking (BC PTE, publisher OnlyCopilotFans, prefix `ocpf`).
**History:** `docs/ChangeLog.md` — **BUILD-12** (original fix, **superseded — it was a provable no-op**) → **BUILD-16** (corrected diagnosis and the real fix) → **BUILD-21** (confirmed fixed on a live sandbox retest).

### The report

From `docs/TestingFeedback.md`, 2026-09-12 session: creating a new Bootcamp, filling in its header fields, then adding an Attendee line via the embedded `ocpfAttendeeSubform` produced *"the view is filtered, and the entry is outside the filter."* The Attendee record was created anyway, with `"Bootcamp No."` blank — confirmed by AJ Ansari directly, not assumed.

The page part is an ordinary parent/child binding:

```al
part(Attendees; "ocpfAttendeeSubform")
{
    SubPageLink = "Bootcamp No." = field("No.");
}
```

### BUILD-12 — the first fix, which did nothing (retained, not deleted)

BUILD-12 added an `OnNewRecord` trigger to `ocpfAttendeeSubform` that read the link and assigned it when blank — the right *shape* of fix, reading the wrong filter group:

```al
// BUILD-12, superseded — this never assigned anything.
if Rec."Bootcamp No." = '' then
    Rec."Bootcamp No." := CopyStr(Rec.GetFilter("Bootcamp No."), 1, MaxStrLen(Rec."Bootcamp No."));
```

No group switch, so `GetFilter` read group 0, which had no filter on that field; it returned `''`; the assignment was blank-into-blank. The ChangeLog records it verbatim as: *"this fix was a provable no-op."*

It shipped, compiled clean (0 errors / 0 warnings), passed code review, and **changed nothing** — AJ retested the `0.0.3.0` build live and reported: *"I got the activity cues. But neither of the two issues were resolved."* That retest is what triggered the re-diagnosis. The ChangeLog deliberately retains BUILD-12 marked superseded rather than deleting it, on the project's standing principle that erasing the wrong turn erases the reason the right one was found.

### BUILD-16 — the real fix

Per the project's role-assignment rule, diagnosis was routed to a fresh-eyes reasoning-role review rather than patched again on a guess, and the resulting root cause was independently re-verified against Microsoft Learn's `Record.FilterGroup()` reference before being applied — not taken on the subagent's word. Three parts landed together.

**After —** `/Users/ajansari/Documents/AL/BootcampClaude/src/Attendee/ocpfAttendeeSubform.Page.al`:

```al
    trigger OnNewRecord(BelowxRec: Boolean)
    var
        PrevFilterGroup: Integer;
        BootcampNoFilter: Text;
    begin
        if Rec."Bootcamp No." <> '' then
            exit;
        // SubPageLink filters live in filter group 4 ("Link"), not the default group 0 —
        // a plain Rec.GetFilter() reads group 0 and misses it entirely.
        PrevFilterGroup := Rec.FilterGroup();
        Rec.FilterGroup(4);
        BootcampNoFilter := Rec.GetFilter("Bootcamp No.");
        Rec.FilterGroup(PrevFilterGroup);
        Rec."Bootcamp No." := CopyStr(BootcampNoFilter, 1, MaxStrLen(Rec."Bootcamp No."));
    end;
```

Same file, the hidden link control added as the first field in the repeater:

```al
                field("Bootcamp No."; Rec."Bootcamp No.")
                {
                    ApplicationArea = All;
                    Visible = false;
                    ToolTip = 'Specifies the bootcamp this person is registered for.';
                }
```

**After —** `/Users/ajansari/Documents/AL/BootcampClaude/src/Attendee/ocpfAttendee.Table.al`, first line of `OnInsert`:

```al
    trigger OnInsert()
    begin
        Rec.TestField("Bootcamp No.");
        if Rec."No." = '' then
            BootcampRegMgt.InitAttendeeNo(Rec);
        BootcampRegMgt.SeedAmountPaid(Rec);
        BootcampRegMgt.ConfirmOverbookingIfNeeded(Rec);
    end;
```

The `TestField` was added deliberately as a discriminating test, not just belt-and-braces: BUILD-16 could not fully rule out a *second* candidate cause (the parent header genuinely not being committed yet when the line was added), and if that were the real story, the `TestField` would fire on retest and point the next fix at the header page instead of the subform. It did not fire. BUILD-21 records the live retest result — *"Good news - everything tested well."* — closing BUILD-12 / BUILD-16 Bug 2.

Housekeeping worth copying: before retesting, existing orphan Attendee rows with blank `"Bootcamp No."` from earlier testing had to be deleted from the **unfiltered** Attendees list. They are invisible in every filtered view by construction, and would have muddied the result.

---

## Where else this shows up

The class is broader than subforms: **any time a child or related record is created inside a context that was implicitly filtered to a parent by the platform, and code needs to read that parent's identity back.**

- **Lines subforms of any header/line document pair** — the canonical case. Every `ListPart` with `SubPageLink` on a non-key or partially-key field.
- **Reports.** `DataItemLink` is in the *same* filter group 4 per the table quoted above. A `dataitem` that needs to know which parent row it is currently nested under reads it the same way.
- **`SubPageView` / `RunPageView` contexts** — same class of bug, **different group**: those are group **3** ("Exec"), not 4. Same fix shape, different number.
- **`SourceTableView` on a document-type-filtered page** — group **2** ("Form"). Common in API pages filtered to one document type, where code wants to know which type it is scoped to.
- **FactBoxes and drill-downs** that receive their context by link rather than by an explicit parameter.
- **Standard BC's own item/vendor catalog pages** — a good mental model for the general shape, and one I verified concretely against this project's downloaded BC v28.4 symbols rather than from memory:

  | Object | Verified from symbols |
  |---|---|
  | Table **99** `"Item Vendor"` | Namespace `Microsoft.Inventory.Item.Catalog`. Field 1 `"Item No."` (`TableRelation = Item`), field 2 `"Vendor No."` (`TableRelation = Vendor`). Clustered primary key: **`Vendor No.`, `Item No.`, `Variant Code`**. `LookupPageID = "Item Vendor Catalog"`. |
  | Page **114** `"Item Vendor Catalog"` | `PageType = List`, `SourceTable = 99`, `DelayedInsert = true`, `DataCaptionFields = 1` (Item No.). |
  | Page **297** `"Vendor Item Catalog"` | `PageType = List`, `SourceTable = 99`, `DataCaptionFields = 2` (Vendor No.). |

  The structural point, which is what generalizes: **one child table (99) is surfaced through two different pages, each scoped to a different parent** — page 114 in the context of an Item, page 297 in the context of a Vendor — and in *both* cases the field identifying the parent is **part of the child's primary key**. A row created in either view that failed to pick up its parent's identity would not merely be orphaned, it would collide on or corrupt the key. That is exactly the shape of the bug this pattern prevents, in a first-party module, at table-99 scale.

  **Stated precisely, because the verification has limits:** symbol files carry object and field metadata, not control-level page properties, so I could *not* read these pages' `SubPageLink` / `RunPageLink` values from the symbols, and I am not asserting which mechanism scopes them or which filter group it uses. The IDs, types, key, and `DelayedInsert` above are verified; the scoping mechanism is not. If you need that detail for your own case, read the actual page source or test it — do not take the analogy as a citation.

---

## Caveats — check these for your own case

- **Reading group 4 is safe; *writing* to it is what the docs warn about.** The verbatim warning quoted under Root cause is about *replacing* the platform's filter by calling `SetFilter`/`SetRange` while parked on group 4. This pattern only calls `GetFilter`/`GetRangeMin` and restores the group immediately. Keep it that way — if you find yourself setting filters in group 4, re-read that warning first.
- **Always restore the previous filter group, on every exit path.** An early `exit` or an error between `FilterGroup(4)` and the restore leaves the record parked on the link group. In the pattern above the blank-guard `exit` is placed *before* the group switch for exactly this reason. If your logic needs to exit mid-block, restore first.
- **`GetFilter` returns a filter *expression*, not a value.** For `SubPageLink = "Parent No." = field("No.")` it is a plain single value, which is why `CopyStr` is adequate. For a hand-written `SubPageLink` with a range, a `filter(...)` expression, or multiple values, the text can contain `..`, `|`, `*`, `<>`, or `&` — and shoving that into a key field produces garbage that `CopyStr` will happily truncate without complaint. Prefer `GetRangeMin` (which errors on a non-simple filter) unless you have a specific reason not to.
- **`CopyStr` silently truncates.** If the filter text is somehow longer than the target field, you get a wrong-but-plausible value rather than an error. This is acceptable for a single-value link where lengths match by construction; verify that assumption holds for your fields.
- **This is a fallback, not the primary mechanism.** The platform's own `SubPageLink` propagation normally works. If your link field is blank *always* rather than intermittently, check the simpler explanations first: is the link field actually spelled correctly in `SubPageLink`? Does the parent field have a value yet at the moment the line is added? Is the parent record committed? A blank-guard fallback masking a genuinely broken `SubPageLink` is worse than no fallback.
- **The parent-not-yet-committed theory is a real, separate cause** with the same symptom. That is precisely why part 3's `TestField` matters: it discriminates. If the `TestField` fires after you apply part 1, your problem is on the *parent* side (header timing / commit), not the subform, and the fix belongs on the header page. Do not assume part 1 was sufficient just because you applied it — retest and see whether the guard fires.
- **`DelayedInsert = true` interacts with this.** `OnNewRecord` fires when the placeholder row is created; the physical `Insert()` happens later. Verify your assignment actually survives to the insert in your own page, especially if you also use `AutoSplitKey` or `UpdatePropagation`.
- **The `TestField` guard has a blast radius.** It now applies to *every* insert path into that table — API endpoints, data migration, configuration packages, test codeunits, and any RapidStart/demo-data seeding. That is usually correct and desirable, but confirm no legitimate existing path inserts a deliberately-unlinked row before adding it. If one does, that path needs a decision, not a silent exemption.
- **Clean up existing orphans before retesting.** Rows already written with a blank link are invisible in every filtered view and will confuse your verification. Find them in an unfiltered list of the child table and deal with them explicitly.
- **Filter group numbers are platform constants, but read them from the docs, not from memory.** Group 4 is `SubPageLink`/`DataItemLink`; group 3 is `SubPageView`/`RunPageView`; group 2 is `SourceTableView`/`SetTableView`. `RunPageLink` is **not** named in the documented table at all — if your context is a page opened by `RunPageLink` rather than an embedded `SubPageLink` part, verify which group actually carries the filter for your case rather than assuming it is 4.

---

## Sources

- `Record.FilterGroup([Integer]) Method` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/record/record-filtergroup-method> (fetched and verified 2026-09-13; page `ms.date` 2024-08-26)
- BC v28.4 symbol files in this project's `.alpackages/` — table 99, pages 114 and 297 (read directly from `SymbolReference.json`)
- This project: `docs/ChangeLog.md` issues BUILD-12 (superseded), BUILD-16, BUILD-21; `docs/TestingFeedback.md` sessions 2026-09-12 and 2026-09-13
