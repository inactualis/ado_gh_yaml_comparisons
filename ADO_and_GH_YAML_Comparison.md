### Comparing Azure DevOps and GitHub YAML Constructs

This repo contains examples of two lifecycles that highlight YAML elements from Azure Devps (ADO) that are not yet present in GitHub (GH) Actions, namely:
- Stages
- Control loops (foreach and forif)
- Object Parameters

The examples lay out a common enterprise-scale pattern in both ADO and GH. This type of pattern is typically employed at organizations where a central team is tasked with ensuring that all automation adheres to corporate Standards and, often, external regulations. This is common in highly regulated industries like Finance and Energy, or any enterprise with rigid governance and compliance requirements for automation. 

Note the following presumptions about the principles we value for this example: 

- Do Not Repeat Yourself - All templates and workflows should be reusable and avoid duplication across environments and repositories.

- Inline Code Should be Avoided Entirely - All code should be encapsulated to ensure maintainability and consistency across environments. 

- Single Source of Truth - Configuration and deployment logic should be centralized to ensure consistency and ease of maintenance.

- Environment Parity - Non-production and production environments should follow the same deployment patterns to reduce risk and increase reliability.

- Explicit Dependencies - Stages and jobs should declare dependencies to control execution order and ensure correct sequencing.

- Parameterization - Inputs should be parameterized to allow flexibility and reuse across different environments and applications.

A comment on the second bullet point. This principle implies a heavy prejudice for centralized Tasks/Actions. Regardless of an organization's skill level, scripting in YAMLs leads to more difficult maintenance, lengthier troubleshooting, and more fragile pipelines. This can make it difficultlt to manage changes and ensure consistent behavior across environments. It also requires a different skill set than employed by most DevOps Engineers.

## Issues with GitHub Actions Pattern
The following sections highlight specific challenges and limitations that arise when using GitHub Actions for this deployment pattern. We'll map these challenges to the priniciples above and discuss how they manifest in the GitHub Actions pattern.

### The GH Actions Require JSON
Let's start by comparing the smallest elements in each lifecycle: the GH Composite Action (action.yaml) versus the ADO step template (azure-appservice-deploy-step.yml). The first thing to notice is that the ADO step template can directly reference environment parameters and other context, whereas the GH reusable workflow and Composite Action rely on serialized JSON payloads and runtime extraction. While this may look manageable in this simple example, the human readability of the GH file quickly degrades as the complexity of the environment and the number of parameters increase. Simply put, JSON in YAML (GH) is more complex than just YAML (ADO). 

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
Because GitHub Actions do not provide the same template-time looping and object traversal model used in ADO templates, the Composite Action uses inline shell + `jq` to iterate over app service definitions. Again, this is a very simple example that will spiral into complex and hard-to-maintain scripts as the number of app services and deployment logic grows.

In Azure DevOps, the equivalent behavior is handled by template expansion + task inputs, so the deployment logic stays declarative and avoids inline scripting in the pipeline/template itself.

**ADO nested step template (`azure-appservice-deploy-step.yml`) — no inline loop/script required:**

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
    displayName: Deploy package to ${{ parameters.appName }}
    inputs:
      azureSubscription: ${{ parameters.azureServiceConnection }}
      appName: ${{ parameters.appName }}
      package: ${{ parameters.packagePath }}
      deploymentMethod: ${{ parameters.deploymentMethod }}
```

**GitHub Composite Action (`action.yml`) — inline script to parse JSON and loop app services:**

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

In Azure DevOps, templates can accept structured Object parameters that allow direct access to nested properties. GitHub Actions does not support Object parameters in the same way; instead, JSON strings are passed and parsed at runtime using `fromJSON(...)` as shown in the examples above. Let's compare azure-appsevice-deploy-stages.yml to deploy-appservices.yml to see how much cleaner our code is with the ADO Object versus the JSON parsing approach in GitHub Actions.

**Azure DevOps (`azure-appservice-deploy-stages.yml`) — native object parameter:**

```yaml
parameters:
  - name: environments
    type: object

