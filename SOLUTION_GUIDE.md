# RPA Change & Deployment Request – UiPath solution

UiPath Forms + Studio + Standalone Orchestrator 2025.10.2 REST API + Queue + REFramework.

```
Requester (attended)                                   Performer (unattended, REFramework)
 Main ─ LoadProvisioningConfig ─ GetEntityConfiguration   GetTransactionData (queue)
      └ ShowRequestForm (UiPath Form) ◄─┐                  Process
      └ ValidateRequest ── errors ──────┘                   ├ LoadProvisioningConfig + ValidateRequest (again)
      └ SubmitRequestToQueue                                ├ ResolveProvisioningTarget (folder path, assignee)
          ├ Request ID RPA-YYYY-NNNNN                       ├ GetAccessToken (client credentials)
          └ Add Queue Item (Reference = Request ID) ──────► ├ GetOrCreateFolder  (Create Folder / Get Folder)
                                                            ├ GetRoleIds         (Get Roles)
                                                            ├ GetAssigneeId
                                                            ├ AssignFolderRoles  (Assign + apply roles + verify)
                                                            ├ SetProgress / SetTransactionStatus Output (Update Queue Item)
                                                            └ SendNotification   (success / failure email)
```

## Deliverables
| Path | What |
|---|---|
| `RPA_Change_Request_Requester.zip` / folder | Attended Studio project (form, validation, Request ID, queue item) |
| `RPA_Provisioning_Performer.zip` / folder | REFramework Studio project (Orchestrator API provisioning) |
| `RPA_Provisioning_Config.xlsx` | Entities, Policies, EntityPolicyMapping, DeploymentMapping, Roles, ApplicationSettings |
| `RPA_Change_Deployment_Request.form.json` | The form (also inside the requester project under `Forms\`) |
| `form-preview.png` | The form as rendered in the browser test |
| `verification/` | Test scripts and their output (see "What was verified") |

## One-time Orchestrator setup (2025.10.2 standalone)
1. **Folder** `Shared/RPA Provisioning` (or change `QueueFolder` / `OrchestratorQueueFolder`).
2. **Queue** `RPA_Provisioning_Requests` in that folder: **Enforce unique references = Yes** (required for safe Request IDs), Auto Retry = 1 (match `QueueMaxRetryNumber`).
3. **External Application** (Identity Server management portal → External Applications): confidential, **application scopes** `OR.Folders OR.Users` (match `ApiScope`). Copy the Client ID and secret.
4. **Credential asset** `RPA_Provisioning_ApiClient` in `Shared/RPA Provisioning`: Username = Client ID, Password = Client Secret. Optional SMTP credential asset (`SmtpCredentialAsset`).
5. **Permissions for the external app** (Tenant → Manage Access → assign roles): a tenant role with Folders View/Create/Edit, Users View, Roles View, and the right to assign users on the parent folders (e.g. Folder Administrator on `UAT` and `Prod`). Without it the API returns 403.
6. **Robots**: requester users need *Queues View* + *Transactions View/Create* in the queue folder. The performer robot needs the same plus *Transactions Edit* and *Assets View*.
7. **Directory groups / users** in `Entities.AssigneeName` must already exist at tenant level (Tenant → Manage Access). The performer fails the item with a clear message otherwise.
8. Put `RPA_Provisioning_Config.xlsx` on a share both robots can read; set requester `in_ConfigPath` and performer `ProvisioningConfigPath` to it.
9. Publish both projects; add a queue trigger for the performer; publish the requester to UiPath Assistant.

## Configuration (RPA_Provisioning_Config.xlsx)
* **Entities** `Entity | IsActive | Description | AssigneeName | AssigneeType` – the user/group (User, Group, Robot, ExternalApplication) that gets folder access.
* **Policies** `Policy | IsActive | Description` – deactivating a policy removes it from every entity.
* **EntityPolicyMapping** `Entity | Policy` – drives the Policy Role list.
* **DeploymentMapping** `Entity | Deployment | SourceEnvironment | TargetEnvironment | ParentFolderPath` – drives Deployment Movement and where the folder is created (`{Entity}` token allowed).
* **Roles** `Policy | OrchestratorRole | RoleId | Notes` – one row per role; several rows per policy allowed. RoleId blank = resolved by name via the API (recommended); filled = must match what the API returns.
* **ApplicationSettings** – Orchestrator URL, Organization, Tenant, `ApiBaseUrlTemplate`, `IdentityTokenUrl`, credential asset, `ApiScope`, timeout, retry, queue, Request ID format, field lengths, folder name template, logo path, SMTP. Descriptions are in the sheet. No secrets.

Folder created = `ParentFolderPath / FolderNameTemplate`, e.g. `UAT/Entity 2/Invoice Matching Bot`.

## Request ID (RPA-YYYY-NNNNN), safe for concurrent requests
No local counter. The requester reads the highest existing reference for the year from the queue (**Get Queue Items**, reference
starts with `RPA-2026-`), then adds the item with `Reference = next number`. The queue's unique-reference constraint makes Orchestrator
reject a duplicate if two people submit at the same moment; the loser retries with the next number (`RequestIdMaxAttempts`).
Two robots therefore can never hold the same Request ID.

## Dynamic Entity behaviour
`GetEntityConfiguration.xaml` groups the mapping sheets per entity and passes the result to the form. In the form, Policy Role and
Deployment Movement are select components whose options are computed from that data with **Refresh On = SelectedEntity** and
**Clear Value On Refresh**, so changing Entity clears both, reloads them and hides them until an Entity is chosen. Custom validation
rejects values not allowed for the entity, and the workflow re-validates on Submit and again in the performer.
A workflow round-trip (Do block) is not needed for this: UiPath Forms run the form.io engine, and UiPath's own Do-block trigger relies on
the same component JavaScript. If your Forms version blocks component JavaScript, tell me and I'll switch it to a Do-block refresh.

## What was verified here, and what was not
Verified in this environment:
* All 18 Invoke Code bodies compile with the VB.NET compiler against .NET 8 (Option Strict On) and pass 31 functional checks, including a
  run against a mock Orchestrator API (token, folder get/create, 503 retry, roles, users, AssignUsers, verify, idempotent re-run, 401 handling)
  and a local SMTP server.
* All 554 VB expressions in the generated XAML compile with each workflow's variables/arguments in scope (Option Strict On).
* The form, in Chromium with form.io 4.14: entity-driven lists, clear-on-change, required/length/pattern/date validation, blocked invalid
  Submit, Reset, Cancel, the workflow error box. Its real submission passes the workflow validation.
* Every XAML file is well-formed; activity XAML for ReadRange, InvokeWorkflowFile, InvokeCode, LogMessage, GetRobotCredential,
  SetTransactionProgress, SetTransactionStatus, ForEach, TryCatch follows the official REFramework 24.10 files.

Not verified (no Studio or Orchestrator here) – check on first open/run:
1. **Create Form** XAML (`uf:FormActivity` property names) and importing the form JSON into the designer.
2. **Get Queue Items** / **Add Queue Item** attribute names (`QueueType`, `Result`, `FilterStrategy`, `ItemInformation`). If Studio flags one, re-select the property in the panel.
3. The exact text of Orchestrator's duplicate-reference error (matched on "duplicate"/"unique").
4. Package versions: System 24.10.5, Excel 2.24.1 (from the REFramework template), Form 23.10.4 – update in Manage Packages if your feed differs.
5. The Identity token URL for your install (`{OrchestratorUrl}/identity/connect/token`) and the role names on your tenant.
