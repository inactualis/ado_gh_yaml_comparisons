### Comparing Azure DevOps and GitHub YAML Constructs

> **TL;DR:** ADO gives native Stages + template-time loop patterns (built with `each` + `if`) + structured Object parameters; GitHub can approximate these features with JSON, matrices, and Composite Actions, but usually with more complexity and duplication.

This repo contains examples of two lifecycles that highlight YAML elements from Azure DevOps (ADO) that are not yet present in GitHub (GH) Actions, namely:
- Stages
- Template-time loop patterns (built with `each` and `if` template expressions)
- Object Parameters

The examples lay out a common enterprise-scale pattern in both ADO and GH. This type of pattern is typically employed at organizations where a central team is tasked with ensuring that all automation adheres to corporate Standards and, often, external regulations. This is common in highly regulated industries like Finance and Energy, or any enterprise with rigid governance and compliance requirements for automation. 

Note the following presumptions about the core principles to which these examples strive to adhere: 

- **Do Not Repeat Yourself** - All templates and workflows should be reusable and avoid duplication across environments and repositories.

- **Inline Code Should be Avoided Entirely** - All code should be encapsulated to ensure maintainability and consistency across environments. 

- **Single Source of Truth** - Configuration and deployment logic should be centralized to ensure consistency and ease of maintenance.

- **Environment Parity** - Non-production and production environments should follow the same deployment patterns to reduce risk and increase reliability.

- **Explicit Dependencies** - Stages and jobs should declare dependencies to control execution order and ensure correct sequencing.

- **Parameterization** - Inputs should be parameterized to allow flexibility and reuse across different environments and applications.

A comment on the second bullet point. This principle implies a heavy prejudice for centralized Tasks/Actions. Regardless of an organization's skill level, scripting in YAMLs leads to more difficult maintenance, lengthier troubleshooting, and more fragile automation. This can make it difficult to manage changes and ensure consistent behavior across environments. It also requires a different skill set than employed by most DevOps Engineers. So while these examples might be easy for *you* to understand in isolation, try to think about them in the context of a major, Enterprise-scale SDLC - including all the humans involved in keeping such a practice operational.  

## The GitHub Actions Pattern has Some Challenges
The following sections highlight specific issues and limitations that arise when using GitHub Actions for this exact deployment pattern. We'll map these challenges to the principles above and discuss how they manifest with Actions.

### GH Actions Often Require JSON
To evaluate this first point fairly, we should compare equivalent layers in the stack: the ADO stage template (`azure-appservice-deploy-stages.yml`) and the GH Reusable Workflow (`deploy-appservices.yml`). At this orchestration layer, ADO accepts a structured object and traverses it directly with template expressions. GitHub reusable workflows, by contrast, only accept `string`/`boolean`/`number` inputs, so complex lifecycle data must be serialized as JSON and parsed with `fromJSON(...)`.

**ADO stage template (`azure-appservice-deploy-stages.yml`) — object input + direct traversal (no JSON parsing):**

```yaml
parameters:
  - name: environments
    type: object

stages:
  - ${{ each env in parameters.environments }}:
      - stage: ${{ env.stageName }}
        dependsOn: ${{ env.dependsOn }}
        jobs:
          - ${{ each appService in env.appServices }}:
              - deployment: ${{ appService.deploymentName }}
                environment: ${{ env.environmentResource }}
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

Again, this is a very simple example, and single-target scenarios can often rely on a default App Service Action in GitHub. In this example pattern, however, we still need iteration across multiple app services and environment-specific branching logic. Without centralizing that behavior in a reusable component (like a Composite Action), teams typically end up reintroducing inline scripting and duplicated YAML - more on this below.

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
### GitHub Actions Lack the Object Parameter Model

The previous section showed this at the reusable-orchestrator layer. Here we focus specifically on the consumer (or caller) layer. Azure DevOps callers can pass a structured YAML object directly into the stage template, while GitHub callers in this pattern pass serialized JSON into the Reusable Workflow. That extra serialization/parsing step increases verbosity and maintenance burden as lifecycle complexity grows.

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
      artifact_name: drop
      artifact_path: artifact
      package_path: app.zip
      vm_image: ubuntu-latest
      environments_json: |
        {
          "dev": {
            "display_name": "Development",
            "enabled": true,
            "is_prod": false,
            "resource_group": "rg-someapp-dev",
            "environment_resource": "dev",
            "app_services": [
              { "name": "someapp-web-dev-01" },
              { "name": "someapp-web-dev-02" }
            ]
          },
          "prod": {
            "display_name": "Production",
            "enabled": true,
            "is_prod": true,
            "resource_group": "rg-someapp-prod",
            "environment_resource": "prod",
            "depends_on": ["deploy_dev"],
            "app_services": [
              { "name": "someapp-web-prod-01" },
              { "name": "someapp-web-prod-02" }
            ]
          }
        }
```