stages:
  - ${{ each env in parameters.environments }}:
      - stage: ${{ env.stageName }}
        displayName: Deploy ${{ env.displayName }}
        dependsOn: ${{ env.dependsOn }}
        jobs:
          - ${{ each appService in env.appServices }}:
              - deployment: ${{ appService.deploymentName }}
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
          stageName: Deploy_dev
          displayName: Development
          isProd: false
          dependsOn: []
          azureServiceConnection: sc-azure-dev
          environmentResource: dev
          appServices:
            - name: someapp-web-dev-01
              deploymentName: Deploy_someapp_web_dev_01
        - name: prod
          stageName: Deploy_prod
          displayName: Production
          isProd: true
          dependsOn:
            - Deploy_dev
          azureServiceConnection: sc-azure-prod
          environmentResource: prod
          appServices:
            - name: someapp-web-prod-01
              deploymentName: Deploy_someapp_web_prod_01
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

### GitHub Actions Cannot use Control Loops ###

Azure DevOps supports template-time control loops (`each`) and conditional blocks (`if`) directly in YAML templates. That lets one template scale across many environments and app services without duplicating job definitions.

GitHub Actions does not have an equivalent template-time loop model inside workflow YAML. The practical impacts are:

- **More duplication**: environment jobs are often repeated with only small differences.
- **Harder maintenance**: changes to shared logic must be made in multiple places unless moved into a Composite Action or reusable workflow.
- **Less expressive hierarchy**: nested loop patterns (environment → app service) are less natural and often require JSON parsing, matrices, or inline script.
- **Dependency complexity**: per-environment ordering and per-app fan-out/fan-in are more manual than ADO template expansion.

**Azure DevOps control loops (`azure-appservice-deploy-stages.yml`) — native nested loops:**

```yaml
stages:
  - ${{ each env in parameters.environments }}:
      - stage: ${{ env.stageName }}
        dependsOn: ${{ env.dependsOn }}
        jobs:
          - ${{ each appService in env.appServices }}:
              - deployment: ${{ appService.deploymentName }}
                environment: ${{ env.environmentResource }}
```

**GitHub Actions common fallback — explicit jobs per environment:**

```yaml
jobs:
  deploy_dev:
    if: ${{ fromJSON(inputs.environments_json).dev.enabled }}
    # ... repeated steps

  deploy_prod:
    needs: deploy_dev
    if: ${{ fromJSON(inputs.environments_json).prod.enabled }}
    # ... mostly repeated steps
```

**GitHub Actions matrix alternative — useful in some cases:**

```yaml
jobs:
  deploy:
    strategy:
      matrix:
        environment: [dev, prod]
    runs-on: ubuntu-latest
    steps:
      - name: Deploy selected environment
        uses: ./.github/actions/deploy-appservices
        with:
          environment_name: ${{ matrix.environment }}
          is_prod: ${{ fromJSON(inputs.environments_json)[matrix.environment].is_prod }}
          resource_group: ${{ fromJSON(inputs.environments_json)[matrix.environment].resource_group }}
          app_services_json: ${{ toJSON(fromJSON(inputs.environments_json)[matrix.environment].app_services) }}
```

Is the GH matrix construct a good alternative? Sometimes, but typically not at enterprise-scale, where strict promotion order and environment-specific controls are required. You must reposrt to multuple, duplicative matrices, where each matrix row becomes a sibling job instance. You can’t rely on a single matrix todefine per-row dependencies like:

test-* depends on all dev-*
prod-* depends on all test-*

- **Good fit** when environments are mostly identical and can run in parallel.
- **Bad fit** when you need strict promotion order (dev → test → prod), environment-specific approvals/gates, or deeply nested lifecycle logic. In those cases, matrix often shifts complexity into conditions and JSON lookups rather than truly reducing it. Consider this matrix that attempts to sequence environments via conditions rather than explicit job dependencies: 



### GitHub Actions Lack the Stages Construct ###

In Azure DevOps, Stages provide a natural way to sequence deployments across environments (e.g., dev → prod) and manage dependencies. GitHub Actions does not have a native Stages construct. Instead, sequencing is achieved using jobs with `needs` relationships. Each environment deployment becomes a separate job, and the order is enforced via `needs`. We've discussed this above as well, where the the matrix option leads to violations of our key principles: we are forced to create multiple, duplicative jobs and matrices for each. 

**Azure DevOps (`azure-appservice-deploy-stages.yml`) — stage-level lifecycle:**

