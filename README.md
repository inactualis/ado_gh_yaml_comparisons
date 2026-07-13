### Comparing Azure DevOps and GitHub YAML Constructs

> **TL;DR:** ADO gives native Stages + Control Loops + structured Object parameters; GitHub can approximate these features with JSON, matrices, and Composite Actions, but usually with more complexity and duplication.

This repo contains examples of two lifecycles that highlight YAML elements from Azure DevOps (ADO) that are not yet present in GitHub (GH) Actions, namely:
- Stages
- Control loops (foreach and forif)
- Object Parameters

The examples lay out a common enterprise-scale pattern in both ADO and GH. This type of pattern is typically employed at organizations where a central team is tasked with ensuring that all automation adheres to corporate Standards and, often, external regulations. This is common in highly regulated industries like Finance and Energy, or any enterprise with rigid governance and compliance requirements for automation. 

Note the following presumptions about the principles we value for this example: 

- **Do Not Repeat Yourself** - All templates and workflows should be reusable and avoid duplication across environments and repositories.

- **Inline Code Should be Avoided Entirely** - All code should be encapsulated to ensure maintainability and consistency across environments. 

- **Single Source of Truth** - Configuration and deployment logic should be centralized to ensure consistency and ease of maintenance.

- **Environment Parity** - Non-production and production environments should follow the same deployment patterns to reduce risk and increase reliability.

- **Explicit Dependencies** - Stages and jobs should declare dependencies to control execution order and ensure correct sequencing.

- **Parameterization** - Inputs should be parameterized to allow flexibility and reuse across different environments and applications.

A comment on the second bullet point. This principle implies a heavy prejudice for centralized Tasks/Actions. Regardless of an organization's skill level, scripting in YAMLs leads to more difficult maintenance, lengthier troubleshooting, and more fragile pipelines. This can make it difficultlt to manage changes and ensure consistent behavior across environments. It also requires a different skill set than employed by most DevOps Engineers.

## The GitHub Actions Pattern has Some Challenges
The following sections highlight specific issues and limitations that arise when using GitHub Actions for this exact deployment pattern. We'll map these challenges to the priniciples above and discuss how they manifest with Actions.

### GH Actions Often Require JSON
Let's start by comparing the smallest elements in each lifecycle: the GH Composite Action (action.yaml) versus the ADO step template (azure-appservice-deploy-step.yml). The first thing to notice is that the ADO step template can directly reference environment parameters and other context, whereas the GH Reusable Workflow and Composite Action rely on serialized JSON payloads and runtime extraction. While this may look manageable in this simple example, the human readability of the GH file quickly degrades as the complexity of the environment and the number of parameters increase. Simply put, JSON in YAML (GH) is more complex than just YAML (ADO). 

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

### GH Actions Often Require Inline Code
Because GitHub Actions do not provide the same template-time looping and object traversal model used in ADO templates, the Composite Action uses inline shell + `jq` to iterate over app service definitions. In Azure DevOps, the equivalent behavior is handled by template expansion + task inputs, so the deployment logic stays declarative and avoids inline scripting in the pipeline/template itself.

Again, this is a very simple example, and single-target scenarios can often rely on a default App Service Action. In this pattern, however, we still need iteration across multiple app services and environment-specific branching logic. Without centralizing that behavior in a reusable component (like a Composite Action), teams typically end up reintroducing inline scripting and duplicated YAML.

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
  environment_name:
    required: true
  is_prod:
    required: true
  resource_group:
    required: true
  artifact_path:
    required: false
    default: artifact
  package_path:
    required: false
    default: app.zip
  app_services_json:
    required: true

runs:
  using: composite
  steps:
    - shell: bash
      env:
        ENVIRONMENT_NAME: ${{ inputs.environment_name }}
        IS_PROD: ${{ inputs.is_prod }}
        RESOURCE_GROUP: ${{ inputs.resource_group }}
        ARTIFACT_PATH: ${{ inputs.artifact_path }}
        PACKAGE_PATH: ${{ inputs.package_path }}
        APP_SERVICES_JSON: ${{ inputs.app_services_json }}
      run: |
        set -euo pipefail
        package_file="${ARTIFACT_PATH}/${PACKAGE_PATH}"
        echo "Deploying to ${ENVIRONMENT_NAME} using package ${package_file}"

        echo "${APP_SERVICES_JSON}" | jq -c '.[]' | while read -r service; do
          app_name="$(echo "${service}" | jq -r '.name')"
          if [ "${IS_PROD}" = "true" ]; then
            az webapp deploy \
              --resource-group "${RESOURCE_GROUP}" \
              --name "${app_name}" \
              --src-path "${package_file}" \
              --type zip \
              --async false
          else
            az webapp deployment source config-zip \
              --resource-group "${RESOURCE_GROUP}" \
              --name "${app_name}" \
              --src "${package_file}"
          fi
        done
