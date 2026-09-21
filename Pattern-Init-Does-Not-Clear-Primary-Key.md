# Pattern: `Record.Init()` does not clear the primary key — and it *does* clear everything else

**Two opposite failures from one method. Reusing a record variable across several inserts fails on the second with "already exists", because `Init()` left the primary key alone. Calling `Init()` on a record the platform already pre-filled wipes the pre-filled values, because `Init()` resets every non-key field.**

`Record.Init()` assigns default values to every field in the record **except the primary key and the timestamp**. Both halves of that sentence bite:

- **The half people forget:** in a loop, a wizard's sample data, an import — one variable, several `Insert()` calls, `Init()` between them — the key value from the previous insert survives, and the second insert presents the same key again. It looks exactly like a number-series bug and is not one.
- **The half people never think about:** a page's `OnNewRecord` (or a "new record" helper it calls) that starts with `Rec.Init()` erases whatever the platform pre-filled into the new row a moment earlier — the parent link from `SubPageLink`/`RunPageLink`, the values copied from single-value filters — and produces an orphan child (see `Pattern-SubPageLink-FilterGroup4.md`).

---

## Applicability and version

A **Business Central AL platform pattern**, not tied to any extension, table, or industry. It applies to any AL code that reuses a record variable across more than one `Insert()`, and to any page trigger that runs after the platform has initialized a new record.

**Version:** core, long-standing AL platform behavior — **no minimum BC version applies.** Microsoft Learn's `Record.Init()` reference states its availability as:

> **Version**: *Available or changed with runtime version 1.0.*

This `Init()` semantic has been present for the entire life of the language (and is inherited from C/AL). **No "designed for version XX+" cutoff should be claimed for it.**

---

## Symptom

### Failure A — the reused variable

- A wizard, "create sample data" routine, import, or migration loop errors with **"a record already exists"** — or, with the table's own caption, *"<Caption> already exists"*.
- The error **names the number series' first available number** — the single most misleading detail in the bug. It reads unmistakably as a numbering problem.
- Afterwards, **the table is empty.** Not "one row" — empty. The unhandled error rolled back the whole transaction, including the first, successful insert.
- **Resetting or recreating the number series does not help.** A brand-new, never-used series reproduces the identical error on its first use.
- The first record always succeeds; the **second and subsequent** inserts fail. A routine that creates exactly one record never shows the bug.

The identifying combination: **empty table + never-used series + error naming that series' first number.** No stale data or series mis-state can produce that. Your code presented the same key twice.

### Failure B — the pre-filled record wiped

- A child row created from a parent's page lands "outside the filter" with its link blank, *even though* the page has a correct filter-group read, or the platform's pre-fill demonstrably worked a moment earlier.
- Default values that should have arrived from a single-value filter (a document type, a location code) are blank on the new row.
- The `OnNewRecord` trigger, or the codeunit procedure it calls, begins with `Rec.Init()` — often copied from a table-level "new record" helper where it was harmless.

---

## Root cause

Verified against Microsoft Learn, `Record.Init() Method` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/record/record-init-method> (fetched 2026-09-21). Quoted verbatim:

> This method assigns default values to each field in the record, including the SystemId field when a table is created. For any new field added later into the record, values are initialized by default or by using InitValue Property (Record).

> **Note** — Primary key and timestamp fields aren't initialized.

> After the method runs, you can change the values in any or all of the fields before you call the Insert method (RecordRef) to enter the record in the table. Be sure that the fields that make up the primary key contain values that make the total primary key unique. If the primary key isn't unique (such as the record already exists), then the record is rejected.

The documentation also — verbatim — names the loop case as exactly what `Init()` is for, which is why the trap is so easy to fall into:

> You aren't required to call the Init() method every time you intend to insert a record as, the fields are already populated, either with default values or the values set by the InitValue property. For the use cases, where the values need to be refreshed with each iteration in a loop or if they're inserted through a parameter, you should use the Init() method to make sure that the record aligns with the other data in the table.

### Failure A, step by step

Given the standard numbering guard in a table's `OnInsert`:

```al
    trigger OnInsert()
    begin
        if Rec."No." = '' then
            Rec."No." := NoSeries.GetNextNo(Setup."<Entity> Nos.");
    end;
```

and a routine that reuses one variable:

1. `MyRec.Init()` — fresh buffer, `"No." = ''`.
2. `MyRec.Insert(true)` — `OnInsert` sees `"No." = ''`, assigns `'X-00001'`. Row inserted. **The buffer now holds `'X-00001'`.**
3. `MyRec.Init()` — clears every ordinary field, **but not `"No."`.** It is the primary key.
4. `MyRec.Insert(true)` — `OnInsert` sees `"No." = 'X-00001'`, not blank, so **numbering is never reached.**
5. The platform tries to insert key `'X-00001'` a second time → **"already exists."** The error rolls back step 2 as well.

### Failure B, step by step

1. The user adds a row in a filtered child page. The platform creates the buffer and pre-fills `"Parent No."` from the link filter.
2. `OnNewRecord` runs and calls `Rec.Init()` — **every non-key field returns to its default**, `"Parent No."` included.
3. The trigger seeds a date and a status and returns. The link is blank.
4. The row is inserted outside the filter and disappears.

