# Pattern: Capturing an API key in a page — `SecretText` cannot be a page field, and `IsolatedStorage.Set` is the only setter

**A wizard or setup page needs the user to type an API key, and the key must then be stored without ever living in a table field or being readable in the debugger. Binding a `SecretText` variable to a page field does not compile; neither does `IsolatedStorage.SetSecret`. The working shape is a `Text` page variable with `ExtendedDatatype = Masked`, copied into a `SecretText` in `OnValidate`, then stored with `IsolatedStorage.Set(Key, SecretText, DataScope)` and read back with `IsolatedStorage.Get(Key, DataScope, SecretText)`.**

`SecretText` (runtime 12.0) is deliberately one-directional: anything can flow into it, nothing can flow out to a debuggable type. The client-facing page control is a debuggable surface, so the type cannot bind there. The storage API gained a `SecretText` overload in runtime 13.0 under the existing name `Set`; there is no method called `SetSecret`.

---

## Applicability and version

A **Business Central AL platform pattern** for any extension that collects a credential from a user: Azure OpenAI keys for Partner-billed Copilot, third-party API keys, SMTP or webhook secrets, licence tokens.

**Version:** `SecretText` — Microsoft Learn: *"Available or changed with runtime version 12.0."* `IsolatedStorage.Set(Text, SecretText [, DataScope])` — *"Available or changed with runtime version 13.0."* (BC 2024 release wave 1). The `Text` overload of `Set` is runtime 2.0. `ExtendedDatatype = Masked` is runtime 1.0. Verified with the AL compiler 18.0 against Business Central 27.5 symbols.

---

## Symptom

- `field(APIKey; WizardAPIKey)` where `WizardAPIKey: SecretText` → compile error **AL0156** (the type cannot be used as a page field source).
- `IsolatedStorage.SetSecret(KeyTok, WizardAPIKey, DataScope::Company)` → compile error **AL0132**, no such method.
- Attempts to "work around" by declaring the key as `Text` end-to-end compile, and then the key is visible in the debugger, in snapshot debugging, and in any `Message`/telemetry that happens to include the variable.
- Storing the key in a table field "because `SecretText` cannot be a field type" ships the secret into the database and into every export of that table.

---

## Root cause

Verified against Microsoft Learn, `Protecting sensitive values with the SecretText data type` —
<https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-secret-text> (fetched 2026-10-10). Quoted verbatim:

> Any value of type `Text` or `Code` can be assigned to a `SecretText` value.

> The [SecretText data type] guarantees that the value remains nondebuggable by preventing any assignment from itself to any debuggable type. This constraint includes the `Variant` data type.

```al
    Credential := PlainText; // Allowed
    PlainText := Credential; // Blocked
```

> The `Unwrap` method allows the value to be extracted from a `SecretText` to a textual destination for compatibility reasons. It's only permitted when building applications for the 'OnPrem' scope [...]

A page field is a two-way binding between a control and a variable. The platform would have to read the value back out to render it, which the type forbids; hence a `SecretText` variable is not a legal page-field source. A plain `Text` variable is, and the assignment `SecretVar := TextVar` is the "Allowed" direction.

On storage, `IsolatedStorage.Set(Text, SecretText [, DataScope]) Method` —
<https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/isolatedstorage/isolatedstorage-set-string-secrettext-datascope-method> (fetched 2026-10-10):

> **Version**: *Available or changed with runtime version 13.0.*
> `[Ok := ] IsolatedStorage.Set(Key: Text, Value: SecretText [, DataScope: DataScope])`
> *[Optional] DataScope* [...] If a value is not passed in, the default value DataScope::Module will be used.

The `Text` overload's remarks add a constraint that bites in wizards —
<https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/isolatedstorage/isolatedstorage-set-string-string-datascope-method> (fetched 2026-10-10):

> The **Set** method initiates a write transaction, which means that it cannot be succeeded by code that opens a modal page. If you want to open a modal page, it must be done before the **Set** method is called.

And on masking, `ExtendedDatatype property` —
<https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/properties/devenv-extendeddatatype-property> (fetched 2026-10-10):

> **Masked** — Displays the value as dots. This will only have effect on fields, where the user can enter and display textual data (including numbers, time, date etc.)

---

## The pattern / fix

### Capture

```al
    // page (NavigatePage wizard or setup card)
    field(WizardAPIKeyTxt; WizardAPIKeyTxt)
    {
        ApplicationArea = All;
        Caption = 'API Key';
        ExtendedDatatype = Masked;                 // dots in the UI
        ToolTip = 'Enter the API key. It is stored securely and never displayed again.';

        trigger OnValidate()
        begin
            WizardAPIKey := WizardAPIKeyTxt;       // Text -> SecretText: the allowed direction
        end;
    }
    ...
    var
        WizardAPIKey: SecretText;
        WizardAPIKeyTxt: Text[250];                // page-bound; never persisted, never logged
```

Keep `WizardAPIKeyTxt` out of every `Message`, label substitution and telemetry dimension. Clear it when the page closes if it must not linger (`Clear(WizardAPIKeyTxt)` in `OnQueryClosePage` after the secret has been handed on).

