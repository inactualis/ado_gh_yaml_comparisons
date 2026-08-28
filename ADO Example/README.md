# ADO Example

This folder demonstrates a DRY Azure DevOps build-and-deploy lifecycle for a .NET application hosted by Azure App Service, using separate app and template repos.

## Repository-style layout

- `someapp-repo/` — app team consumer repo example
- `template-repo/build/net/` — shared .NET build templates
- `template-repo/deploy/` — shared deployment templates grouped by target platform

## Files

- `someapp-repo/someapp-automation.yml`  
  Consumer pipeline that lives in the app repo. It imports shared build and deployment stage templates via `resources.repositories` and passes lifecycle parameters.

- `template-repo/build/net/net-build-stage.yml`
  Generic reusable build stage that exposes SDK, source, artifact, and agent parameters.

- `template-repo/build/net/net-build-steps.yml`
  Nested build steps that restore, build, publish, package, and publish one deployable artifact.

- `template-repo/deploy/az-appservice/azure-appservice-deploy-stages.yml`
  Reusable stage template. It loops through environments, then loops through app services within each environment.

- `template-repo/deploy/az-appservice/azure-appservice-deploy-step.yml`
  Nested step template containing the single `AzureWebApp@1` task.

## How it works

1. `someapp-repo/someapp-automation.yml` references the shared repo alias (`templatesRepo`).
2. It calls `build/net/net-build-stage.yml@templatesRepo` to build and publish `drop/app.zip` once.
3. It calls `deploy/az-appservice/azure-appservice-deploy-stages.yml@templatesRepo` for the deployment lifecycle.
4. `Deploy_dev` depends on `Build`, and `Deploy_prod` depends on `Deploy_dev`, so the same artifact is promoted in order.
5. The deployment stage template iterates over `parameters.environments`, then creates deployment jobs for each app service.
6. Each deployment job calls the nested step template, which runs `AzureWebApp@1` using:
   - non-prod: `zipDeploy`
   - prod: `runFromPackage`

## Build parameter model

The caller passes generic build inputs:

- `stageName`
- `displayName`
- `vmImage`
- `sdkVersion`
- `projectPath`
- `artifactName`
- `packageName`

Builds always use the `Release` configuration. Only the folder and SDK parameter identify this as a .NET implementation; application-specific paths remain caller-owned.

## Deployment parameter model

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

- The sample consumer file uses `trigger: none` and `pr: none`.
- The application is built once; every environment receives the same immutable package.
- Stage sequencing is controlled per environment through `dependsOn`.
- Build and deployment task behavior remain centralized and free of consumer-owned inline scripts.

## Adopting this pattern

1. Copy/adapt `someapp-repo/someapp-automation.yml` in each app repo.
2. Point `resources.repositories.name` to your shared template repo.
3. Set the SDK project path when the project is not at the repository root.
4. Define your environment entries (`dev`, `prod`, etc.) and app service names.
5. Keep reusable templates centralized in the shared repo.