---

## The pattern / fix

### For loops and multi-insert routines — `Clear()` or a fresh variable, never `Init()` alone

```al
    procedure CreateSampleRecords(Count: Integer)
    var
        Entity: Record "<Prefix> Entity";
        i: Integer;
    begin
        for i := 1 to Count do begin
            Clear(Entity);               // resets the whole variable: primary key, filters, everything
            Entity.Init();               // then defaults, exactly as the docs intend
            Entity.Description := StrSubstNo(SampleDescLbl, i);
            Entity.Insert(true);         // OnInsert sees "No." = '' every time and numbers it
        end;
    end;
```

`Clear(Record)` resets the variable itself — primary key fields, filters, marks, the current key. `Init()` after it is still right, so `InitValue` defaults apply. Alternatives that also work: a variable declared inside a local procedure called once per record (a fresh variable per call), or explicitly assigning every primary-key field before each insert. What does not work is `Init()` alone.

### For page triggers — never `Init()` a record the platform initialized

```al
    trigger OnNewRecord(BelowxRec: Boolean)
    begin
        // The platform already ran Init and pre-filled from the link filter. Do not Init again.
        if Rec."Parent No." = '' then
            Rec.Validate("Parent No.", GetParentNoFromLink());   // see Pattern-SubPageLink-FilterGroup4.md
        if Rec."Document Date" = 0D then
            Rec."Document Date" := WorkDate();
    end;
```

Seed field by field, each guarded on blank. If a shared codeunit procedure is used for "new record defaults" from both a page and code, make it seed without `Init()`, and let the code path call `Clear` + `Init` itself before calling it.

### Discriminating test

Add `Rec.TestField("Parent No.")` (or the relevant key part) as the first line of the table's `OnInsert`. In Failure A it stays silent (the key is *present*, just duplicated) and the "already exists" error names the duplicate. In Failure B it fires and names the wiped field. Either way the error now says what is actually wrong.

---

## Worked example — a minimal reproduction

An assisted setup wizard's Finish action creates three sample records of `"<Prefix> Entity"`, whose `OnInsert` numbers a blank `"No."` from a new series with starting number `X-00001`:

```al
// Before — fails on the second insert with '<Entity> X-00001 already exists'; table empty afterwards
Entity.Init(); Entity.Description := 'Sample 1'; Entity.Insert(true);
Entity.Init(); Entity.Description := 'Sample 2'; Entity.Insert(true);   // "No." still = 'X-00001'
Entity.Init(); Entity.Description := 'Sample 3'; Entity.Insert(true);
```

```al
// After — three records, X-00001 to X-00003
Clear(Entity); Entity.Init(); Entity.Description := 'Sample 1'; Entity.Insert(true);
Clear(Entity); Entity.Init(); Entity.Description := 'Sample 2'; Entity.Insert(true);
Clear(Entity); Entity.Init(); Entity.Description := 'Sample 3'; Entity.Insert(true);
```

Recreating the number series, resetting *Last No. Used*, and deleting all rows change nothing in the "before" version — which is how the routine is recognized as this pattern and not a series problem.

---

## Where else this shows up

- **Assisted setup wizards** that create a default record or sample data on Finish — the most common site, because the wizard is also where the number series was just created, so everyone looks at the series.
- **Data migration / import codeunits** that loop over a source and insert into one record variable.
- **Test codeunits** creating several fixtures with one variable — the second fixture fails, and the test reports a numbering error.
- **Copy / duplicate actions** that `Init()` the target after `TransferFields` from the source — `TransferFields` copies the key; `Init()` does not clear it.
- **Page `OnNewRecord` triggers and "new record" helpers** — Failure B. Especially child pages, where the wiped field is the parent link.

---

## Caveats — check these for your own case

- **`Clear(Record)` also clears filters and marks.** In a loop that relies on a filtered range being iterated on the same variable, use a separate variable for the inserts rather than `Clear` on the iterator.
- **`Init()` is still correct and still wanted** after `Clear`, or on a genuinely fresh variable — it applies `InitValue` defaults. This pattern removes `Init()` only where the platform already ran it (page triggers).
- **A table whose primary key is assigned in `OnInsert` hides Failure A until the second insert.** A table whose key is assigned before `Insert` (explicit codes) shows it immediately as a duplicate-key error with the *typed* code — easier to recognize.
- **`SystemId` is regenerated on insert**, so the reused variable never fails on it; the failure is always on the business primary key.
- **The transaction rollback hides evidence.** The first insert did succeed; you will not see it. Reason from the error text and the emptiness of the table, not from what is in it.

---

## Sources

- `Record.Init() Method` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/record/record-init-method> (fetched and verified 2026-09-21)
- `Clear Method` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/system/system-clear-joker-method>
- Microsoft documentation quoted under © Microsoft Corporation, [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
- Companion: `Pattern-SubPageLink-FilterGroup4.md` (Failure B is one of that pattern's caveats made explicit).
