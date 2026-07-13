This repo contains examples of YAML elements in Azure DeVops that are not yet present in GitHub Actions, namely:
- Stages
- Control loops (foreach and forif)
- Object Parameters

The example lays out a typical enterprise-scale pattern for Azure DevOps and a close equivalent in GitHub Actions. Note the following presumtions about the principles we value for this example: 

- Do Not Repeat Yourself - All templates and workflows should be reusable and avoid duplication across environments and repositories.

- Inline Code should be avoided at all costs - All logic should be encapsulated within the Task/Action or reusable component to ensure maintainability and consistency across environments. Scripting in YAMLs leads to heavy maintenance and fragile pipelines, making it difficult to manage changes and ensure consistent behavior across environments.

- Single Source of Truth - Configuration and deployment logic should be centralized to ensure consistency and ease of maintenance.

- Environment Parity - Non-production and production environments should follow the same deployment patterns to reduce risk and increase reliability.

- Explicit Dependencies - Stages and jobs should declare dependencies to control execution order and ensure correct sequencing.

- Parameterization - Inputs should be parameterized to allow flexibility and reuse across different environments and applications.


## Issues with GitHub Actions Pattern
The following sections highlight specific challenges and limitations that arise when using GitHub Actions for this deployment pattern compared to Azure DevOps. We'll map these challenges to the priniciples above and discuss how they manifest in the GitHub Actions pattern.

### GH Actions Require JSON
First let's compare the smallest eleements in each lifecyle: the GH Composite Action (action.yaml) versus the ADO step template (azure-appservice-deploy-step.yml). The first thing to notice is that the ADO step template can directly reference environment parameters and other context, whereas the GH reusable workflow and composite action rely on serialized JSON payloads and runtime extraction.

**ADO step template (`azure-appservice-deploy-step.yml`) — direct parameter binding (no JSON parsing):**

```yaml
parameters:
  - name: azureServiceConnection
    type: string
  - name: appName
    type: string
  - name: packagePath
    type: string
  - name: deploymentMethod
    type: string

steps:
  - task: AzureWebApp@1
    inputs:
      azureSubscription: ${{ parameters.azureServiceConnection }}
      appName: ${{ parameters.appName }}
      package: ${{ parameters.packagePath }}
      deploymentMethod: ${{ parameters.deploymentMethod }}
```

**GitHub reusable workflow (`deploy-appservices.yml`) — JSON extraction with `fromJSON(...)`:**

```yaml
on:
  workflow_call:
    inputs:
      environments_json:
        required: true
        type: string

jobs:
  deploy_dev:
    if: ${{ fromJSON(inputs.environments_json).dev.enabled }}
    environment: ${{ fromJSON(inputs.environments_json).dev.environment_resource }}
    steps:
      - uses: ./.github/actions/deploy-appservices
        with:
          is_prod: ${{ fromJSON(inputs.environments_json).dev.is_prod }}
          resource_group: ${{ fromJSON(inputs.environments_json).dev.resource_group }}
          app_services_json: ${{ toJSON(fromJSON(inputs.environments_json).dev.app_services) }}
```

### GH Actions Require Inline Code
Because GitHub Actions does not provide the same template-time looping and object traversal model used in ADO templates, the composite action uses inline shell + `jq` to iterate over app service definitions.

**GitHub composite action (`action.yml`) — inline script to parse JSON and loop app services:**

```yaml
inputs:
  app_services_json:
    required: true

runs:
  using: composite
  steps:
    - shell: bash
      env:
        APP_SERVICES_JSON: ${{ inputs.app_services_json }}
      run: |
        echo "${APP_SERVICES_JSON}" | jq -c '.[]' | while read -r service; do
          app_name="$(echo "${service}" | jq -r '.name')"
          echo "Deploying ${app_name}"
        done
```
### GitHub Actions Lacks the Object Parameter Model ###

In Azure DevOps, templates can accept structured Object parameters that allow direct access to nested properties. GitHub Actions does not support Object parameters in the same way; instead, JSON strings are passed and parsed at runtime using `fromJSON(...)` as shown in the example above. Let's compare azure-appsevice-deploy-stages.yml to deploy-appservices.yml to see how much cleaner our code is with the ADO Object versus the JSON parsing approach in GitHub Actions.

