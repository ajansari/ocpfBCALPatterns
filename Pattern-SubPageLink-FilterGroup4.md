# Pattern: Reading a parent link set by the platform — filter group 4 for `SubPageLink`, group 0 for `RunPageLink`

**A child record created inside its parent's page comes out with the link field blank, or "the view is filtered, and the entry is outside the filter" appears and the row vanishes — because code read the parent link from the wrong filter group, or from only one of the two groups the page can be opened with.**

When a child page is opened in a parent's context — a `ListPart` bound by `SubPageLink`, a list opened from an action with `RunPageLink`, a report `dataitem` bound by `DataItemLink` — the platform sets the filter that says "these children belong to that parent" in a **filter group that depends on how the page was opened**. `SubPageLink` and `DataItemLink` filters live in group **4** ("Link"). `RunPageLink` filters live in group **0** ("Std"). `Rec.GetFilter(...)` reads only the group the record is currently on, which defaults to 0. So a subform reading group 0 finds nothing, a list opened by an action reading group 4 finds nothing, and the same child page is routinely opened both ways. This pattern is the read that works in both entry points, plus the defensive measures that make the failure loud instead of silent.

---

## Applicability and version

A **Business Central AL platform pattern**, not tied to any extension, table, industry, or object naming. It applies to any AL object that reads a filter the platform set on its behalf.

**Version:** core, long-standing AL platform behavior — **no minimum BC version applies.** Microsoft Learn's `Record.FilterGroup([Integer])` reference states its availability as:

> **Version**: *Available or changed with runtime version 1.0.*

Runtime version 1.0 is the first AL runtime, so the filter-group numbering has been present for the entire life of the AL language. **No "designed for version XX+" cutoff should be claimed for it.**

**Confirmed on live sandboxes** on BC 27.x and 28.x tenants, in both directions: a subform reading group 0 and an action-opened list reading group 4 each produced the orphan; the two-group read below fixed both.

---

## Symptom

- A user opens a parent card, adds a row in an embedded lines subform (or opens a child list from an action on the card and adds a row there), fills it in, leaves the row, and sees **"The view is filtered, and the entry is outside the filter. Some actions may not work."** The row disappears.
- The child record **was inserted** — with the linking field (`"Parent No."`, `"Document No."`, `"Customer No."`, whatever it is) **blank**.
- The orphan row is invisible from every normal entry point, because every normal entry point is filtered to a parent. It shows only in an unfiltered list of the child table. Rows quietly accumulate.
- Any `FlowField` rollup on the parent (counts, sums, "remaining") is silently wrong, because orphans count against no parent.
- On a list without `DelayedInsert`, the row already shows its number-series number before the user has filled anything else — the record was written on the first keystroke, blank link included.
- **A guard that reads the link from the filter appears to fix it in code review and changes nothing at runtime.** `GetFilter` on a field with no filter in the *current* group legitimately returns `''`, and assigning `''` to a blank field is a valid no-op. No compile error, no warning, no exception.
- **The fix that worked on the subform breaks on the action-opened list, or the reverse.** That is the tell for the two-group problem specifically.

---

## Root cause

AL keeps several filter groups on a record at once. `SetFilter`, `SetRange`, and `GetFilter` all operate on the currently selected group, which defaults to 0. The platform puts the filters it sets on your behalf into reserved groups.

Verified against Microsoft Learn, `Record.FilterGroup([Integer]) Method` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/record/record-filtergroup-method> (fetched 2026-09-21). The documentation's own table, quoted verbatim for the rows that matter:

> | Number | Name | Description |
> | --- | --- | --- |
> | 0 | Std | The default group where filters are placed when no other group has been selected explicitly. This group is used for filters that can be set from the filter dialogs by the end user. |
> | 2 | Form | Used for the filtering actions that result from the following: - SetTableView Method (XMLport), SetTableView Method (Page) - SourceTableView Property - DataItemTableView Property. |
> | 3 | Exec | Used for the filtering actions that result from the following: - SubPageView Property - RunPageView Property |
> | 4 | Link | Used for the filtering actions that result from the following: - DataItemLink Property (Reports) - SubPageLink Property |

