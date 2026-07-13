# ADO Example

This folder demonstrates a DRY Azure DevOps deployment lifecycle for Azure App Service using **separate app and template repos**.

## Repository-style layout in this example

- `someapp-repo/` — represents an app team's repo
- `template-repo/` — represents a shared template repo

## File overview

- `someapp-repo/azure-pipelines.yml`  
  Consumer pipeline (what an app team keeps in their own repo). It references the shared template repo and passes environment-specific parameters.

- `template-repo/azure-appservice-deploy-stages.yml`  
  Stage template that defines the deployment lifecycle. It loops through environments, then loops through each app service in each environment.

- `template-repo/azure-appservice-deploy-step.yml`  
  Nested step template that contains the single `AzureWebApp@1` task. This keeps task usage centralized and avoids duplicate task definitions in the parent template.

## How the lifecycle works

1. `someapp-repo/azure-pipelines.yml` imports the stage template from the shared template repo alias (`templatesRepo`).
2. The stage template iterates through `environments` (for example: `dev`, `prod`).
3. For each environment, it creates deployment jobs for each app service listed in `appServices`.
4. Each deployment job calls the nested step template exactly once.
5. The nested step template runs `AzureWebApp@1` with the deployment method selected by environment type:
   - non-prod -> `zipDeploy`
   - prod -> `runFromPackage`

## Parameter model

The caller pipeline provides these key values:

- `artifactName`: artifact to download (default example: `drop`)
- `packagePath`: package location in the artifact (default example: `$(Pipeline.Workspace)/drop/app.zip`)
- `vmImage`: agent image (default example: `ubuntu-latest`)
- `environments`: object array defining lifecycle stages and app-service targets

Each environment entry contains:

- `name`
- `displayName`
- `isProd`
- `dependsOn`
- `azureServiceConnection`
- `environmentResource`
- `appServices` (array of app names)

## Why this structure

- Keeps lifecycle logic DRY
- Supports multiple environments and multiple app services
- Preserves clear prod vs non-prod behavior
- Centralizes the deploy task in one nested file for easier maintenance

## Adopting this pattern

For a new app team repo:

1. Copy/adapt `someapp-repo/azure-pipelines.yml`.
2. Point `resources.repositories.name` to your shared template repo.
3. Set service connections and app names for your environments.
4. Keep the reusable templates in the shared repo (for this example: `template-repo/`).