**Azure DevOps (`azure-appservice-deploy-stages.yml`) — native object parameter:**

```yaml
parameters:
  - name: environments
    type: object

stages:
  - ${{ each env in parameters.environments }}:
      - stage: Deploy_${{ replace(env.name, '-', '_') }}
        displayName: Deploy ${{ env.displayName }}
        dependsOn: ${{ env.dependsOn }}
        jobs:
          - ${{ each appService in env.appServices }}:
              - deployment: Deploy_${{ replace(appService.name, '-', '_') }}
                environment: ${{ env.environmentResource }}
```

**GitHub Actions (`deploy-appservices.yml`) — JSON string + runtime parsing:**

```yaml
on:
  workflow_call:
    inputs:
      environments_json:
        required: true
        type: string

jobs:
  deploy_dev:
    if: ${{ fromJSON(inputs.environments_json).dev.enabled }}
    environment: ${{ fromJSON(inputs.environments_json).dev.environment_resource }}
    steps:
      - uses: ./.github/actions/deploy-appservices
        with:
          is_prod: ${{ fromJSON(inputs.environments_json).dev.is_prod }}
          resource_group: ${{ fromJSON(inputs.environments_json).dev.resource_group }}
          app_services_json: ${{ toJSON(fromJSON(inputs.environments_json).dev.app_services) }}
```
If we go one level higher, we see how the consumer workflows differ. Github passes the JSON string to the Reusable Workflow, which then parses it at runtime to access environment-specific properties. The need for inline json here leads to more verbose and difficult to maintain YAML. Remember that we are looking at an exceedingly simple example here. This issue compounds with the complexity of the lifecycle.   This contrasts with Azure DevOps, where the object parameter allows direct access to nested properties without runtime parsing. The Object is simply much easier for humans to read, while JSON requires careful formatting and diligence to maintain. 

**Azure DevOps consumer (`someapp-automation.yml`) — passes structured object directly:**

```yaml
stages:
  - template: azure-appservice-deploy-stages.yml@templatesRepo
    parameters:
      environments:
        - name: dev
          displayName: Development
          isProd: false
          dependsOn: []
          azureServiceConnection: sc-azure-dev
          environmentResource: dev
          appServices:
            - name: someapp-web-dev-01
        - name: prod
          displayName: Production
          isProd: true
          dependsOn:
            - Deploy_dev
          azureServiceConnection: sc-azure-prod
          environmentResource: prod
          appServices:
            - name: someapp-web-prod-01
```

**GitHub consumer (`someapp-automation.yml`) — passes serialized JSON string:**

```yaml
jobs:
  deploy:
    uses: your-org/shared-workflows/.github/workflows/deploy-appservices.yml@main
    with:
      environments_json: |
        {
          "dev": {
            "display_name": "Development",
            "enabled": true,
            "is_prod": false,
            "environment_resource": "dev",
            "app_services": [{ "name": "someapp-web-dev-01" }]
          },
          "prod": {
            "display_name": "Production",
            "enabled": true,
            "is_prod": true,
            "environment_resource": "prod",
            "depends_on": ["deploy_dev"],
            "app_services": [{ "name": "someapp-web-prod-01" }]
          }
        }
```




### GitHub Actions Lack the Stages Construct ###

In Azure DevOps, Stages provide a natural way to sequence deployments across environments (e.g., dev → prod) and manage dependencies. GitHub Actions does not have a native stages construct. Instead, sequencing is achieved using jobs with `needs` relationships. Each environment deployment becomes a separate job, and the order is enforced via `needs`. 

**Azure DevOps (`azure-appservice-deploy-stages.yml`) — stage-level lifecycle:**

```yaml
stages:
  - ${{ each env in parameters.environments }}:
      - stage: Deploy_${{ replace(env.name, '-', '_') }}
        dependsOn: ${{ env.dependsOn }}
        jobs:
          - deployment: Deploy_${{ replace(appService.name, '-', '_') }}
```

**GitHub Actions (`deploy-appservices.yml`) — job sequencing with `needs`:**

```yaml
jobs:
  deploy_dev:
    if: ${{ fromJSON(inputs.environments_json).dev.enabled }}

  deploy_prod:
    needs: deploy_dev
    if: ${{ fromJSON(inputs.environments_json).prod.enabled }}
```