### Store and read

```al
    procedure SavePartnerCredentials(EndpointURL: Text; APIKey: SecretText)
    begin
        IsolatedStorage.Set(PartnerEndpointURLKeyTok, EndpointURL, DataScope::Company);   // Text overload
        IsolatedStorage.Set(PartnerAPIKeyTok, APIKey, DataScope::Company);                // SecretText overload (runtime 13+)
        MarkConfigured();                                                                 // a Boolean on the setup table, so IsConfigured() need not touch storage
    end;

    procedure GetPartnerAPIKey(): SecretText
    var
        APIKey: SecretText;
    begin
        if IsolatedStorage.Get(PartnerAPIKeyTok, DataScope::Company, APIKey) then
            exit(APIKey);
        exit(APIKey);                                                                     // empty SecretText
    end;

    procedure ClearPartnerCredentials()
    begin
        if IsolatedStorage.Contains(PartnerAPIKeyTok, DataScope::Company) then
            IsolatedStorage.Delete(PartnerAPIKeyTok, DataScope::Company);
    end;
```

`DataScope::Company` keeps one key per company, which is what a per-company setup record implies. The default is `DataScope::Module` (one per extension per tenant); choose deliberately.

### Consume

Pass the `SecretText` straight into APIs that accept it. The System Application's `Azure OpenAI.SetAuthorization(ModelType, Endpoint, Deployment, ApiKey: SecretText)` and `HttpHeaders.Add(Name, SecretText)` do. Never `Unwrap()` in a Cloud-target app.

### Test the credential before saving it

For a wizard, add a **Test Connection** footer action that uses the in-memory `SecretText` against the real service, and gate **Next** on a passing test. Save to IsolatedStorage only on **Finish**, so a mistyped key is never persisted.

---

## Worked example — a minimal reproduction

Shipped project: General Journal Copilot (OnlyCopilotFans), BC 27.5. The first generated draft declared `field(APIKey; WizardAPIKey)` with `WizardAPIKey: SecretText` and called `IsolatedStorage.SetSecret(...)`; neither compiled. The shipped fix is the capture/store/read code above, with the Azure OpenAI endpoint URL stored as `Text` under a second key and a `"Partner Endpoint Configured"` Boolean on the setup table. The wizard's Test Connection action calls the AI module with the in-memory `SecretText` before Finish writes anything.

---

## Where else this shows up

- **Partner-billed Copilot** (own Azure OpenAI key) and any **bring-your-own-key** AI, mapping, payment or messaging integration.
- **OAuth client secrets** captured once by an administrator.
- **Setup cards** that show an "Update credentials" link: do not display the stored key; offer re-entry only.
- **Upgrade code** migrating a secret out of a legacy table field into IsolatedStorage (read the field into a `SecretText`, `Set`, blank the field, and mark the field obsolete).

---

## Caveats — check these for your own case

- **`Set` starts a write transaction.** Do not open a modal page (a confirmation, a lookup) after it in the same flow. Confirm first, then `Set`.
- **IsolatedStorage is per extension.** Another app cannot read your key, and your key does not survive if your app is uninstalled with data deletion. Document re-entry in the uninstall/reinstall path.
- **`DataScope::Company` vs `Module` vs `User`.** A user-scoped key disappears for other users; a module-scoped key is shared across companies. Match the scope to the setup record's scope.
- **`SecretText` cannot be a table field type.** That is by design; the storage is IsolatedStorage, not a table.
- **Runtime 13 minimum for the `SecretText` overload of `Set`.** On runtime 12 you can still hold the key in a `SecretText`, but storing it requires `Unwrap()`, which is OnPrem-only; in practice target runtime 13+.
- **Do not log the `Text` shadow variable**, including in `StrSubstNo` for error messages. Treat it as the secret it is.

---

## Sources

- `Protecting sensitive values with the SecretText data type` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/devenv-secret-text> (fetched and verified 2026-10-10)
- `SecretText data type` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/secrettext/secrettext-data-type> (fetched and verified 2026-10-10)
- `IsolatedStorage.Set(Text, SecretText [, DataScope]) Method` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/isolatedstorage/isolatedstorage-set-string-secrettext-datascope-method> (fetched and verified 2026-10-10)
- `IsolatedStorage.Set(Text, Text [, DataScope]) Method` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/methods-auto/isolatedstorage/isolatedstorage-set-string-string-datascope-method> (fetched and verified 2026-10-10)
- `ExtendedDatatype property` — <https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/developer/properties/devenv-extendeddatatype-property> (fetched and verified 2026-10-10)
- Compiler checks: `SecretText` as a page-field source → AL0156; `IsolatedStorage.SetSecret` → AL0132 (AL compiler 18.0, BC 27.5 symbols, 2026-10-09).
- Microsoft documentation quoted under © Microsoft Corporation, [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
- Companion: the OnlyBCFans Business Central Copilot Blueprint, reference 04 (Partner-billed path).
