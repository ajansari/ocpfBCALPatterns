# Pattern: A page extension can `addlast(Prompting)` even when the base page declares no Prompting area

**A Copilot action was dropped into the ordinary Actions menu because "the base page has no Prompting area, so `addlast(Prompting)` cannot work". It can. The anchor for `addlast`/`addfirst` is an area name, the Prompting area is supported on the page's type, and the extension creates the area if the base page did not. The only thing that fails is misspelling it — `Prompts`, `Prompt`, `Copilot` — which produces an unknown-anchor error that reads as "no such area on this page".**

`area(Prompting)` is where the client renders Copilot prompt actions: the sparkle icon in the action bar, the floating action bar on lists and worksheets, the "Here's what Copilot can do on this page" menu on cards. An action placed in `processing` instead still opens the dialog, but it is not recognised as a Copilot action by the client or by Copilot itself.

---

## Applicability and version

A **Business Central AL platform pattern** for any page extension that adds a Copilot entry point to a Microsoft or third-party page.

**Version:** Microsoft Learn, *Prompting using a floating action bar*: *"Applies to: Business Central 2024 release wave 1 (runtime 13) and later."* Supported page types: *"Starting with runtime 13, the supported page types include `List`, `ListPart`, `StandardDialog`, and `Worksheet`. With runtime 14, the supported page types also include `Card`, `Document`, and `ListPlus`."* Verified with the AL compiler 18.0 against Business Central 27.5 symbols on page 39 "General Journal" (a `Worksheet`) and page 42 "Sales Order" (a `Document`), neither of which declares a Prompting area itself.

---

## Symptom

- A design note or review comment says: *"The base page does not declare an `area(Prompts)` section. Third-party extensions cannot use `addlast(Prompts)` to contribute entries to the Copilot dropdown."* The action is then placed in `addlast(processing)` with a promoted actionref, and the note promises to revisit "when Microsoft adds a Prompts area".
- The shipped action works but shows up as an ordinary action: no Copilot grouping, no floating action bar, no appearance in the Copilot pane.
- Reading the base page's symbols confirms the area is absent (page 39 in 27.5 declares only `navigation`, `processing` and `Promoted`), which seems to prove the note right.

---

## Root cause

Two facts were conflated: *the base page does not declare the area* (true) and *therefore a page extension cannot anchor to it* (false). The anchor rules —
<https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-page-ext-object> (fetched 2026-10-10), quoted verbatim:

> `addfirst` — `addfirst(Anchor)` — **Anchor**: areas and groups
> `addlast` — `addlast(Anchor)` — **Anchor**: areas and groups

The Prompting article shows the extension form explicitly —
<https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-page-prompting-floating-actionbar> (fetched 2026-10-10):

> To create prompt actions, you must create a new area in the `actions` section of the page object. The area must be set to `area(Prompting)`. You can then add one or more actions in the area and run a `PromptDialog` object that you want to activate when the action is clicked. Only objects of the `PromptDialog` page type can be run from a prompting area.

> Here's an example if you're extending a list page:

```al
actions
{
    addlast(Prompting)
    {
        action(MyPromptAction)
        {
            Caption = 'Draft a proposal';
            Image = Sparkle;
            RunObject = page "My Prompt Dialog";
        }
    }
}
```

The article's detailed example (Microsoft's own BCTech `JobPlanningLinesCopilot.PageExt.al`) does the same with `addlast(Prompting)` on the Job Planning Lines part, using `OnAction` triggers rather than `RunObject`.

Compiler verification (AL 18.0, BC 27.5 symbols): a page extension with `addlast(Prompting)` on page 39 "General Journal" compiles with 0 errors, and the resulting symbol package contains the action under the extension. The failed attempt in the originating project used `addlast(Prompts)`: an area name that does not exist on any page.

---

## The pattern / fix

```al
pageextension 50130 "PFX Copilot Journal Ext" extends "General Journal"
{
    actions
    {
        addlast(Prompting)                               // creates the area if the base page has none
        {
            action(PFXSuggest)
            {
                ApplicationArea = All;
                Caption = 'Suggest balancing account';   // verb first; do not say "Copilot" in the caption
                Image = Sparkle;                         // Sparkle normally; SparkleFilled only to emphasise one of several
                Visible = IsCapabilityRegistered;        // hide where Copilot cannot run (on-premises)
                ToolTip = 'Asks Copilot which G/L account balances this document. You confirm before anything is inserted.';

                trigger OnAction()
                begin
                    RunSuggest();                        // opens the PromptDialog page via RunModal and handles the result
                end;
            }
        }
    }

    trigger OnOpenPage()
    var
        CopilotCapability: Codeunit "Copilot Capability";
    begin
        IsCapabilityRegistered := CopilotCapability.IsCapabilityRegistered(Enum::"Copilot Capability"::PFXSuggestSomething);
    end;

    var
        IsCapabilityRegistered: Boolean;
}
```

