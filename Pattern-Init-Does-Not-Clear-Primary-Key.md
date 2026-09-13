# Pattern: `Record.Init()` does not clear the primary key — reusing one record variable for multiple inserts

**A wizard, demo-data routine, or import loop inserts the first record fine and then fails on the second with "a record already exists" — and it looks exactly like a No. Series bug, but it isn't one.**

`Record.Init()` resets a record's fields to their default values — *except* the primary key and the timestamp, which it explicitly leaves alone. So when one record variable is reused across several `Insert()` calls, `Init()` between them does **not** clear whatever key value the previous insert already assigned. The near-universal BC numbering guard — `if Rec."No." = '' then AssignNextNo(Rec);` in `OnInsert` — then sees a non-blank `"No."`, skips renumbering entirely, and the platform attempts to insert the same primary key a second time. The resulting duplicate-key error names the number series' own number, so every instinct points at the number series. The number series is fine. This pattern is the fix, and the diagnostic path to recognising it quickly.

---

## Applicability and version

This is a **Business Central AL platform pattern**, not tied to any specific extension, table, or vertical. It applies to any AL code that reuses a record variable across more than one `Insert()`.

**Version:** core, long-standing AL platform behavior — **no minimum BC version applies.** Microsoft Learn's `Record.Init()` reference states its availability as:

> **Version**: *Available or changed with runtime version 1.0.*

Runtime version 1.0 is the first AL runtime, so this `Init()` semantic has been present for the entire life of the language (and is inherited from C/AL before it). It was not introduced at, or changed by, any later BC release, and **no "designed for version XX+" cutoff should be claimed for it.**

**Verified/tested against, in the project this was extracted from:** BC v28.4 symbol files (Base Application `28.4.53241.54183`, System Application `28.4.53241.54101`, Business Foundation `28.4.53241.53312`, Application `28.4.53241.53312`, System `28.0.53984.0`), AL Runtime `17.0`, BC Application minimum `28.0.0.0`, recommended `28.4+`. Confirmed working on a live BC sandbox.

---

## Symptom

What a developer actually sees, in concrete terms:

- An Assisted Setup Wizard (or any "create sample data" / "seed demo records" routine) runs, and errors with **"a record already exists"** — or, with the table's own caption, *"Bootcamp already exists"*, *"Sales Header already exists"*.
- The error **names the number series' first available number** — `B10001`, `SO-00001` — which is the single most misleading detail in the whole bug. It reads unmistakably as a numbering problem.
- Afterwards, **the table is empty.** Not "contains one row" — empty. This is because the unhandled error rolled back the entire transaction, undoing the first, genuinely-successful insert along with the failed second one.
- **Resetting or recreating the number series does not help.** Creating a brand-new series, with a fresh Code, Starting No. and Ending No., never used before, reproduces the identical error on its very first-ever use.
- The first record always succeeds. It is always the **second and subsequent** inserts that fail. A routine that creates exactly one record never shows the bug at all.
- Nothing is wrong at compile time. This is clean, plausible, review-passing AL.

The combination that identifies it: **empty table + brand-new never-used series + error naming that series' first number.** No amount of stale data or series mis-state can produce that. Something in your code is presenting the same key twice.

---

## Root cause

`Init()` is not a full reset. It assigns default values to fields — and deliberately skips the two categories of field the platform considers not-yours-to-default.

Verified against Microsoft Learn, `Record.Init() Method` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/record/record-init-method>

Quoted verbatim, the note that is the entire root cause:

> Note
>
> Primary key and timestamp fields aren't initialized.

And, verbatim, the surrounding context — note that the documentation *explicitly* makes uniqueness of the key the caller's responsibility, and states the consequence of getting it wrong:

> This method assigns default values to each field in the record, including the SystemId field when a table is created. For any new field added later into the record, values are initialized by default or by using [InitValue Property (Record)](../../properties/devenv-initvalue-property).

> After the method runs, you can change the values in any or all of the fields before you call the [Insert method (RecordRef)](../recordref/recordref-insert--method) to enter the record in the table. Be sure that the fields that make up the primary key contain values that make the total primary key unique. If the primary key isn't unique (such as the record already exists), then the record is rejected.

The documentation also — verbatim — flags the reused-variable-in-a-loop case as precisely the scenario `Init()` exists for, which is exactly why the trap is so easy to fall into: developers reach for `Init()` in loops *because the docs tell them to*, and reasonably assume it clears everything:

> You aren't required to call the `Init()` method every time you intend to insert a record as, the fields are already populated, either with default values or the values set by the `InitValue` property. For the use cases, where the values need to be refreshed with each iteration in a loop or if they're inserted through a parameter, you should use the `Init()` method to make sure that the record aligns with the other data in the table.

### The failure, step by step

Given the standard BC numbering guard in the table's `OnInsert`:

```al
    trigger OnInsert()
    begin
        if Rec."No." = '' then
            MyMgt.InitNo(Rec);          // assigns NoSeries.GetNextNo(...)
    end;
```

and a routine that reuses one variable:

1. `MyRec.Init()` — fresh buffer, `"No." = ''`.
2. `MyRec.Insert(true)` — `OnInsert` sees `"No." = ''`, calls `InitNo`, assigns `'B10001'`. Row inserted. **The buffer now holds `'B10001'`.**
3. `MyRec.Init()` — clears Description, Date, Price, every ordinary field… **but not `"No."`.** It's the primary key. The buffer still holds `'B10001'`.
4. `MyRec.Insert(true)` — `OnInsert` sees `"No." = 'B10001'`, which is not blank, so **`InitNo` is never called at all.** The numbering code is not buggy; it is not *reached*.
5. The platform attempts to insert primary key `'B10001'` a second time → **"already exists."**
6. The unhandled error rolls back the whole transaction, undoing step 2's successful insert → the table is empty when you go looking.

Step 4 is the crux and the reason for every wrong diagnosis: **the number series is never consulted a second time.** `GetNextNo` is not returning a duplicate; it is not being called.

### A note on how this masquerades — worth reading before you diagnose your own

In the project this was extracted from, **the first two diagnoses were both wrong**, and the ChangeLog retains both marked superseded rather than deleting them:

- **First wrong diagnosis (BUILD-11): "stale data / environment, not a code defect."** The reasoning was sound in isolation — resetting a No. Series doesn't un-claim numbers already used by existing rows, so a leftover record at the series' starting number would produce exactly this error, and that's standard BC behavior, not an app bug. It was closed as an environment issue with no code change. It was disproved by two facts the theory could not survive: the table was verified **empty**, and a **brand-new** series reproduced it on first use.
- **Second wrong diagnosis (BUILD-13): "`GetNextNo` is returning the same number twice."** This one went further — it read the actual BC v28.4 `"No. Series - Stateless Impl."` (codeunit 306) source, found no defect there, and *still* concluded the mechanism must be duplicate numbers, because an empty table plus that error seemed to leave no alternative. The "fix" was a self-healing retry loop in the numbering procedure: keep requesting the next number until one is free. It compiled clean, shipped, and **changed nothing**, because the loop lived inside a procedure the bug never reached (step 4 above). It was later **reverted** — see "Do not do this" below.
- **Third, correct diagnosis (BUILD-16)**, found only after a live retest came back *"neither of the two issues were resolved"* and the problem was handed to a fresh-eyes review instead of being patched again on a guess. Root cause verified against the Microsoft Learn `Init()` documentation quoted above before any code was touched.

The transferable lesson: **an "already exists" error naming a number-series number is not evidence of a number-series problem.** It is evidence that *something* presented a duplicate key. Rule out the caller before you go anywhere near the series — and when a plausible theory survives only because you can't think of an alternative, that is a signal to verify a platform assumption against the documentation, not to write a defensive workaround around a mechanism you haven't confirmed.

---

## The pattern / fix

Two valid fixes. **Fix B is the recommended general pattern**; Fix A is the correct minimal-diff choice when refactoring isn't proportionate.

### Fix A — explicitly clear the primary key after each `Init()`

```al
local procedure CreateSampleRecords()
var
    MyRec: Record "My Table";
begin
    MyRec.Init();
    MyRec."No." := '';                  // Init() does NOT clear the primary key — do it explicitly.
    MyRec.Description := SampleOneTxt;
    MyRec."Some Amount" := 1500;
    MyRec.Insert(true);

    MyRec.Init();
    MyRec."No." := '';                  // Same again — required on every reuse, not just the first.
    MyRec.Description := SampleTwoTxt;
    MyRec."Some Amount" := 1500;
    MyRec.Insert(true);
end;
```

