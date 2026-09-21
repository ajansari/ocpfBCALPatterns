# Pattern: A number-series field gets its lookup from `TableRelation`, and numbering follows Business Foundation

**The number-series field on a setup card or assisted setup wizard opens no lookup, opens the wrong page, or the picked value never reaches the setup record — and the extension cannot number its first record.**

Table 308 `"No. Series"` declares its own lookup page. A setup-table field of type `Code[20]` with `TableRelation = "No. Series"` therefore has a dropdown, a lookup page, and validation with no further code. The defect appears when the field on the *page* is bound to something that does not carry that relation — a page variable — or when the lookup is reimplemented by hand and the result is never written back. The numbering side has its own trap: code copied from pre-24 examples uses the obsolete `NoSeriesManagement` codeunit instead of Business Foundation's `"No. Series"`.

---

## Applicability and version

A **Business Central AL platform pattern**, not tied to any extension, table, or industry.

**Version:** the `TableRelation` half is core AL behavior with no minimum version. The numbering half targets the **Business Foundation** `"No. Series"` module, which is the supported API from **BC 24** on; the object IDs, namespace, fields, and public procedures below were read from the Business Foundation **27.5.46862.46929** symbol file, not from memory:

| Object | Verified from symbols |
|---|---|
| Table **308** `"No. Series"` | Namespace `Microsoft.Foundation.NoSeries`. `LookupPageId = "No. Series"`, `DrillDownPageId = "No. Series"`. Fields 1 `Code`, 2 `Description`, 3 `"Default Nos."`, 4 `"Manual Nos."`, 5 `"Date Order"`. |
| Table **309** `"No. Series Line"` | Fields 1 `"Series Code"`, 2 `"Line No."`, 3 `"Starting Date"`, 4 `"Starting No."`, 5 `"Ending No."`, 6 `"Warning No."`, 7 `"Increment-by No."`, 8 `"Last No. Used"`, 9 `Open`, 14 `Implementation` (enum). |
| Codeunit **310** `"No. Series"` (public) | `GetNextNo(NoSeriesCode)`, `PeekNextNo(NoSeriesCode)`, `GetLastNoUsed`, `TestManual`, `IsManual`, `TestAutomatic`, `AreRelated(Default, Related)`, `TestAreRelated`, `HasRelatedSeries`, `LookupRelatedNoSeries(Original, var New)`, `LookupRelatedNoSeries(Original, DefaultHighlighted, var New)`, `DrillDown`, `GetNoSeriesLine`, `MayProduceGaps`, `IsNoSeriesInDateOrder`. |
| Codeunit **308** `"No. Series - Batch"` (public) | `GetNextNo`, `PeekNextNo`, `GetLastNoUsed`, `TestManual`, `SimulateGetNextNo`, `SetSimulationMode`, `SaveState`. |
| Codeunit **299** `"No. Series - Setup"` (public) | `CalculateOpen`, `IncrementNoText`, `UpdateNoSeriesLine` — **no procedure that creates a series.** |

---

## Symptom

- On the setup card or the wizard, clicking the number-series field shows no dropdown and no *Select from full list*; or an *AssistEdit* button opens the *No. Series* list but the chosen code does not appear in the field afterwards.
- The wizard finishes, and the setup card's number-series field is blank.
- Creating the first record fails with *"<Entity> Nos. must have a value in <Setup>"*, or with a `NoSeriesManagement` reference that no longer compiles on the target version.
- A "create a series for me" option in the wizard errors because the code was typed into a `TableRelation` field before the series existed.

---

## Root cause

- **A page variable has no `TableRelation`.** The relation is a property of a *table field*. A page control bound to a variable inherits nothing, so it gets no lookup unless the control itself declares `TableRelation`.
- **A hand-rolled `OnLookup`/`OnAssistEdit` that runs the *No. Series* page and forgets to assign the result** — or assigns it to the variable but never copies the variable to the setup record on Finish.
- **`NoSeriesManagement` (codeunit 396) is obsolete**; Business Foundation's `"No. Series"` (codeunit 310) replaced it, with a different surface.
- **Business Foundation exposes no "create series" procedure** (codeunit 299 above), so a wizard that offers to create one must insert the two tables itself — and cannot do that through a field whose `TableRelation` validates against a code that does not yet exist.