### GitHub Actions Lack ADO-Style Template-Time Loop Patterns

Azure DevOps supports template-time loop patterns built with iteration (`each`) and conditional blocks (`if`) directly in YAML templates. That lets one template scale across many environments and app services without duplicating job definitions.

GitHub Actions does not have an equivalent template-time loop model inside workflow YAML. The practical impacts are:

- **More duplication**: environment jobs are often repeated with only small differences.
- **Harder maintenance**: changes to shared logic must be made in multiple places unless moved into a Composite Action.
- **Less expressive hierarchy**: nested loop patterns (environment → app service) are less natural and often require JSON parsing, matrices, or inline script.
- **Dependency complexity**: per-environment ordering and per-app fan-out/fan-in are more manual than ADO template expansion.

**Azure DevOps template-time loop patterns (`azure-appservice-deploy-stages.yml`) — native nested loops:**

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

Also notice that the GH matrix approach is often less suitable where strict promotion order and environment-specific controls are required. In many implementations, teams end up with multiple, duplicative matrices, where each matrix row becomes a sibling job instance. In practice, you generally can’t rely on a single matrix to define per-row dependencies such as:

- `test-*` depends on all `dev-*`
- `prod-*` depends on all `test-*`

- **Matrices are a good fit** when environments are mostly identical and can run in parallel. Examples might include IAC to sibling environments. 
- **But they're a bad fit** when you need strict promotion order (dev → test → prod), environment-specific approvals/gates, or deeply nested lifecycle logic. In those cases, matrix often shifts complexity into conditions and JSON lookups rather than truly reducing it. 

Consider the complexity of this basic example that attempts to implement environment promotion order using a matrix; this would be a single `dependsOn` property in the object we pass to our ADO Stages model. In GH, it would be more readable with independent jobs and explicit `needs` relationships: 

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

### GitHub Actions Lack the Stages Construct

In Azure DevOps, Stages provide a natural way to sequence deployments across environments (e.g., dev → prod) and manage dependencies. We define a Stage once, use our Object type parameter to control which environments are included, and then iterate over that as many times as needed. And since the Object parameter type is structured, we can easily access environment-specific properties (like `dependsOn`, `isProd`, etc.) within the stage definition.

In GitHub, sequencing is achieved using jobs with `needs` relationships. Since there is no first-class `stages` construct in workflow YAML, each environment deployment is typically modeled as a separate job. The snippets below intentionally isolate that stage-vs-job sequencing difference; matrix-specific tradeoffs were covered in the prior section.

**Azure DevOps (`azure-appservice-deploy-stages.yml`) — stage-level lifecycle:**

```yaml
stages:
  - ${{ each env in parameters.environments }}:
      - stage: ${{ env.stageName }}
        dependsOn: ${{ env.dependsOn }}
        jobs:
          - ${{ each appService in env.appServices }}:
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

Remember that this is a very simple expression of the challenge. Even so, we are required to define two separate jobs (`deploy_dev` and `deploy_prod`) with their own inline steps, even though the work is largely similar - and this compounds with every new environment we add. While our example pattern lessens the concern by referencing a Composite Action, this is where a lot of organizations create cascading complexity: it is common to see Actions defined repeatedly under each job (rather than taking the extra step of encapsulating the work in a Composite Action). 

Here's how that looks in practice. Notice how the inline code is now also repeated - and this is just two environments. The problems this creates are substantial and meaningful at scale. 

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



## Reference

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