```yaml
stages:
  - ${{ each env in parameters.environments }}:
      - stage: ${{ env.stageName }}
        dependsOn: ${{ env.dependsOn }}
        jobs:
          - deployment: ${{ appService.deploymentName }}
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

Again, remember that this is a very simple example of the challenge. The example above lessens the concern by referencing a Composite Action, but this is where a lot of organizations create cascading complexity: it is common to see Actions defined repeatedly under each job (rather than taking the extra step of encapsulating the work in a Composite Action). We wind up violating several of our core principles to approximate the ADO Stage construct, and the problems this creates are substantial and meaningul at scale. 

**Most Common Pattern Seen in GitHub Actions (`deploy-appservices.yml`):**
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

### Official documentation

**Azure DevOps**
- YAML schema: https://learn.microsoft.com/azure/devops/pipelines/yaml-schema/?view=azure-pipelines
- Template parameters (`string`, `object`, etc.): https://learn.microsoft.com/azure/devops/pipelines/process/template-parameters?view=azure-devops
- Template expressions and `each` loops: https://learn.microsoft.com/azure/devops/pipelines/process/template-expressions?view=azure-devops
- Stages: https://learn.microsoft.com/azure/devops/pipelines/process/stages?view=azure-devops
- Deployment jobs: https://learn.microsoft.com/azure/devops/pipelines/process/deployment-jobs?view=azure-devops
- `AzureWebApp@1` task: https://learn.microsoft.com/azure/devops/pipelines/tasks/reference/azure-web-app-v1?view=azure-pipelines

**GitHub Actions**
- Workflow syntax: https://docs.github.com/actions/using-workflows/workflow-syntax-for-github-actions
- Reusing workflows (`workflow_call`): https://docs.github.com/actions/using-workflows/reusing-workflows
- Composite Actions: https://docs.github.com/actions/creating-actions/creating-a-composite-action
- Expressions and `fromJSON(...)`: https://docs.github.com/actions/learn-github-actions/expressions
- Job dependencies (`needs`): https://docs.github.com/actions/using-jobs/using-jobs-in-a-workflow
- Environments: https://docs.github.com/actions/deployment/targeting-different-environments/using-environments-for-deployment
- `actions/download-artifact`: https://github.com/actions/download-artifact
- `azure/login`: https://github.com/Azure/login

**Azure CLI deployment commands**
- `az webapp deploy`: https://learn.microsoft.com/cli/azure/webapp?view=azure-cli-latest#az-webapp-deploy
- `az webapp deployment source config-zip`: https://learn.microsoft.com/cli/azure/webapp/deployment/source?view=azure-cli-latest#az-webapp-deployment-source-config-zip

## File comparison matrix

| Layer / Purpose | Azure DevOps Example | GitHub Actions Example | How they match |
|---|---|---|---|
| **Consumer entrypoint** | `ADO Example/someapp-repo/someapp-automation.yml` | `GitHub Example/someapp-repo/someapp-automation.yml` | App-team-owned workflow/pipeline that kicks off deployment and passes environment/app configuration. |
| **Reusable orchestrator** | `ADO Example/template-repo/azure-appservice-deploy-stages.yml` | `GitHub Example/.github/workflows/deploy-appservices.yml` | Centralized reusable lifecycle definition (dev → prod sequencing, per-environment orchestration). |
| **Reusable deploy unit** | `ADO Example/template-repo/azure-appservice-deploy-step.yml` | `GitHub Example/.github/actions/deploy-appservices/action.yml` | **Direct equivalent**: ADO step template ↔ GitHub Composite Action. Both encapsulate per-app deployment logic. |

## Capability mapping

| Capability | Azure DevOps Pattern | GitHub Actions Pattern |
|---|---|---|
| Multi-environment lifecycle | `stages` generated from environment objects | Multiple jobs (`deploy_dev`, `deploy_prod`) with conditional execution |
| Execution order | `dependsOn` on generated stages | `needs` between jobs |
| Reuse across repos | Template repo via `resources.repositories` + `template:` | Reusable workflow via `uses: org/repo/.github/workflows/...@ref` |
| Reusable task/action unit | Nested `step` template | Composite Action |
| Structured environment input | YAML object parameters (`environments`) | JSON string input (`environments_json`) parsed with `fromJSON(...)` |
| Prod vs non-prod deploy mode | Template condition sets `deploymentMethod` (`runFromPackage` / `zipDeploy`) | Composite Action branches to different Azure CLI commands based on `is_prod` |