---

## The pattern / fix

### Part 1 — the setup table field

```al
    field(2; "<Entity> Nos."; Code[20])
    {
        Caption = '<Entity> Nos.';
        ToolTip = 'Specifies the number series that assigns numbers to new <entities>.';
        TableRelation = "No. Series";
    }
```

Nothing else is written for the lookup: no `OnLookup`, no `Page.RunModal`, no `Lookup` or `Editable` property.

### Part 2 — the wizard bound to a temporary copy of the setup table

```al
page 50110 "<Prefix> Setup Wizard"
{
    PageType = NavigatePage;
    SourceTable = "<Prefix> Setup";
    SourceTableTemporary = true;

    // ... in the step that asks for the series:
    field("<Entity> Nos."; Rec."<Entity> Nos.")
    {
        ApplicationArea = All;
        ToolTip = 'Specifies the number series that assigns numbers to new <entities>.';
    }

    // Finish: copy the temporary record into the real one, through Validate
    local procedure FinishAction()
    var
        Setup: Record "<Prefix> Setup";
    begin
        if not Setup.Get() then begin
            Setup.Init();
            Setup.Insert(true);
        end;
        Setup.Validate("<Entity> Nos.", Rec."<Entity> Nos.");
        Setup.Modify(true);
    end;

    trigger OnOpenPage()
    begin
        Rec.Init();
        Rec.Insert();   // the temporary row the wizard edits
    end;
}
```

The control inherits the table's `TableRelation`, so the lookup is the platform's. **Only if the setup table genuinely cannot be the wizard's source** may the field bind to a variable — and then the *control* must carry `TableRelation = "No. Series";`, which is the single acceptable variable-bound form.

### Part 3 — offering to create a series (variable-bound, guarded, direct insert)

```al
    // Page variables, not TableRelation fields: the code does not exist yet.
    field(NewSeriesCode; NewSeriesCode) { ApplicationArea = All; Caption = 'New Series Code'; ToolTip = 'Specifies the code of the number series to create.'; }
    field(NewStartingNo; NewStartingNo) { ApplicationArea = All; Caption = 'Starting No.'; ToolTip = 'Specifies the first number the new series assigns.'; }

    procedure CreateNoSeries(SeriesCode: Code[20]; SeriesDescription: Text[100]; StartingNo: Code[20])
    var
        NoSeries: Record "No. Series";
        NoSeriesLine: Record "No. Series Line";
    begin
        if NoSeries.Get(SeriesCode) then
            exit;                                   // idempotent — re-running the wizard is safe
        NoSeries.Init();
        NoSeries.Code := SeriesCode;
        NoSeries.Description := SeriesDescription;
        NoSeries."Default Nos." := true;
        NoSeries.Insert(true);

        NoSeriesLine.Init();
        NoSeriesLine."Series Code" := SeriesCode;
        NoSeriesLine."Line No." := 10000;
        NoSeriesLine.Validate("Starting No.", StartingNo);
        NoSeriesLine."Increment-by No." := 1;
        NoSeriesLine.Open := true;
        NoSeriesLine.Insert(true);
    end;
```

After creating, assign the new code to the temporary setup record (`Rec.Validate("<Entity> Nos.", NewSeriesCode)`) so Finish carries it over like a picked one.

### Part 4 — the numbered table, Microsoft's shape, on codeunit `"No. Series"`

