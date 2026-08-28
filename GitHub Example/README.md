# GitHub Actions example

This folder mirrors the ADO build-and-deploy lifecycle using GitHub Actions building blocks.

It includes all three layers:

- caller workflow in an app repo
- Reusable Workflows for build and deployment orchestration
- Composite Actions for reusable build and deployment behavior

## Files

- `someapp-repo/someapp-automation.yml`  
  Consumer workflow in an app team's repo. It calls the build workflow first, then calls the deployment workflow with `needs: build`.

- `.github/workflows/build-net.yml`
  Reusable Workflow that checks out the app source, invokes the centralized .NET build action, and uploads one deployment artifact.

- `.github/workflows/deploy-az-appservice.yml`
  Reusable Workflow invoked via `workflow_call`. It defines required Azure OIDC secrets, then runs `deploy_dev` and `deploy_prod` jobs.

- `.github/actions/build/net/action.yml`
  Composite Action that restores, builds, publishes, and packages a .NET application.

- `.github/actions/deploy/az-appservice/action.yml`
  Composite Action that loops through app services and deploys each app using Azure CLI.

Reusable workflow files remain directly under `.github/workflows` because GitHub only discovers workflow files at that level. Composite Actions do not have that restriction, so they are grouped under `build/net` and `deploy/az-appservice` to parallel the ADO template structure.

## Lifecycle behavior

1. `someapp-repo/someapp-automation.yml` calls `build-net.yml`.
2. The build workflow creates and uploads `drop/app.zip` once.
3. The consumer's deployment job uses `needs: build`, then calls `deploy-az-appservice.yml`.
4. The deployment workflow reads `environments_json` and conditionally runs:
   - `deploy_dev`
   - `deploy_prod` (with `needs: deploy_dev`)
5. Each deployment job downloads the same artifact, logs into Azure with OIDC (`azure/login@v2`), and calls the Composite Action.
6. The Composite Action deploys each app service from JSON input:
   - non-prod uses `az webapp deployment source config-zip`
   - prod uses `az webapp deploy --type zip`

## Build input model

The build workflow accepts:

- `sdk_version`
- `project_path`
- `artifact_name`
- `package_name`
- `vm_image`

Builds always use the `Release` configuration. The application-specific paths stay in the app-owned caller while build behavior remains centralized.

## Deployment input model

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

Replace `your-org/shared-workflows` in the consumer with the owner and repository containing these reusable workflows and actions. Within each reusable workflow, the `$/` prefix resolves its Composite Action from that shared repository at the workflow's running commit; unlike `./`, it does not resolve against the app repository's checked-out workspace. The `$/` syntax is not available in GitHub Enterprise Server.

## ADO parity notes

- ADO uses object parameters and template expansion loops.
- GitHub example uses JSON input parsed at runtime (`fromJSON(...)`).
- ADO can place templates in arbitrary nested folders; GitHub reusable workflows must stay directly under `.github/workflows`.
- Both build once and preserve the same build → dev → prod lifecycle and multi-app-service deployment pattern.
