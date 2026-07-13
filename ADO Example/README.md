# ADO Example

This folder demonstrates a DRY Azure DevOps deployment lifecycle for Azure App Service using separate app and template repos.

## Repository-style layout

- `someapp-repo/` — app team consumer repo example
- `template-repo/` — shared template repo example

## Files

- `someapp-repo/someapp-automation.yml`  
  Consumer pipeline that lives in the app repo. It imports the shared stage template via `resources.repositories` and passes lifecycle parameters.

- `template-repo/azure-appservice-deploy-stages.yml`  
  Reusable stage template. It loops through environments, then loops through app services within each environment.

- `template-repo/azure-appservice-deploy-step.yml`  
  Nested step template containing the single `AzureWebApp@1` task.

## How it works

1. `someapp-repo/someapp-automation.yml` references the shared repo alias (`templatesRepo`) and calls `azure-appservice-deploy-stages.yml@templatesRepo`.
2. The stage template iterates over `parameters.environments`.
3. For each environment, it creates a stage using `stageName` and deployment jobs per app service.
4. Each deployment job calls the nested step template.
5. The nested step template runs `AzureWebApp@1` using:
   - non-prod: `zipDeploy`
   - prod: `runFromPackage`

## Parameter model

The caller passes:

- `artifactName` (default: `drop`)
- `packagePath` (default: `$(Pipeline.Workspace)/drop/app.zip`)
- `vmImage` (default: `ubuntu-latest`)
- `environments` (object array of lifecycle stages)

Each environment object includes:

- `name`
- `stageName`
- `displayName`
- `isProd`
- `dependsOn`
- `azureServiceConnection`
- `environmentResource`
- `appServices` (array of `{ name, deploymentName }`)

## Notes

- The sample consumer file currently uses `trigger: none` and `pr: none`.
- Stage sequencing is controlled per environment through `dependsOn`.
- This structure keeps deploy task behavior centralized and easy to maintain.

## Adopting this pattern

1. Copy/adapt `someapp-repo/someapp-automation.yml` in each app repo.
2. Point `resources.repositories.name` to your shared template repo.
3. Define your environment entries (`dev`, `prod`, etc.) and app service names.
4. Keep reusable templates centralized in the shared repo.
