# Pattern: A `PromptDialog` closes with `Action::OK`, not `LookupOK` — and the save belongs to whichever side owns the error

**The "Keep it" button on a Copilot prompt dialog does nothing at all: no insert, no message, no error. The page's `OnQueryClosePage` compared `CloseAction` to `Action::LookupOK`, which a `PromptDialog` never sends. `systemaction(OK)` closes the page with `Action::OK`; `systemaction(Cancel)` with `Action::Cancel`. Once that is right, Microsoft documents two sanctioned places to perform the save: inside `OnQueryClosePage`, or in the caller from the value `RunModal()` returns.**

The `PromptDialog` page type (runtime 12.1+) exposes a fixed set of system actions. `OK` and `Cancel` carry no `OnAction` trigger; the only hook for "the user accepted" is the page-close path. Code that was written from memory of lookup pages, or copied from a `List` page used as a lookup, tests for `LookupOK` and never matches.

---

## Applicability and version

A **Business Central AL platform pattern** for any `PageType = PromptDialog` page, regardless of the Copilot feature it fronts. The close-action half also applies to any modal page closed with an **OK** button rather than a lookup selection.

**Version:** `PromptDialog` exists from **runtime 12.1** (Business Central 2023 release wave 2). The `Action` option and `OnQueryClosePage` are runtime 1.0. Verified with the AL compiler 18.0 against Business Central 27.5 symbols.

---

## Symptom

- The user clicks **Keep it** (or whatever caption `systemaction(OK)` carries). The dialog closes. Nothing else happens: no record is inserted, no message appears, no error is raised, the audit log is silent.
- **Generate** works and the content area shows a valid proposal, so the AI path is fine. Only acceptance is dead.
- **Discard** may appear to work, because `Action::Cancel` was spelled correctly or because "nothing happened" is the expected result of a discard.
- Adding `OnAction` to `systemaction(OK)` to "fix" it fails to compile: **AL0811** on `PromptDialog` system actions `OK` and `Cancel`.

Typical offending code:

```al
    trigger OnQueryClosePage(CloseAction: Action): Boolean
    begin
        case CloseAction of
            Action::LookupOK:          // never matches on a PromptDialog
                AcceptedByUser := true;
            Action::Cancel:
                LogDiscard();
        end;
        exit(true);
    end;
```

---

## Root cause

Verified against Microsoft Learn, `Action system option` —
<https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/action/action-option> (fetched 2026-10-10). Quoted verbatim:

> **OK** — Represents the result of the user closing a page window by performing one of the following actions: - Chooses the **OK** button. - Chooses the **X** button when there was no **Cancel** button on the window. - Presses the Esc key when there is no **Cancel** button on the window.

> **LookupOK** — Represents the result of the user closing a lookup window by performing one of the following actions: - Chooses the **OK** button. - Chooses an item in the Lookup window.

A `PromptDialog` is a page window with an OK button, not a lookup window. Its `systemaction(OK)` therefore closes with `Action::OK`. Microsoft's own sample on the `PromptDialog` reference page tests exactly that —
<https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-page-type-promptdialog> (fetched 2026-10-10):

> The `OnQueryClosePage` trigger saves the job proposal when the page is closed with the `OK` action.

```al
    trigger OnQueryClosePage(CloseAction: Action): Boolean
    var
        SaveCopilotJobProposal: Codeunit "Save Copilot Job Proposal";
    begin
        if CloseAction = CloseAction::OK then
            SaveCopilotJobProposal.Save(CustomerNo, CopilotJobProposal);
    end;
```

The same page states which system actions exist:

> These system actions are `Generate`, `Regenerate`, `Attach`, `Ok` and `Cancel`.

and the content-mode design article says where the save code may live —
<https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/copilot-design-content-mode> (fetched 2026-10-10):

> There are two system actions that control the behavior: `systemaction(OK)` for saving and `systemaction(Cancel)` for discarding. These actions appear in the UI by default without you having to add any code [...] However, you do have to add code that handles the content based on which action the user chooses.

> Add the code that saves or discards the proposal using either the [OnQueryClosePage] or the value returned from the [RunModal].

> Don't save AI-generated proposals to the database until the user chooses to do so.

On what `OnQueryClosePage` is allowed to do, the trigger reference —
<https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/triggers-auto/page/devenv-onqueryclosepage-page-trigger> (fetched 2026-10-10):

> If an error occurs in the OnQueryClosePage trigger or it returns **false**, then the page is not closed.

> You can use this trigger to write to the database.

So writing inside the trigger is permitted, and an error there keeps the dialog open rather than disappearing. A project note that "errors inside `OnQueryClosePage` are silently swallowed" is **not** corroborated by Microsoft Learn; the silent no-op described above was the `LookupOK` mismatch, not a platform behaviour.

---

## The pattern / fix