`RunPageLink` is not in that table. Its own property page settles where it goes — verbatim, from `RunPageLink Property` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/properties/devenv-runpagelink-property>:

> The filters defined by this property are visible in the UI and can be modified by end-users. If it was intended to hide them from end-users, consider using the RunPageView property instead.

"Visible in the UI and can be modified by end-users" is the definition of group 0 above. `RunPageView`, by contrast, states the opposite and is in group 3:

> The filters defined by this property are not visible in the UI and cannot be modified by end-users.

So the map is: **`SubPageLink` / `DataItemLink` → group 4. `RunPageLink` → group 0. `RunPageView` / `SubPageView` → group 3. `SourceTableView` / `SetTableView` → group 2.** Reading one group to find a link that may have been set through either mechanism is the bug.

The same page also documents that all groups apply at once — which is why the *view* is correctly filtered even though your *code* cannot see the filter:

> All groups are active at all times. The only way to turn off a group is to remove the filters set in that group.

And it warns about the difference between reading a reserved group and writing to it (see Caveats):

> It's possible to use one of the internally used groups. If you do this, you replace the filter that Dynamics 365 Business Central assumes is in this group. If, for example, you use filter group 4 in a page, you will replace the filtering that is actually the result of applying the SubPageLink Property. This could seriously alter the way pages and subpages interact.

---

## The pattern / fix

Four parts. Part 1 is the read. Parts 2–4 convert a silent orphan into a loud, actionable error and stop the half-filled insert.

### Part 1 — read the link from group 4, then group 0, only when blank, restoring the group on every path

```al
page 50101 "Child Lines Subform"
{
    PageType = ListPart;
    SourceTable = "Child Line";
    DelayedInsert = true;   // Part 4

    // ... layout, including the hidden link control from Part 2 ...

    trigger OnNewRecord(BelowxRec: Boolean)
    begin
        // Inert when the platform's own pre-fill already worked — only fires on blank.
        if Rec."Parent No." = '' then
            Rec.Validate("Parent No.", GetParentNoFromLink());
        // Seed other defaults here, field by field. Never Rec.Init() in this trigger:
        // Init() clears every non-key field, including the link just set.
    end;

    local procedure GetParentNoFromLink(): Code[20]
    var
        ParentNo: Code[20];
    begin
        if TryGetSingleValueFilter(4, ParentNo) then // SubPageLink, DataItemLink
            exit(ParentNo);
        if TryGetSingleValueFilter(0, ParentNo) then // RunPageLink, end-user filters
            exit(ParentNo);
        exit('');
    end;

    local procedure TryGetSingleValueFilter(FilterGroupNo: Integer; var ParentNo: Code[20]) Found: Boolean
    var
        PrevFilterGroup: Integer;
    begin
        PrevFilterGroup := Rec.FilterGroup();
        Rec.FilterGroup(FilterGroupNo);
        // Nested ifs on purpose: the second test must not run when there is no filter,
        // because GetRangeMin errors on a field with no filter.
        if Rec.GetFilter("Parent No.") <> '' then
            if Rec.GetRangeMin("Parent No.") = Rec.GetRangeMax("Parent No.") then begin
                ParentNo := Rec.GetRangeMin("Parent No.");
                Found := true;
            end;
        Rec.FilterGroup(PrevFilterGroup);
    end;
}
```

What is load-bearing and must survive adaptation:

1. **Both groups, 4 first.** A page used only as a part could read 4 alone; a page used only from actions could read 0 alone. Pages get reused, so read both — the cost is nothing, and the second read is what saves the entry point nobody tested.
2. **Save and restore the group.** `Rec.FilterGroup()` with no argument gets the current group; `Rec.FilterGroup(n)` sets it. A record left parked on group 4 means every later `SetRange` writes into the platform's link group — the documented warning above. The restore happens inside the helper, before any caller code runs, so no exit path can skip it.
3. **Guard on blank first.** The platform's own pre-fill of the link into a new row usually works. This trigger is a fallback and must never be an unconditional overwrite.
4. **`GetRangeMin`/`GetRangeMax`, not `GetFilter` text.** `GetFilter` returns a filter *expression* as `Text`; `GetRangeMin` returns the value in the field's own type, and the min = max check proves it is a single value. A `field()` link always produces one. Note what `GetRangeMin` does and does not do: per Microsoft Learn it errors *only* when the field has no filter at all (hence the `GetFilter <> ''` guard first); a hand-written range `A..B` returns `A` without erroring, which is exactly why the min = max comparison is required.
5. **`Validate`, not assignment,** so the link field's own `OnValidate` (defaults it pulls from the parent, related lookups) runs exactly as it would for a typed value.

### Part 2 — expose the link field on the child page, hidden

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

Standard BC subforms expose their link field as a control even when the user never sees it. A field the page knows about is a field the page can pre-fill; it is also the field a tester unhides with *Personalize* to see whether the value arrived.

### Part 3 — make the failure loud at the table level

```al
    trigger OnInsert()
    begin
        Rec.TestField("Parent No.");
        // ... numbering, seeding, other OnInsert work ...
    end;
```

Keep this permanently, after the page is fixed. It is a table invariant — *a child of this table is never valid without a parent* — and it holds for every caller: this page, a page added later, an API insert, a migration, a test codeunit. `NotBlank = true` on the field is **not** a substitute: it guards typed entry only and does nothing for a value assigned in code or left blank by the platform.

### Part 4 — `DelayedInsert = true` on the child list or list part

The row is then written when the user leaves it, with every field present, instead of on the first keystroke with the rest blank. This is the setting on every Microsoft document subform, and it is what stops the "numbered row with nothing else in it" symptom.

---

## Worked example — a minimal reproduction

A parent table `"Parent Header"` (`"No."`) and a child `"Child Line"` (`"Parent No."`, `"Line No."`). The child is shown twice: as a part on the project card, and from an action on the project list.

```al
// On the card
part(Lines; "Child Line Subform")
{
    ApplicationArea = All;
    SubPageLink = "Parent No." = field("No.");     // filter lands in group 4
}

// On the list
action(Lines)
{
    ApplicationArea = All;
    Caption = 'Lines';
    Image = List;
    RunObject = page "Child Line Subform";
    RunPageLink = "Parent No." = field("No.");     // filter lands in group 0
    RunPageView = sorting("Parent No.");          // required alongside RunPageLink for performance
}
```

**The first wrong fix** — the shape almost every first attempt takes — reads group 0 without switching:

```al
// Superseded — a provable no-op from the subform: GetFilter reads group 0, the link is in group 4.
if Rec."Parent No." = '' then
    Rec."Parent No." := CopyStr(Rec.GetFilter("Parent No."), 1, MaxStrLen(Rec."Parent No."));
```

It compiles clean, passes review, and changes nothing from the card. **The second wrong fix** switches to group 4 and nothing else — now the card works and the list-opened page orphans every row, because `RunPageLink` is in group 0. Both were shipped, on two builds of one project, before the two-group read replaced them. With Part 1 in `OnNewRecord`, Part 2 in the repeater, Part 3 in `"Child Line"`'s `OnInsert`, and `DelayedInsert = true`, a line created from either entry point stays visible with `"Parent No."` filled, and the unfiltered child list has no blank-link rows.

Housekeeping before retesting: delete existing orphan rows with a blank link from the **unfiltered** child list. They are invisible in every filtered view by construction and will muddy the result.

---

