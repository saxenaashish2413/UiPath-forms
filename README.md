# RPA_Change_Deployment_Request (UiPath Studio project)

## Open and run
1. Unzip and open `project.json` in UiPath Studio (Windows project, VB.NET).
2. If Studio reports a missing package version, open **Manage Packages** and install the latest
   **UiPath.Form.Activities** and **UiPath.System.Activities** available to you.
3. Load the form layout once: select the **Create Form** activity in `Main.xaml` → **Open Form Designer** →
   import `Forms\RPA_Change_Deployment_Request.form.json` (Import button, or paste the JSON into the JSON/Edit view) → Save.
4. Press **Run**.

## What Main.xaml does
1. Reads `Data\entity-config.json` into `strEntityConfig`.
2. Shows the form. `strEntityConfig` goes in through the `EntityConfig` form field.
   `ProcessType`, `SelectedEntity` and `DeploymentMovement` come back through the form field bindings.
3. Reads `SelectedPolicyRoles` (String[]) from the form's output JSON, and fills any of the other three that came back empty.
4. Logs the request and shows a summary message box. If the form is closed without submitting, it logs a warning.

## Add or change entities
Edit `Data\entity-config.json` only. The form needs no changes. See `../SETUP.md` for the format and a DataTable/Excel option.
