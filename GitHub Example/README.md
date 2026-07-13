# GitHub Actions example

This folder mirrors the ADO lifecycle using GitHub Actions building blocks.

It includes all three layers:

- caller workflow in an app repo
- reusable workflow
- composite action

## Files

- `someapp-repo/someapp-automation.yml`  
  Consumer workflow in an app team's repo. It calls a reusable workflow with `uses:` and passes lifecycle inputs, including `environments_json`.

- `.github/workflows/deploy-appservices.yml`  
  Reusable workflow invoked via `workflow_call`. It defines required Azure OIDC secrets, then runs `deploy_dev` and `deploy_prod` jobs.

- `.github/actions/deploy-appservices/action.yml`  
  Composite action that loops through app services and deploys each app using Azure CLI.

## Lifecycle behavior

1. `someapp-repo/someapp-automation.yml` calls the reusable workflow.
2. Reusable workflow reads `environments_json` and conditionally runs:
   - `deploy_dev`
   - `deploy_prod` (with `needs: deploy_dev`)
3. Each job downloads artifact, logs into Azure with OIDC (`azure/login@v2`), and calls the composite action.
4. The composite action deploys each app service from JSON input:
   - non-prod uses `az webapp deployment source config-zip`
   - prod uses `az webapp deploy --type zip`

## Input model

Caller workflow provides:

- `artifact_name`
- `artifact_path`
- `package_path`
- `vm_image`
- `environments_json`

`environments_json` contains objects such as `dev` and `prod`, each with:

- `display_name`
- `enabled`
- `is_prod`
- `resource_group`
- `environment_resource`
- optional `depends_on`
- `app_services` (array of `{ "name": "..." }`)

## Required secrets

The reusable workflow expects:

- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID`

The sample consumer workflow uses `secrets: inherit`.

## ADO parity notes

- ADO uses object parameters and template expansion loops.
- GitHub example uses JSON input parsed at runtime (`fromJSON(...)`).
- Both preserve the same dev → prod lifecycle and multi-app-service deploy pattern.