Correct, minimal, and obvious in a diff. Its weakness is that it is a *per-field* remedy for a *per-variable* problem: it fixes `"No."` and leaves every other piece of stale state on that reused buffer untouched — including any field added to the table next year by someone who never reads this procedure. The comment is not decoration; without it the line looks redundant and a future cleanup will delete it.

For a composite primary key, **every** key field must be cleared, not just the numbered one:

```al
    MyRec.Init();
    MyRec."Document No." := '';
    MyRec."Line No." := 0;              // all key fields, each to its own type's blank value
```

### Fix B — use a fresh record variable per insert (recommended)

Push the insert into its own helper so the variable's lifetime is exactly one record:

```al
local procedure CreateSampleRecords()
begin
    CreateOneSample(SampleOneTxt, CalcDate('<+30D>', Today()));
    CreateOneSample(SampleTwoTxt, CalcDate('<+60D>', Today()));
end;

local procedure CreateOneSample(NewDescription: Text[100]; NewDate: Date)
var
    MyRec: Record "My Table";           // fresh, zero-initialised on every call — no carry-over possible
begin
    MyRec.Init();
    MyRec.Description := NewDescription;
    MyRec."Target Date" := NewDate;
    MyRec.Insert(true);
end;
```

For a loop, the same idea — scope the variable to the iteration by scoping the procedure:

```al
local procedure ImportRows(var SourceBuffer: Record "My Source Buffer")
begin
    if SourceBuffer.FindSet() then
        repeat
            InsertOneRow(SourceBuffer);     // fresh target variable inside
        until SourceBuffer.Next() = 0;
end;
```

**Why this is the better general recommendation:**

1. **It removes the bug class, not the bug.** The defect is "stale state on a reused buffer." The primary key is merely the first symptom, because it's the one the platform enforces. Any field you forget to reset — and any field added to the table *later*, by someone who never sees this code — leaks silently from row *n* into row *n+1* with no error at all. That variant is strictly worse than the duplicate-key crash, because nothing fails; you just get quietly wrong data. Fix A leaves that door open; Fix B closes it structurally.
2. **It cannot be regressed by a later edit.** Fix A depends on a line that looks redundant surviving every future cleanup, refactor, and reviewer who doesn't know this pattern. A variable whose scope is one insert has no such dependency — correctness is enforced by the language, not by a comment.
3. **It costs nothing.** A local record variable is cheap; there is no performance argument for reusing one across inserts.
4. **It reads better.** "Create one of these" is a named, testable, reusable unit. Adding a third sample record becomes one call, not a copy-pasted block with a key-clearing line to remember.

Use Fix A when the surrounding procedure genuinely can't be restructured proportionately — a long existing routine, a hotfix on a release branch, a change you want to keep reviewable in two lines. That is exactly the tradeoff the worked example below made, and it is a legitimate choice; just add the comment.

### Do not do this — the retry-loop anti-fix

```al
// ANTI-PATTERN — this was BUILD-13 in the source project. It was reverted.
CandidateNo := NoSeries.GetNextNo(Setup."My Nos.");
while ExistingRec.Get(CandidateNo) do
    CandidateNo := NoSeries.GetNextNo(Setup."My Nos.");
MyRec."No." := CandidateNo;
```

It is superficially attractive: inert in the normal case, self-healing when a number is taken, terminates naturally on an exhausted series. It is still wrong, for reasons worth internalising:

- **It doesn't fix this bug.** It lives inside the numbering procedure, and the whole point of the root cause is that the numbering procedure **is never called** on the colliding insert. Zero effect.
- **It papers over a mechanism nobody has confirmed.** It was written precisely because the real cause hadn't been found — a workaround standing in for a diagnosis. If it *had* appeared to work, it would have hidden the real defect instead of fixing it.
- **It burns number-series numbers** on every collision, permanently, creating gaps that are an audit problem in numbering-sensitive domains.
- **It's dead code** the moment the real cause is fixed, and most code standards (including this project's) forbid carrying unnecessary code.

When the correct fix landed, this loop was reverted and the numbering procedures returned to a plain assignment. **Reverting a wrong fix is part of applying the right one.** A codebase that accumulates one workaround per wrong diagnosis becomes impossible to reason about.

### Diagnostic checklist — is this your bug?

Before touching the number series, in order:

1. **Does the routine insert more than one record from one variable?** If no, this pattern doesn't apply. If yes, it is the prime suspect.
2. **Does the first insert succeed and a later one fail?** Classic signature.
3. **Is the table empty afterwards?** That's the transaction rollback, and it rules out stale-data theories.
4. **Does a brand-new, never-used series reproduce it?** If yes, it is categorically not the series.
5. **Is the numbering procedure even being called on the failing insert?** Confirm it — a temporary `Message` or a breakpoint. If it isn't, stop looking at numbering code entirely; you have found the bug.

---

## Worked example from this project

**Project:** Bootcamp Registration Tracking (BC PTE, publisher OnlyCopilotFans, prefix `ocpf`).
**History:** `docs/ChangeLog.md` — **BUILD-11** (wrong diagnosis: stale data, **superseded**) → **BUILD-13** (wrong fix: self-healing retry loop, **superseded and reverted**) → **BUILD-16** (real root cause and fix) → **BUILD-21** (confirmed fixed on a live sandbox retest).

### The report

From `docs/TestingFeedback.md`, 2026-09-12 session: the Assisted Setup Wizard's sample-bootcamp creation step errored *"a record already exists"* at the Bootcamp No. series' first available number (`B10001`). AJ Ansari then established the two facts that broke both early theories: the `ocpfBootcamp` table was verified **empty**, and a **brand-new** No. Series with fresh Code / Starting No. / Ending No. reproduced the identical error on its very first-ever use.

### Before — the defect

`/Users/ajansari/Documents/AL/BootcampClaude/src/Setup/ocpfBootcampRegSetupWizard.Page.al`, `CreateSampleBootcamps` reused one `Bootcamp` variable for both sample records:

```al
// BEFORE — fails on the second Insert with "Bootcamp already exists".
local procedure CreateSampleBootcamps()
var
    Bootcamp: Record "ocpfBootcamp";
begin
    Bootcamp.Init();
    Bootcamp."Topic" := SampleTopic1Txt;
    // ... other fields ...
    Bootcamp.Insert(true);          // OnInsert assigns "No." := 'B10001'

    Bootcamp.Init();                // clears Topic/Price/etc. — NOT "No."
    Bootcamp."Topic" := SampleTopic2Txt;
    // ... other fields ...
    Bootcamp.Insert(true);          // "No." is 'B10001', guard skips numbering → duplicate key
end;
```

against this guard in `ocpfBootcamp`'s `OnInsert`:

```al
    if Rec."No." = '' then
        BootcampRegMgt.InitBootcampNo(Rec);
```

### After — the fix as actually applied (Fix A)

`/Users/ajansari/Documents/AL/BootcampClaude/src/Setup/ocpfBootcampRegSetupWizard.Page.al`:

```al
local procedure CreateSampleBootcamps()
var
    Bootcamp: Record "ocpfBootcamp";
begin
    Bootcamp.Init();
    Bootcamp."No." := '';
    Bootcamp."Topic" := SampleTopic1Txt;
    Bootcamp."Bootcamp Date" := CalcDate('<+30D>', Today());
    Bootcamp."Price" := 1500;
    Bootcamp."Max Seats" := 20;
    Bootcamp."Min Seats" := 6;
    Bootcamp.Insert(true);

    Bootcamp.Init();
    Bootcamp."No." := '';
    Bootcamp."Topic" := SampleTopic2Txt;
    Bootcamp."Bootcamp Date" := CalcDate('<+60D>', Today());
    Bootcamp."Price" := 1500;
    Bootcamp."Max Seats" := 20;
    Bootcamp."Min Seats" := 6;
    Bootcamp.Insert(true);
end;
```

Two lines. Fix A, not Fix B — a deliberate minimal-diff choice on a routine that creates exactly two fixed sample records and is unlikely to grow. **If you are writing this from scratch rather than patching it, prefer Fix B.**

### And the revert — `/Users/ajansari/Documents/AL/BootcampClaude/src/Bootcamp/ocpfBootcampRegMgt.Codeunit.al`

BUILD-13's self-healing retry loop was removed (AJ Ansari, 2026-09-12) as part of the same change, on the grounds that it solved a problem that never existed via this path and the project's standards forbid unnecessary code. The numbering procedures are back to a plain assignment, and that is what the pattern above recommends:

```al
    procedure InitBootcampNo(var Bootcamp: Record "ocpfBootcamp")
    begin
        GetSetup();
        Setup.TestField("Bootcamp Nos.");
        Bootcamp."No. Series" := Setup."Bootcamp Nos.";
        Bootcamp."No." := NoSeries.GetNextNo(Setup."Bootcamp Nos.");
    end;
```

`InitAttendeeNo` is identical in shape. Both are plain, loop-free, and correct.

BUILD-21 records the live retest: AJ published `0.0.4.0` to a sandbox and reported *"Good news - everything tested well."* — closing BUILD-11 / BUILD-13 / BUILD-16 Bug 1.

---

## When to use this / where it applies

Any AL code path that calls `Insert()` more than once against the same record variable. Concretely:

- **Assisted Setup Wizards** that create sample, demo, or default records — the case that produced this. Wizards are especially prone because they run once, on a fresh company, under a transaction that rolls back on error, which destroys the evidence.
- **Install and upgrade codeunits** seeding default setup rows, categories, or reference data.
- **Demo / test data generators**, including test codeunit `Initialize` helpers.
- **Import and conversion routines** — `repeat ... until` loops reading a source buffer and inserting into a target table from one variable declared outside the loop.
- **"Copy document" / "duplicate record" features** that read a source and insert a modified copy, especially when copying several lines.
- **Anything with a numbering guard of the shape `if Rec."No." = '' then …`** — which is essentially every BC table with a No. Series, including every standard one. The guard is correct; it simply assumes the caller hands it a blank key on a new record.
- **Composite-key child tables** (document lines, ledger-style detail rows) where a `"Line No."` is computed rather than blank — same mechanism, and `Init()` won't reset that either.
- **By extension, any field carrying state across a reused buffer**, not just the key. The key is the loud case; everything else fails quietly.

---

## Caveats — check these for your own case

- **Clear *every* primary key field, not just the obvious one.** Composite keys need each field reset to its own blank (`''` for Code/Text, `0` for Integer, `0D` for Date). Missing one reintroduces the bug in a subtler form.
- **`Clear(MyRec)` is a third option, with a real side effect.** It resets the record *including* the primary key, but it also **removes filters, marks, and the current key setting** on that variable. If the variable carries filters you still need — very common in a loop — `Clear` will break the loop in a way that looks unrelated. Use it knowingly, not as a bigger hammer.
- **`Init()` *does* initialise `SystemId`** — "including the SystemId field when a table is created," per the verbatim quote above. `SystemId` is not the trap here; the user-defined primary key is. Don't conflate them.
- **`Insert(true)` vs `Insert(false)` matters to this bug.** The whole mechanism runs through the `OnInsert` trigger's numbering guard, which only fires on `Insert(true)`. With `Insert(false)` the trigger never runs, you are responsible for the key yourself, and the same duplicate appears even more directly. Check which one your code calls.
- **The transaction rollback destroys your evidence.** Because the failure rolls back the whole transaction, the successful first insert vanishes too — which is exactly what made the stale-data theory look plausible and then impossible. Don't infer "nothing was created" from an empty table after a failed run.
- **Verify the numbering procedure is actually being called** before you change anything inside it. This is the single check that would have saved two wrong fixes. A temporary `Message` or a breakpoint settles it in one run.
- **If you inherited a retry loop or similar workaround, revert it** when you fix the real cause. Leaving it costs number-series gaps and leaves the next reader unable to tell which code is load-bearing.
- **Watch for the silent variant.** If your table's primary key happens to be assigned some other way (user-entered, or genuinely unique per iteration), this bug does **not** raise an error — it just carries stale values in *non-key* fields from one row to the next. Same root cause, no crash, wrong data. Fix B prevents both; Fix A prevents only the loud one.
- **Don't generalise the fix into the numbering codeunit.** The correct fix belongs at the call site that reuses the variable. Defensive logic inside `InitNo`/`GetNextNo` wrappers is how the superseded BUILD-13 loop happened.

---

## Sources

- `Record.Init() Method` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/record/record-init-method> (fetched and verified 2026-09-13; page `ms.date` 2024-08-26)
- This project: `docs/ChangeLog.md` issues BUILD-11 (superseded), BUILD-13 (superseded and reverted), BUILD-16, BUILD-21; `docs/TestingFeedback.md` sessions 2026-09-12 and 2026-09-13
