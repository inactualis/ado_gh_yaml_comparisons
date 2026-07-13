# GitHub Actions example

This folder mirrors the ADO lifecycle using GitHub Actions building blocks.

It includes all three layers:

- caller workflow in an app repo
- Reusable Workflow
- Composite Action

## Files

- `someapp-repo/someapp-automation.yml`  
  Consumer workflow in an app team's repo. It calls a Reusable Workflow with `uses:` and passes lifecycle inputs, including `environments_json`.

- `.github/workflows/deploy-appservices.yml`  
  Reusable Workflow invoked via `workflow_call`. It defines required Azure OIDC secrets, then runs `deploy_dev` and `deploy_prod` jobs.

- `.github/actions/deploy-appservices/action.yml`  
  Composite Action that loops through app services and deploys each app using Azure CLI.

## Lifecycle behavior

1. `someapp-repo/someapp-automation.yml` calls the Reusable Workflow.
2. Reusable Workflow reads `environments_json` and conditionally runs:
   - `deploy_dev`
   - `deploy_prod` (with `needs: deploy_dev`)
3. Each job downloads artifact, logs into Azure with OIDC (`azure/login@v2`), and calls the Composite Action.
4. The Composite Action deploys each app service from JSON input:
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

> Note: in this current sample implementation, `depends_on` may be included in the payload for parity/context, but job ordering is defined directly in workflow YAML with `needs` (for example, `deploy_prod` needs `deploy_dev`).

## Required secrets

The Reusable Workflow expects:

- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID`

The sample consumer workflow uses `secrets: inherit`.

## ADO parity notes

- ADO uses object parameters and template expansion loops.
- GitHub example uses JSON input parsed at runtime (`fromJSON(...)`).
- Both preserve the same dev → prod lifecycle and multi-app-service deploy pattern.