## Where else this shows up

Any time a child or related record is created inside a context the platform filtered to a parent, and code needs to read that parent's identity back.

- **Lines subforms of any header/line pair** — every `ListPart` with `SubPageLink` (group 4).
- **Child lists opened from an action** — `RunPageLink` (group 0). The same page as above, opened differently.
- **Reports** — `DataItemLink` is in group 4, per the documented table. A `dataitem` that needs to know which parent row it is nested under reads it the same way.
- **`SubPageView` / `RunPageView` contexts** — group **3**. Same read shape, different number; but a `where()` there takes only `const()` and `filter()` — never `field()` — so it is never how a parent identity arrives.
- **`SourceTableView` on a document-type-filtered page** — group **2**. Common in API pages filtered to one document type, where code wants to know which type it is scoped to.
- **FactBoxes and drill-downs** that receive their context by link rather than by an explicit parameter.
- **Standard BC's own many-to-one catalogs** — one child table surfaced through two pages, each scoped to a different parent — are the mental model for "the same child, two contexts".

---

## Caveats — check these for your own case

- **The documented warning is about replacing the platform's filter, not about reading it.** Learn says the internal groups "shouldn't be used" and then explains the harm as *replacing* the filter in that group; this pattern only calls `GetFilter`/`GetRangeMin`/`GetRangeMax` and restores the group immediately. Never `SetRange`/`SetFilter` while parked on 4.
- **Restore on every path.** The helper restores before it returns, and the blank guard sits before the call, so there is no exit between switch and restore. If you inline it, keep that property.
- **A `field()` link is a single value; a hand-written `filter()` is not.** `GetRangeMin` does **not** error on a range — it returns the lower bound — so the `GetRangeMin = GetRangeMax` test is the only thing rejecting `A..B` or `A|B`. Keep it.
- **This is a fallback, not the primary mechanism.** If the link is blank *always* rather than in one entry point, check the simpler explanations first: is the field name in `SubPageLink`/`RunPageLink` spelled correctly? Is the parent's key actually populated when the part renders? Is anything calling `Rec.Init()` after the pre-fill (see `Pattern-Init-Does-Not-Clear-Primary-Key.md`)? Does another field's `OnValidate` reset the link?
- **The parent-not-yet-committed theory is a separate cause** with the same symptom. Part 3 discriminates: if `TestField` fires *after* Part 1 is in place, the problem is on the parent side (header timing / commit), not the child page.
- **`DelayedInsert = true` changes when `Insert()` runs.** `OnNewRecord` fires when the placeholder row appears; the physical insert happens when the user leaves the row. Verify the assignment survives to the insert if you also use `AutoSplitKey` or `UpdatePropagation`.
- **The `TestField` guard has a blast radius.** It applies to every insert path — API, migration, configuration packages, test codeunits, demo data. Usually correct; confirm no legitimate path inserts a child without a parent.
- **Filter group numbers are platform constants, but quote them from the docs, not memory.** Group 4 is `SubPageLink`/`DataItemLink`; group 0 is `RunPageLink` and end-user filters; group 3 is `SubPageView`/`RunPageView`; group 2 is `SourceTableView`/`SetTableView`.

---

## Sources

- `Record.FilterGroup([Integer]) Method` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/record/record-filtergroup-method> (fetched and verified 2026-09-21)
- `RunPageLink Property` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/properties/devenv-runpagelink-property> (fetched 2026-09-21; page last updated 2024-10-01)
- `RunPageView Property` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/properties/devenv-runpageview-property> (fetched 2026-09-21)
- `Record.GetRangeMin Method` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/record/record-getrangemin-method> (fetched 2026-09-21; its one documented error: "An error is thrown if there is no filter on the specified field.")
- `SubPageLink Property` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/properties/devenv-subpagelink-property> (fetched 2026-09-21)
- Microsoft documentation quoted under © Microsoft Corporation, [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