```
### GitHub Actions Lack the Object Parameter Model ###

In Azure DevOps, templates can accept structured Object parameters that allow direct access to nested properties. GitHub Actions does not support Object parameters in the same way; instead, JSON strings are passed and parsed at runtime. This adds verbosity and can increase maintenance overhead, because authors must manage JSON serialization/deserialization and expression paths in addition to deployment logic.

Let's compare azure-appsevice-deploy-stages.yml to deploy-appservices.yml to see how much cleaner our code is with the ADO Object versus the JSON parsing approach in GitHub Actions.

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
If we go one level higher, we see how the consumer workflows are impacted by this difference. Github must pass JSON strings to the Reusable Workflow, which then parses those strings at runtime to access environment-specific properties. The need for inline json here again leads to more verbose and difficult to maintain YAML at every level in our template stack. And remember that we are looking at the simplest example: this issue compounds with the complexity of the lifecycle. 

This contrasts with Azure DevOps, where the object parameter allows direct access to nested properties without runtime parsing. The Object is much easier for humans to read and write, while JSON requires careful formatting and diligence to maintain. 

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

### GitHub Actions Lack ADO-Style Template-Time Control Loops ###

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

Also notice that the GH matrix approach is often less suitable where strict promotion order and environment-specific controls are required. In many implementations, teams end up with multiple, duplicative matrices, where each matrix row becomes a sibling job instance. In practice, you generally can’t rely on a single matrix to define per-row dependencies like:

test-* depends on all dev-*
prod-* depends on all test-*

- **Matrices are a good fit** when environments are mostly identical and can run in parallel. Examples might include IAC to sibling environments. 
- **But they're a bad fit** when you need strict promotion order (dev → test → prod), environment-specific approvals/gates, or deeply nested lifecycle logic. In those cases, matrix often shifts complexity into conditions and JSON lookups rather than truly reducing it. 

Consider the complexity of this basic example that attempts to implement environment promotion order using a matrix; this would be a single "dependson" property in the object we pass to our ADO Stages model. In GH, it would be more readable with independent jobs and explicit `needs` relationships: 

**Example of matrix complexity shifting into conditions + JSON lookups:**

```yaml
jobs:
  deploy_by_environment:
    strategy:
      matrix:
        env: [dev, test, prod]
    runs-on: ubuntu-latest

    # Gating is now encoded in conditional logic, not stage orchestration
    if: ${{
      fromJSON(inputs.environments_json)[matrix.env].enabled &&
      (matrix.env != 'prod' || github.ref == 'refs/heads/main')
    }}

    environment: ${{ fromJSON(inputs.environments_json)[matrix.env].environment_resource }}

    steps:
      - name: Deploy matrix environment
        uses: ./.github/actions/deploy-appservices
        with:
          environment_name: ${{ matrix.env }}
          is_prod: ${{ fromJSON(inputs.environments_json)[matrix.env].is_prod }}
          resource_group: ${{ fromJSON(inputs.environments_json)[matrix.env].resource_group }}
          app_services_json: ${{ toJSON(fromJSON(inputs.environments_json)[matrix.env].app_services) }}
```
What's worse, this overly complex snippet will not actually work for true gated promotion order. Matrix rows are sibling job runs, and GitHub does not provide per-row `needs` dependencies (for example, "all `test` rows must wait for all `dev` rows"). The `if` expression can filter runs, but it cannot create a deterministic dependency chain between matrix values. 

In practice, this means `dev`, `test`, and `prod` matrix rows may still be scheduled as peers, and environment/branch checks only gate individual rows, not stage-to-stage progression. Again, to enforce strict promotion (`dev -> test -> prod`), you need separate jobs with explicit `needs` relationships, optionally with a matrix inside each environment job for fan-out.

### GitHub Actions Lack the Stages Construct ###

In Azure DevOps, Stages provide a natural way to sequence deployments across environments (e.g., dev → prod) and manage dependencies. We define a Stage once, use our Object type parameter to control which environments are included, and then iterate over that as many times as needed. And since the Object parameter type is structured, we can easily access environment-specific properties (like `dependsOn`, `isProd`, etc.) within the stage definition.

In GitHub, sequencing is achieved using jobs with `needs` relationships. Since we don't have access to ADO's Stages, each environment deployment becomes a separate job. As we discussed above, this challenge begins to compound if we try to employ GitHub's matrix construct.

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

Again, remember that this is a very simple example of the challenge. The example above lessens the concern by referencing a Composite Action, but this is where a lot of organizations create cascading complexity: it is common to see Actions defined repeatedly under each job (rather than taking the extra step of encapsulating the work in a Composite Action). Notice how the inline code is not repeated. The problems this creates are substantial and meaningul at scale. 

**GitHub Pattern Without Composite Action (`deploy-appservices.yml`):**
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

      - name: Deploy development app services (inline)
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

      - name: Deploy production app services (inline)
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