Again, remember that this is a very simple example of the concern. The GitHub Yaml requires that we write each job out separately (as opposed to a single stage with multiple deployments in Azure DevOps). Here we are required to violate our DRY principle and repeat the job definitions for each environment, even though the logic is largely the same. THis example remediates the concern by referencing a composite, but in practice this is where a lot of organization create cascading compleixty. It is common to see the actions define under each job rather than taking the extra step of encacapstulaing the work in a composite. That looks like this: 

```yaml
jobs:
  deploy_dev:
    runs-on: ${{ inputs.vm_image }}
    if: ${{ fromJSON(inputs.environments_json).dev.enabled }}
    environment: ${{ fromJSON(inputs.environments_json).dev.environment_resource }}
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: ${{ inputs.artifact_name }}
          path: ${{ inputs.artifact_path }}

      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy dev app services (inline)
        shell: bash
        run: |
          set -euo pipefail
          package_file="${{ inputs.artifact_path }}/${{ inputs.package_path }}"
          echo '${{ toJSON(fromJSON(inputs.environments_json).dev.app_services) }}' | jq -c '.[]' | while read -r service; do
            app_name="$(echo "${service}" | jq -r '.name')"
            az webapp deployment source config-zip \
              --resource-group "${{ fromJSON(inputs.environments_json).dev.resource_group }}" \
              --name "${app_name}" \
              --src "${package_file}"
          done

  deploy_prod:
    runs-on: ${{ inputs.vm_image }}
    needs: deploy_dev
    if: ${{ fromJSON(inputs.environments_json).prod.enabled }}
    environment: ${{ fromJSON(inputs.environments_json).prod.environment_resource }}
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: ${{ inputs.artifact_name }}
          path: ${{ inputs.artifact_path }}

      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy prod app services (inline)
        shell: bash
        run: |
          set -euo pipefail
          package_file="${{ inputs.artifact_path }}/${{ inputs.package_path }}"
          echo '${{ toJSON(fromJSON(inputs.environments_json).prod.app_services) }}' | jq -c '.[]' | while read -r service; do
            app_name="$(echo "${service}" | jq -r '.name')"
            az webapp deploy \
              --resource-group "${{ fromJSON(inputs.environments_json).prod.resource_group }}" \
              --name "${app_name}" \
              --src-path "${package_file}" \
              --type zip \
              --async false
          done
```



##  Reference ##

## File comparison matrix

| Layer / Purpose | Azure DevOps Example | GitHub Actions Example | How they match |
|---|---|---|---|
| **Consumer entrypoint** | `ADO Example/someapp-repo/someapp-automation.yml` | `GitHub Example/someapp-repo/someapp-automation.yml` | App-team-owned workflow/pipeline that kicks off deployment and passes environment/app configuration. |
| **Reusable orchestrator** | `ADO Example/template-repo/azure-appservice-deploy-stages.yml` | `GitHub Example/.github/workflows/deploy-appservices.yml` | Centralized reusable lifecycle definition (dev → prod sequencing, per-environment orchestration). |
| **Reusable deploy unit** | `ADO Example/template-repo/azure-appservice-deploy-step.yml` | `GitHub Example/.github/actions/deploy-appservices/action.yml` | **Direct equivalent**: ADO step template ↔ GitHub composite action. Both encapsulate per-app deployment logic. |

## Capability mapping

| Capability | Azure DevOps Pattern | GitHub Actions Pattern |
|---|---|---|
| Multi-environment lifecycle | `stages` generated from environment objects | Multiple jobs (`deploy_dev`, `deploy_prod`) with conditional execution |
| Execution order | `dependsOn` on generated stages | `needs` between jobs |
| Reuse across repos | Template repo via `resources.repositories` + `template:` | Reusable workflow via `uses: org/repo/.github/workflows/...@ref` |
| Reusable task/action unit | Nested `step` template | Composite action |
| Structured environment input | YAML object parameters (`environments`) | JSON string input (`environments_json`) parsed with `fromJSON(...)` |
| Prod vs non-prod deploy mode | Template condition sets `deploymentMethod` (`runFromPackage` / `zipDeploy`) | Composite action branches to different Azure CLI commands based on `is_prod` |