```al
namespace <Publisher>.<ExtensionShort>;

using Microsoft.Foundation.NoSeries;

table 50100 "<Prefix> <Entity>"
{
    fields
    {
        field(1; "No."; Code[20])
        {
            Caption = 'No.';

            trigger OnValidate()
            begin
                if Rec."No." <> xRec."No." then begin
                    Setup.Get();
                    NoSeries.TestManual(Setup."<Entity> Nos.");
                    Rec."No. Series" := '';
                end;
            end;
        }
        field(20; "No. Series"; Code[20])
        {
            Caption = 'No. Series';
            Editable = false;
            TableRelation = "No. Series";
        }
        // ...
    }

    trigger OnInsert()
    begin
        if Rec."No." = '' then begin
            Setup.Get();
            Setup.TestField("<Entity> Nos.");
            Rec."No. Series" := Setup."<Entity> Nos.";
            if NoSeries.AreRelated(Setup."<Entity> Nos.", xRec."No. Series") then
                Rec."No. Series" := xRec."No. Series";
            Rec."No." := NoSeries.GetNextNo(Rec."No. Series");
        end;
    end;

    procedure AssistEdit(OldRec: Record "<Prefix> <Entity>"): Boolean
    begin
        Setup.Get();
        Setup.TestField("<Entity> Nos.");
        if NoSeries.LookupRelatedNoSeries(Setup."<Entity> Nos.", OldRec."No. Series", Rec."No. Series") then begin
            Rec."No." := NoSeries.GetNextNo(Rec."No. Series");
            exit(true);
        end;
    end;

    var
        Setup: Record "<Prefix> Setup";
        NoSeries: Codeunit "No. Series";
}
```

On the card, the `"No."` control gets:

```al
                field("No."; Rec."No.")
                {
                    ApplicationArea = All;
                    ToolTip = 'Specifies the number of the <entity>.';

                    trigger OnAssistEdit()
                    begin
                        if Rec.AssistEdit(xRec) then
                            CurrPage.Update();
                    end;
                }
```

---

## Worked example — a minimal reproduction

A wizard with a page variable `NoSeriesCode: Code[20]` and `field(NoSeriesCode; NoSeriesCode)` with no `TableRelation` on the control: the field is a plain text box. Adding `TableRelation = "No. Series"` to the *control* gives it the dropdown; binding the wizard to a temporary copy of the setup table removes the need for the variable altogether and makes Finish a one-line `Validate`. On the same project, the Finish action that inserted three sample records from one variable failed with *"already exists"* naming the new series' first number — that is `Pattern-Init-Does-Not-Clear-Primary-Key.md`, not a series problem.

---

## Where else this shows up

- **Every setup table with a `... Nos.` field** — the same `TableRelation` is the entire lookup.
- **Document tables** with a `"Posting No. Series"` — same `TableRelation`, same `AreRelated` shape at posting time.
- **Journal batches** with a `"No. Series"` field.
- **Any wizard that binds to variables** for convenience — every relation the setup table had is lost with them.

---

## Caveats — check these for your own case

- **`"Default Nos."` and `"Manual Nos."`** — a series with `"Default Nos." = false` is not offered as a default; `"Manual Nos." = true` is what lets a user type a number (`TestManual` enforces this in `"No."`'s `OnValidate`).
- **Related series** — `AreRelated` / `LookupRelatedNoSeries` only matter when the setup series has relations on the *No. Series Relationships* page; the shape above works either way and is Microsoft's own.
- **Gaps** — from BC 24, `"No. Series Line".Implementation` may be *Sequence*, which `MayProduceGaps` reports; if the numbers must be gapless (legal documents), the series must be *Normal*.
- **Direct insert into tables 308/309 is a supported data operation**, but the two rows must be consistent: line 10000, a starting number, `Open = true`. Do not omit the line — a series without lines errors on first use.
- **Several inserts from one variable in the wizard** need `Clear()` between them; `Init()` alone reuses the first number (companion pattern).
- **Permission sets** must grant `tabledata "No. Series" = R` (and `RIM` on 308/309 if the wizard creates series) to the users who run the wizard.

---

## Sources

- Business Foundation 27.5.46862.46929 symbol file — tables 308, 309; codeunits 299, 308, 310 (read from `SymbolReference.json`, 2026-09-21).
- Business Foundation `NoSeries` module source (tables 308/309, codeunits 299/308/310) — <https://github.com/microsoft/BCApps/tree/main/src/Business%20Foundation/App/NoSeries> (MIT-licensed; verified reachable 2026-09-21)
- `Create Number Series` (user documentation for the *No. Series* pages the lookup opens) — <https://learn.microsoft.com/en-us/dynamics365/business-central/ui-create-number-series> (verified reachable 2026-09-21)
- `TableRelation Property` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/properties/devenv-tablerelation-property>
- Microsoft documentation referenced under © Microsoft Corporation, [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
- Companion: `Pattern-Init-Does-Not-Clear-Primary-Key.md`.