### 1. Match `Action::OK` and `Action::Cancel`

```al
    trigger OnQueryClosePage(CloseAction: Action): Boolean
    begin
        case CloseAction of
            Action::OK:
                begin
                    if not HasValidSuggestion then
                        exit(false);           // nothing to keep: stay open rather than close empty-handed
                    AcceptedByUser := true;
                    LogAccept();
                end;
            Action::Cancel:
                LogDiscard();
        end;
        exit(true);
    end;
```

Returning `false` when there is nothing to accept is the documented way to keep the page open. Do not add `OnAction` to `OK` or `Cancel`; the compiler rejects it.

### 2. Choose where the save runs — both are documented

**Option A — save inside `OnQueryClosePage`** (Microsoft's sample). Simplest. An error in the save keeps the dialog open and shows the error to the user, which is acceptable UX for "the proposal could not be saved".

**Option B — save in the caller after `RunModal()`**. The dialog only records the decision and exposes the accepted values through public procedures; the action that opened it performs the write and refreshes its own `Rec`:

```al
    // in the host page action
    GJCopilotDialog.SetContext(Rec."Journal Template Name", Rec."Journal Batch Name", Rec."Document No.");
    if GJCopilotDialog.RunModal() = Action::OK then begin        // RunModal returns the close action
        AcceptedAccountNo := GJCopilotDialog.GetAcceptedAccountNo();   // page variables survive while the caller holds the page variable
        if not LineInserter.InsertBalancingLine(..., AcceptedAccountNo) then
            Message(LineInsertFailureMsg)
        else
            if not LineInserter.VerifyBalanceAfterInsertion(...) then
                Message(StillUnbalancedWarningMsg);
    end;
    CurrPage.Update(false);
```

Choose B when the write touches the caller's own record set (a journal, a document's lines) and the caller needs to validate the business outcome afterwards and refresh; choose A when the proposal is self-contained (a new record or set of records with its own save codeunit). Either way the model's output is validated against the database before the write.

---

## Worked example — a minimal reproduction

Shipped project: General Journal Copilot (OnlyCopilotFans), BC 27.5, May 2026. Symptom exactly as above: "Keep It" was a silent no-op after a successful Generate. The case branch read `Action::LookupOK`. Changing it to `Action::OK` restored acceptance; the project then moved the journal-line insert into the host page action (Option B) so the balancing check and `CurrPage.Update(false)` ran on the journal page.

```al
// Before
Action::LookupOK:

// After
Action::OK:
```

---

## Where else this shows up

- Any **`PromptDialog`** page with a Keep/Discard decision.
- **`StandardDialog` / `ConfirmationDialog`** pages and ordinary pages run with `RunModal()` and closed by an **OK** button: they also return `Action::OK`, not `LookupOK`.
- **Lookup pages** are the inverse trap: a `List` page opened from a field's `OnLookup` closes with `Action::LookupOK` / `LookupCancel`, so code copied *from* a lookup into a dialog, or *from* a dialog into a lookup, breaks in both directions.
- **Request pages** and `Page.RunModal()` results compared with the wrong member.

---

## Caveats — check these for your own case

- **`OnQueryClosePage` only fires for modal pages.** Learn: "The trigger is only invoked when the page is hosted in a modal popup window (MPO)." A `PromptDialog` is always modal, so this is not a problem there, but it is for pages you also open non-modally.
- **The X button maps to `OK` or `Cancel` depending on whether a Cancel button exists** (see the Action option quotes). On a `PromptDialog` with `systemaction(Cancel)` present, X means Cancel.
- **`Regenerate` and `Generate` do take `OnAction`**; only `OK`/`Cancel` are trigger-less. `Attach` behaves like a system action with platform-handled behaviour.
- **Option B relies on page variables remaining readable after the page closed.** They are, as long as the caller still holds the `Page` variable instance; do not `Clear` it before reading the getters.
- **Responsible AI:** whichever option you choose, nothing is written before the user chooses OK. Logging the generation for audit is fine; inserting the proposal is not.

---

## Sources

- `Action system option` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/action/action-option> (fetched and verified 2026-10-10)
- `OnQueryClosePage (Page) trigger` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/triggers-auto/page/devenv-onqueryclosepage-page-trigger> (fetched and verified 2026-10-10)
- `The PromptDialog Page Type` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-page-type-promptdialog> (fetched and verified 2026-10-10)
- `Design content mode of prompt dialog page` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/copilot-design-content-mode> (fetched and verified 2026-10-10)
- Compiler check: `systemaction(OK)` with an `OnAction` trigger → AL0811 (AL compiler 18.0, BC 27.5 symbols, 2026-10-09).
- Microsoft documentation quoted under © Microsoft Corporation, [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
- Companion: the OnlyBCFans Business Central Copilot Blueprint, reference 05 (PromptDialog page and host-page action).