Design guidance from the same article, verbatim:

> You should set the [Image property] to either `Sparkle` or `SparkleFilled`.

> You shouldn't use "Copilot" in prompt action captions. Instead, focus on the assistive task that Copilot performs, starting with a verb such as draft, suggest, search, or troubleshoot.

> Prompt actions are displayed in both Business Central online and on-premises environments. However, Microsoft Copilot is exclusively available for Business Central online. Therefore, it's good practice to display prompt actions only in online environments [...] The preferred approach is to register the Copilot capability only if the environment is online [...] Then, use the [CopilotCapability.IsCapabilityRegistered] method on the prompt action's Visible property.

### Fallback when the page type is not supported

> If your Copilot feature is reached from a page that doesn't yet support Copilot prompt actions, you can use a standard action.

That is the only legitimate reason to put a Copilot action in `processing`: the page type is outside the supported list for your runtime. Check the page's `PageType`, not whether it already has a Prompting area.

---

## Worked example — a minimal reproduction

Shipped project: General Journal Copilot (OnlyCopilotFans), BC 27.5. The technical design recorded that `addlast(Prompts)` failed and concluded page 39 could not take Copilot actions; v1.0.0.13 ships the action in `addlast(processing)` with an `addfirst(Promoted)` actionref. Compiling the same action under `addlast(Prompting)` against the identical symbols succeeds. The design note was corrected on 2026-10-10 and the move to `Prompting` is scheduled for the next release.

```al
// Does not compile - no such area on any page
addlast(Prompts) { action(SuggestBalancingAccount) { ... } }

// Compiles on page 39 (Worksheet) in BC 27.5, although page 39 declares no Prompting area
addlast(Prompting) { action(SuggestBalancingAccount) { ... } }
```

---

## Where else this shows up

- **Every Copilot entry point added by extension** to Microsoft's lists, worksheets, cards and documents.
- **`addlast(Promoted)` on a base page with no promoted section** is the same mechanism and already widely used, which is a quick way to convince a reviewer.
- **Other named areas** a base page may not declare (`creation`, `reporting`): the extension can anchor to them regardless.
- **Agents and code generators** that check the base page's symbol file for the area and refuse to emit `addlast(Prompting)`; this pattern is the counter-evidence.

---

## Caveats — check these for your own case

- **Area name is exactly `Prompting`.** `Prompts`, `Prompt` and `Copilot` all fail with an unknown-anchor error that looks like a page limitation.
- **The runtime 13/14 page-type matrix is real.** On a page type outside it (for example a `RoleCenter` or `CardPart`), fall back to a standard action as Microsoft describes.
- **`RunObject` versus `OnAction`.** One sentence in the article says *"Prompt actions are displayed only if the `RunObject` property is specified"*, yet Microsoft's own detailed example uses `OnAction` triggers with no `RunObject`. The compile succeeds either way. If your action does pre-validation before opening the dialog (so `RunObject` is not an option), test in a sandbox that the client still renders it in the Copilot area; the originating project had not yet done this when the pattern was written.
- **Only `PromptDialog` pages may be run from a prompting area**, per the article. Do not point a prompt action at a `Card` or `List`.
- **`Enabled` versus `Visible`.** Microsoft's guidance hides the action where Copilot is unavailable (`Visible`). A separate `Enabled` expression for "configured and capability active" is reasonable on top, with a tooltip that says what to do when disabled; tooltips must be string literals (a Label or variable fails with AL0219).

---

## Sources

- `Prompting using a floating action bar` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-page-prompting-floating-actionbar> (fetched and verified 2026-10-10)
- `Page extension object` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-page-ext-object> (fetched and verified 2026-10-10)
- `The PromptDialog Page Type` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-page-type-promptdialog> (fetched and verified 2026-10-10)
- Microsoft BCTech sample — <https://github.com/microsoft/BCTech/blob/master/samples/AzureOpenAI/Advanced_SuggestJob/SuggestResource/JobPlanningLinesCopilot.PageExt.al>
- Compiler check: `addlast(Prompting)` on page 39 "General Journal" and page 42 "Sales Order", 0 errors (AL compiler 18.0, Base Application 27.5.46862.49619 symbols, 2026-10-09); page 39's own action areas in those symbols: `navigation`, `processing`, `Promoted`.
- Microsoft documentation quoted under © Microsoft Corporation, [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
- Companion: the OnlyBCFans Business Central Copilot Blueprint, reference 05 (PromptDialog page and host-page action).
