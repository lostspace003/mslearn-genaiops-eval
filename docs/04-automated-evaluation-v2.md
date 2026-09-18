---
lab:
    title: 'Automated evaluation with cloud evaluators (v2)'
    description: 'Scale quality testing with automated cloud evaluators for systematic evaluation of AI agents'
    level: 300
    duration: 40 minutes
---

# Automated evaluation with cloud evaluators (v2)

This exercise takes approximately **40 minutes**.

> **Note**: This is a corrected revision of [Lab 04](04-automated-evaluation.md), verified by running it end to end on a local Windows 11 machine (Python 3.13, Azure CLI 2.84, azd 1.34) against a fresh `azd up` deployment in Sweden Central. Every step below was executed. Steps that differ from the original are marked **[v2]** with an explanation of what fails without the change.

## Introduction

In this exercise, you'll use Microsoft Foundry's cloud evaluators to automatically assess quality at scale for the Adventure Works Trail Guide Agent. You'll run evaluations against a test dataset to validate quality metrics and establish an automated evaluation pipeline for future changes.

**Scenario**: You're operating the Adventure Works Trail Guide Agent. You want to evaluate it against a test dataset (89 query-response pairs) to validate quality metrics and establish an automated evaluation pipeline that can scale as your agent evolves.

You'll use the following evaluation criteria—automated at scale:

- **Intent Resolution**: Does the response fully address what the user asked?
- **Relevance**: Is the response appropriate and on-topic for the query?
- **Groundedness**: Are the claims factually accurate and based on domain knowledge?

## Set up the environment

To complete the tasks in this exercise, you need:

- Visual Studio Code
- Azure subscription with Microsoft Foundry access
- Git and [GitHub](https://github.com) account
- Python 3.9 or later
- Azure CLI and Azure Developer CLI (azd) installed

> **Tip**: If you haven't installed these prerequisites yet, see [Lab 00: Prerequisites](00-prerequisites.md) for installation instructions and links.

All steps in this lab will be performed using Visual Studio Code and its integrated terminal.

### Create repository from template

You'll start by creating your own repository from the template to practice realistic workflows.

1. In a web browser, navigate to the template repository on [GitHub](https://github.com) at `https://github.com/MicrosoftLearning/mslearn-genaiops`.
1. Select **Use this template** > **Create a new repository**.
1. Enter a name for your repository (e.g., `mslearn-genaiops`).
1. Set the repository to **Public** or **Private** based on your preference.
1. Select **Create repository**.

### Clone the repository in Visual Studio Code

After creating your repository, clone it to your local machine.

1. In Visual Studio Code, open the Command Palette by pressing **Ctrl+Shift+P**.
1. Type **Git: Clone** and select it.
1. Enter your repository URL: `https://github.com/[your-username]/mslearn-genaiops.git`
1. Select a location on your local machine to clone the repository.
1. When prompted, select **Open** to open the cloned repository in VS Code.

> **[v2] Do not clone with `--depth 1`.** If you clone shallow from the command line, the later `git push` to your own repository is rejected with `remote unpack failed: index-pack failed`. Run `git fetch --unshallow` to recover.

### Install the required azd extension **[v2]**

`azure.yaml` in this template declares a required azd extension:

```yaml
requiredVersions:
  extensions:
    "azure.ai.agents": ">=0.1.0-preview"
```

`azd up` fails on a machine that does not have it. Check and install before provisioning:

```powershell
azd extension list --installed
azd extension install azure.ai.agents
```

### Check model availability in your region **[v2]**

The template deploys `gpt-5.1` on the `GlobalStandard` SKU. That combination does not exist in every region — in **Sweden Central**, `gpt-5.1` is offered only as `Standard`, so `azd up` fails at the model deployment step.

Verify what is actually available **before** provisioning:

```powershell
az login
az cognitiveservices model list -l swedencentral --query "[?contains(model.name,'gpt-5')].{name:model.name, ver:model.version, sku:model.skus[0].name}" -o table
```

Then confirm you have quota for the model you intend to use:

```powershell
az cognitiveservices usage list -l swedencentral --query "[?contains(name.value,'GlobalStandard') && contains(name.value,'5-mini')].{name:name.value,used:currentValue,limit:limit}" -o table
```

If your chosen model is not available as `GlobalStandard`, edit the `aiProjectDeploymentsJson` block in `infra/main.bicep` before continuing. This lab was verified with:

```json
[
  {
    "name": "gpt-5-mini",
    "model": {
      "format": "OpenAI",
      "name": "gpt-5-mini"
    },
    "sku": {
      "name": "GlobalStandard",
      "capacity": 10
    }
  }
]
```

Whichever model you deploy becomes your judge model — use that same name for `MODEL_NAME` later.

### Deploy Microsoft Foundry resources

Now you'll use the Azure Developer CLI to deploy all required Azure resources.

1. In Visual Studio Code, open a terminal by selecting **Terminal** > **New Terminal** from the menu.

1. Authenticate with Azure Developer CLI:

    ```powershell
    azd auth login
    ```

    This opens a browser window for Azure authentication. Sign in with your Azure credentials.

1. Authenticate with Azure CLI:

    ```powershell
    az login
    ```

    Sign in with your Azure credentials when prompted.

    > ⚠️ **Important**
    > In some environments, the VS Code integrated terminal may crash or close during the interactive login flow.
    > If this happens, authenticate using explicit credentials instead:
    > ```powershell
    > az login --username <your-username> --password <your-password>
    > ```

1. Provision resources:

    ```powershell
    azd up
    ```

    When prompted, provide:
    - **Environment name** (e.g., `dev`, `test`) - Used to name all resources
    - **Azure subscription** - Where resources will be created
    - **Location** - Azure region (recommended: Sweden Central)

    > **[v2] Non-interactive alternative.** To avoid the prompts (useful when re-running), create the environment first and then provision:
    > ```powershell
    > azd env new <env-name> --subscription <subscription-id> --location swedencentral
    > azd up --no-prompt
    > ```

    The command deploys the infrastructure from the `infra\` folder, creating:
    - **Resource Group** - Container for all resources
    - **Foundry (AI Services)** - The hub with your model deployment
    - **Foundry Project** - Your workspace for creating and managing prompts
    - **Log Analytics Workspace** - Collects logs and telemetry data
    - **Application Insights** - Monitors performance and usage
    - **[v2] Container Registry** - also created, because `enableHostedAgents` defaults to `true`. This lab does not need hosted agents; set `ENABLE_HOSTED_AGENTS=false` in your azd environment if you want to skip it and the ~4 minute capability-host step.

    Provisioning takes about 6 minutes; the Foundry capability host alone accounts for roughly 4 of those.

1. Create a `.env` file with the environment variables:

    ```powershell
    azd env get-values | Out-File .env -Encoding utf8
    ```

    > **[v2]** The original lab uses `azd env get-values > .env`, which in **Windows PowerShell 5.1** writes UTF-16 LE and produces variables that are read incorrectly. `Out-File -Encoding utf8` avoids the problem outright. If you used the redirect, check the encoding indicator in the bottom-right of VS Code and re-save as **UTF-8** if it shows anything else.

    This creates a `.env` file in your project root with all the provisioned resource information. Confirm it contains `AZURE_AI_PROJECT_ENDPOINT`, and that the value ends in `/api/projects/<project-name>`.

    The repository's `.gitignore` already ignores `*.env`, so `.env` will not be committed.

### Install Python dependencies

With your Azure resources deployed, install the required Python packages.

1. In the VS Code terminal, create and activate a virtual environment:

    ```powershell
    python -m venv .venv
    .venv\Scripts\Activate.ps1
    ```

1. **[v2]** Add the virtual environment to `.gitignore`. The shipped `.gitignore` covers `.env` and `.azure/` but **not** `.venv/`, so a later `git add .` would commit the entire virtual environment:

    ```powershell
    Add-Content .gitignore "`n# Local Python environment`n.venv/`n__pycache__/"
    ```

1. Install the required dependencies:

    ```powershell
    python -m pip install -r requirements.txt
    ```

    This installs necessary dependencies including:
    - `azure-ai-projects` - SDK for working with Microsoft Foundry
    - `azure-identity` - Azure authentication
    - `python-dotenv` - Load environment variables

1. Add the agent configuration to your `.env` file:

    Open the `.env` file in your repository root and add:

    ```
    AGENT_NAME="trail-guide"
    MODEL_NAME="<model_name>"
    ```

    > **Note**: Set `MODEL_NAME` to the model deployment you created earlier. This lab was verified with `gpt-5-mini`.

1. **[v2]** Set the console encoding for this session:

    ```powershell
    $env:PYTHONIOENCODING = "utf-8"
    ```

    `evaluate_agent.py` prints a `✓` character. On Windows, Python encodes output using the system code page (cp1252), and the script crashes partway through with:

    ```
    Error: 'charmap' codec can't encode character '✓' in position 2: character maps to <undefined>
    ```

    The failure happens *after* the dataset upload, so it looks like an upload problem when it is purely an encoding problem.

## Understand the evaluation workflow

Cloud evaluation follows a structured workflow:

```text
1. Prepare Dataset
          ↓
2. Define Evaluation Criteria (Evaluators)
          ↓
3. Create Evaluation Definition
          ↓
4. Run Evaluation against Dataset
          ↓
5. Poll for Completion
          ↓
6. Retrieve & Interpret Results
```

### Dataset preparation

The repository includes `data/trail_guide_evaluation_dataset.jsonl` with 89 pre-generated query-response pairs covering diverse hiking scenarios. Each entry includes:

- `query`: User question
- `response`: Agent-generated answer
- `ground_truth`: Reference answer for accuracy comparison

### Evaluators

You'll use Microsoft Foundry's built-in quality evaluators:

| Evaluator | Measures | Output | Use Case |
|-----------|----------|--------|----------|
| **Intent Resolution** | Query intent addressed | 1-5 score | Ensure user needs are met |
| **Relevance** | Response addresses query | 1-5 score | Validate query-response alignment |
| **Groundedness** | Factual accuracy | 1-5 score | Ensure reliable information |

All evaluators use the configured judge model as an LLM judge and return:

- **Score**: 1-5 scale (5 = excellent)
- **Label**: Pass/Fail based on threshold (default: 3)
- **Reason**: Explanation of the score
- **Threshold**: Configurable pass/fail cutoff

## Run cloud evaluation

### Verify the evaluation dataset

First, examine the prepared dataset structure.

1. View the first few entries in the dataset:

    ```powershell
    Get-Content data/trail_guide_evaluation_dataset.jsonl -Head 3
    ```

1. Count total entries in the dataset:

    ```powershell
    (Get-Content data/trail_guide_evaluation_dataset.jsonl).Count
    ```

    Expected: 89 entries

### Reduce the dataset for a first run **[v2]**

A full 89-item run takes 15-60+ minutes. The original lab suggests using "a smaller temporary dataset" for a smoke test but gives no command. Trim the dataset to the first 5 rows so you can validate authentication, upload, evaluator configuration and scoring in about two minutes:

```powershell
$rows = Get-Content data/trail_guide_evaluation_dataset.jsonl -Head 5
Set-Content data/trail_guide_evaluation_dataset.jsonl -Value $rows -Encoding utf8
```

Restore the full dataset at any time with:

```powershell
git restore data/trail_guide_evaluation_dataset.jsonl
```

Do the full 89-item run once the pipeline is proven. A 5-item run validates the plumbing, not the agent's quality.

### Fix the score-extraction bug **[v2]**

`src/evaluators/evaluate_agent.py` reads per-item scores from `item.evaluator_outputs`, which the Evals API does not return. The evaluation completes successfully, reports `Errored items: 0`, and then prints:

```
Average Scores (1-5 scale, threshold: 3)
  No scores returned — open Azure AI Foundry portal > Evaluations for details.
```

The scores are actually in `item.results`, where each entry has a `metric` (and `name`) plus a numeric `score`. In `retrieve_and_display_results`, replace this block:

```python
    for item in scored_items:
        if hasattr(item, "evaluator_outputs"):
            for output in item.evaluator_outputs:
                if output.name in scores and hasattr(output, "score"):
                    scores[output.name].append(output.score)
```

with:

```python
    for item in scored_items:
        # The Evals API returns one entry per evaluator in item.results, each
        # carrying 'metric' (or 'name') and a numeric 'score'. Entries may come
        # back as dicts or as model objects depending on SDK version.
        for result in (getattr(item, "results", None) or []):
            if isinstance(result, dict):
                metric = result.get("metric") or result.get("name")
                value = result.get("score")
            else:
                metric = getattr(result, "metric", None) or getattr(result, "name", None)
                value = getattr(result, "score", None)
            if metric in scores and value is not None:
                scores[metric].append(float(value))
```

Without this change the lab appears to succeed while producing no results, locally and in CI alike.

### Run the evaluation

Execute the complete evaluation pipeline with one command.

1. **Run the evaluation**

    ```powershell
    python src/evaluators/evaluate_agent.py
    ```

    Expected output for a 5-item run:

    ```
    ================================================================================
     Trail Guide Agent - Cloud Evaluation
    ================================================================================

    Configuration:
      Project: https://<account>.services.ai.azure.com/api/projects/<project>
      Model:   gpt-5-mini
      Dataset: trail-guide-evaluation-dataset (v1)

    ================================================================================
    Step 1: Uploading evaluation dataset
    ================================================================================

    ✓ Dataset uploaded successfully

    ================================================================================
    Step 2: Creating evaluation definition
    ================================================================================

    ✓ Evaluation definition created
      Evaluation ID: eval_846c0a0668a9467c888b8bf688b2261b

    ================================================================================
    Step 3: Running cloud evaluation
    ================================================================================

    ✓ Evaluation run started
      Run ID: evalrun_4b3b2dba4bba47d8abf3ddb3ec9540e2
      Status: queued

    ================================================================================
    Step 4: Polling for completion
    ================================================================================
      [96s] Status: in_progress

    ✓ Evaluation completed in 130 seconds

    ================================================================================
    Step 5: Retrieving results
    ================================================================================

      Total items  : 5
      Errored items: 0
      Scored items : 5

    Average Scores (1-5 scale, threshold: 3)
      Intent Resolution: 5.00 (n=5)
      Relevance        : 4.80 (n=5)
      Groundedness     : 5.00 (n=5)

    Pass Rates (score >= 3)
      Intent Resolution: 100.0%
      Relevance        : 100.0%
      Groundedness     : 100.0%
    ```

    > **[v2] If the very first run fails with a connection error.** Immediately after `azd up`, the first dataset upload can fail with `('Connection aborted.', ConnectionResetError(10054, 'An existing connection was forcibly closed by the remote host'))`. This is role propagation, not a configuration error. Wait a minute and run the script again — the second attempt succeeds, and the script reuses the already-uploaded dataset version.

    > **Note**: The script uploads the dataset as `trail-guide-evaluation-dataset` version `1`. Foundry refuses to upload the same name and version twice, so on later runs the script reuses the existing version. If you change the dataset contents and want the new rows evaluated, bump `dataset_version` in the script.

1. **Commit the results file**

    The script writes a summary to `evaluation_results.txt` in your project root.

    If Git reports `Author identity unknown`, configure your identity once before committing:

    ```powershell
    git config --global user.name "Your GitHub Username"
    git config --global user.email "your-email@example.com"
    ```

    ```powershell
    git add evaluation_results.txt
    git commit -m "Add evaluation results"
    git push
    ```

### Review results in the Foundry portal

1. In the [Microsoft Foundry portal](https://ai.azure.com), open your project and select **Evaluations**.

1. Open the run whose ID the script printed and view:
   - **Aggregate metrics**: Overall pass rates and score distributions
   - **Individual test results**: Score, label (pass/fail), and reasoning for each query-response pair
   - **Evaluator details**: How each evaluator scored each response

1. Identify patterns:
   - Which types of queries score lowest?
   - Are there consistent reasoning themes in failures?
   - Do certain evaluators flag more issues than others?

## Automate with GitHub Actions

The evaluation script integrates with GitHub Actions to automatically run evaluations on pull requests that modify agent code, and post results as a PR comment.

1. **Uncomment the PR trigger in the workflow**

    Open `.github/workflows/evaluate-agent.yml` and uncomment the `pull_request` trigger.

    Change this:

    ```yaml
    on:
      # pull_request:
      #   branches: [main]
      #   paths:
      #     - 'src/agents/trail_guide_agent/**'
      workflow_dispatch:
    ```

    To this:

    ```yaml
    on:
      pull_request:
        branches: [main]
        paths:
          - 'src/agents/trail_guide_agent/**'
      workflow_dispatch:
    ```

    Save the file and commit this workflow change before creating the test pull request. If you skip this step, the workflow will not start automatically for PRs.

1. **Configure Azure authentication**

    > **Important - Tenant alignment**
    >
    > Before creating the service principal, make sure your Azure CLI session is using the same tenant as the subscription that holds your Foundry resources:
    > ```powershell
    > az account show --query "{subscription:id, tenant:tenantId}" -o table
    > ```
    > If needed, switch to the correct tenant and subscription before continuing. Creating the app in the wrong tenant is a common cause of OIDC sign-in failures later.

    Create a service principal for GitHub Actions:

    ```powershell
    az ad sp create-for-rbac --name "github-agent-evaluator"
    ```

    Save the `appId` and `tenant` values from the output. The workflow uses OIDC federated credentials, so the generated `password` is not used in this lab.

    > **[v2]** With Azure CLI 2.84 this command creates the app and service principal **without any role assignment**. The role assignment below is therefore mandatory, not a top-up.

    Assign the **Foundry User** role so the service principal can call the Foundry project API:

    ```powershell
    az role assignment create `
      --assignee "<appId>" `
      --role "Foundry User" `
      --scope "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.CognitiveServices/accounts/<ai-account-name>"
    ```

    Use the `AZURE_SUBSCRIPTION_ID`, `AZURE_RESOURCE_GROUP`, and `AZURE_AI_ACCOUNT_NAME` values from your `.env` file to fill in the scope.

    > **[v2] If `--assignee` cannot be resolved.** On accounts where the Graph lookup fails (common with Microsoft accounts signed into a work tenant), the command returns `MissingSubscription`. Pass the object ID instead:
    > ```powershell
    > $objectId = az ad sp show --id "<appId>" --query id -o tsv
    > az role assignment create `
    >   --assignee-object-id $objectId `
    >   --assignee-principal-type ServicePrincipal `
    >   --role "Foundry User" `
    >   --scope "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.CognitiveServices/accounts/<ai-account-name>"
    > ```
    > Run these commands in **PowerShell**, not Git Bash. Git Bash rewrites the leading `/subscriptions/...` scope into a Windows path, which also surfaces as `MissingSubscription`.

    Verify the assignment landed:

    ```powershell
    az role assignment list --all --assignee "<appId>" --query "[].{role:roleDefinitionName,scope:scope}" -o table
    ```

1. **Create federated credentials** **[v2]**

    GitHub sends a different token subject for each trigger type, so you need one credential per subject. **Current GitHub repositories issue an immutable subject that embeds numeric owner and repository IDs**, for example:

    ```
    repo:myuser@231288280/myrepo@1375878516:ref:refs/heads/main
    ```

    The classic `repo:<org>/<repo>:ref:refs/heads/main` form no longer matches, and the workflow fails with `AADSTS700213`. The reliable procedure is to read the subject GitHub actually presents and mirror it exactly.

    First create credentials using the classic subjects:

    Create `federated-credential.json`:

    ```json
    {
      "name": "github-actions",
      "issuer": "https://token.actions.githubusercontent.com",
      "subject": "repo:<your-org>/<your-repo>:ref:refs/heads/main",
      "audiences": ["api://AzureADTokenExchange"]
    }
    ```

    ```powershell
    az ad app federated-credential create `
      --id "<appId>" `
      --parameters @federated-credential.json
    Remove-Item federated-credential.json
    ```

    Create `federated-credential-pr.json`:

    ```json
    {
      "name": "github-actions-pr",
      "issuer": "https://token.actions.githubusercontent.com",
      "subject": "repo:<your-org>/<your-repo>:pull_request",
      "audiences": ["api://AzureADTokenExchange"]
    }
    ```

    ```powershell
    az ad app federated-credential create `
      --id "<appId>" `
      --parameters @federated-credential-pr.json
    Remove-Item federated-credential-pr.json
    ```

    Then, after your first workflow run (next steps), if Azure Login fails with `AADSTS700213`, read the exact subject from the run log:

    ```powershell
    gh run view <run-id> --repo <your-org>/<your-repo> --log | Select-String "subject claim"
    ```

    Create one more credential for that exact string, and a matching `:pull_request` variant:

    ```powershell
    @"
    {
      "name": "github-actions-immutable",
      "issuer": "https://token.actions.githubusercontent.com",
      "subject": "<subject exactly as printed in the run log>",
      "audiences": ["api://AzureADTokenExchange"]
    }
    "@ | Set-Content fc-main.json -Encoding utf8

    az ad app federated-credential create --id "<appId>" --parameters "@fc-main.json"
    Remove-Item fc-main.json
    ```

    Repeat with the same `repo:<owner>@<id>/<repo>@<id>` prefix and `:pull_request` as the suffix. An app can hold several federated credentials, so keeping both the classic and immutable forms is fine and makes the setup portable.

    > **Important**: Subjects are case-sensitive and must match character for character.

1. **Configure GitHub Secrets**

    Add the following secrets to your repository under **Settings → Secrets and variables → Actions → New repository secret**:

    | Secret Name                    | Where to find it                                        |
    |--------------------------------|---------------------------------------------------------|
    | `AZURE_CLIENT_ID`              | `appId` from `az ad sp create-for-rbac` output          |
    | `AZURE_TENANT_ID`              | `tenant` from `az ad sp create-for-rbac` output         |
    | `AZURE_SUBSCRIPTION_ID`        | `AZURE_SUBSCRIPTION_ID` in your `.env` file             |
    | `AZURE_AI_PROJECT_ENDPOINT`    | `AZURE_AI_PROJECT_ENDPOINT` in your `.env` file         |

    > **[v2] Add the `MODEL_NAME` repository variable — it is not optional if you deployed anything other than `gpt-5.1`.** The workflow falls back to `gpt-5.1`, which will not exist in your project.
    >
    > **Settings → Secrets and variables → Actions → Variables → New repository variable**
    > Name: `MODEL_NAME`, Value: the deployment you created (this lab used `gpt-5-mini`)

    The same thing from the CLI:

    ```powershell
    gh secret set AZURE_CLIENT_ID --body "<appId>"
    gh secret set AZURE_TENANT_ID --body "<tenant>"
    gh secret set AZURE_SUBSCRIPTION_ID --body "<subscription-id>"
    gh secret set AZURE_AI_PROJECT_ENDPOINT --body "<project-endpoint>"
    gh variable set MODEL_NAME --body "gpt-5-mini"
    ```

1. **Test the workflow manually**

    Before testing with a PR, verify the workflow runs successfully with a manual trigger:

    1. Go to your repository on GitHub
    1. Navigate to **Actions → Evaluate Trail Guide Agent**
    1. Click **Run workflow**, select `main`, and click **Run workflow**
    1. Wait for the run to complete and verify it passes

    With the 5-row dataset this run takes about 3.5 minutes end to end.

1. **Test with a pull request**

    The workflow runs automatically when a PR modifies files under `src/agents/trail_guide_agent/`. To test it, create a branch, make a small change, and open a PR:

    ```powershell
    git checkout -b test/trigger-evaluation
    # Make a small change to any agent file, for example:
    code src/agents/trail_guide_agent/prompts/v1_instructions.txt
    git add .
    git commit -m "test: trigger evaluation workflow"
    git push origin test/trigger-evaluation
    ```

    Then open a pull request from `test/trigger-evaluation` → `main` on GitHub. The workflow will start automatically.

1. **View results in the PR**

    Once the workflow completes, a comment is posted to your PR with:
    - Evaluation scores and pass rates for each criterion
    - Full log output in a collapsible section
    - Link to detailed results in the Microsoft Foundry portal

## Clean up resources **[v2]**

The original lab has no cleanup step. The deployed resources bill continuously — the Foundry account, Application Insights, Log Analytics and the Container Registry all keep costing money after the lab ends.

When you're finished, delete everything the template created:

```powershell
azd down --purge --force
```

`--purge` is important for the Foundry (AI Services) account: without it the account is soft-deleted and its name stays reserved.

If you created the service principal, remove it too:

```powershell
az ad sp delete --id "<appId>"
```

## Troubleshooting

### `azd up` fails at the model deployment

**Symptom**: Provisioning fails while creating the model deployment.

**Resolution**:
- The requested model and SKU combination is not available in your region. Run the `az cognitiveservices model list` command from the setup section and pick a model listed as `GlobalStandard`
- Check quota with `az cognitiveservices usage list` before retrying
- Update `aiProjectDeploymentsJson` in `infra/main.bicep` and rerun `azd up`

### Script crashes with a `charmap` codec error

**Symptom**: `'charmap' codec can't encode character '✓'`.

**Resolution**: Set `$env:PYTHONIOENCODING = "utf-8"` before running the script. The crash is an encoding failure in the progress output, not an Azure error.

### Evaluation completes but reports "No scores returned"

**Symptom**: `Errored items: 0` and `Scored items: 5`, but every metric prints "No scores returned".

**Resolution**: Apply the score-extraction fix in the **Fix the score-extraction bug** section above. The scores are present in the run; the script is reading the wrong attribute.

### Connection reset on the first run after `azd up`

**Symptom**: `ConnectionResetError(10054, 'An existing connection was forcibly closed by the remote host')` during dataset upload.

**Resolution**: Wait 1-2 minutes for the role assignment to propagate and run the script again.

### Evaluation taking longer than expected

**Symptom**: Evaluation runs for 20+ minutes or appears stuck.

**Resolution**:
- Check Azure OpenAI quota and rate limits in Azure portal
- Verify the model deployment has enough capacity in the selected region; low-capacity deployments can stay in `running` for a long time without surfacing an immediate error
- Reduce dataset size for initial testing
- Check the Microsoft Foundry portal to confirm the evaluation run is still active before cancelling the script locally

### Authentication errors

**Symptom**: `401 Unauthorized` or `403 Forbidden` errors.

**Resolution**:
- Run `az login` to refresh Azure credentials
- Verify the service principal has the **Foundry User** role at the CognitiveServices account scope. `Foundry Developer` alone is **not sufficient**
- Check `AZURE_AI_PROJECT_ENDPOINT` in `.env` is correct and includes `/api/projects/<project>`
- If the first run happens immediately after `azd up`, wait 1-2 minutes and retry once

### `MissingSubscription` from `az role assignment create`

**Symptom**: `(MissingSubscription) The request did not have a subscription or a valid tenant level resource provider.`

**Resolution**:
- Run the command in **PowerShell**, not Git Bash. Git Bash rewrites the leading `/subscriptions/...` into a Windows path
- If the Graph lookup for `--assignee` fails, use `--assignee-object-id` with `--assignee-principal-type ServicePrincipal`

### OIDC login fails (`AADSTS700213`)

**Symptom**: `No matching federated identity record found for presented assertion subject 'repo:...'`.

**Resolution**:

Read the subject GitHub actually sent, from the failed run's log:

```powershell
gh run view <run-id> --repo <your-org>/<your-repo> --log | Select-String "subject claim"
```

Two things commonly differ from the lab's literal strings:
- **Trigger type**: `workflow_dispatch`/`push` on main uses `...:ref:refs/heads/main`, while `pull_request` uses `...:pull_request`. You need a credential for each
- **Immutable subjects**: current repositories embed numeric IDs, e.g. `repo:<owner>@231288280/<repo>@1375878516:ref:refs/heads/main`

Create a federated credential whose `subject` matches the logged string exactly.

### Push to your repository is rejected

**Symptom**: `remote unpack failed: index-pack failed`.

**Resolution**: The clone is shallow. Run `git fetch --unshallow` and push again.

### Rate limit errors during evaluation

**Symptom**: Evaluation fails with `429 Too Many Requests` errors.

**Resolution**:
- Check the deployment's tokens-per-minute (TPM) quota
- Increase quota in Azure portal if needed
- Split large datasets into smaller batches

## Next steps

- Continue to [Lab 05: Monitoring and tracing](05-monitoring-tracing.md) to track production agent performance with Application Insights
- Explore [Lab 06: Optimize with fine-tuning](06-optimize-finetuning.md)

> **[v2]** The original lab links to `05-monitoring.md` and `06-tracing.md`, neither of which exists in the repository. The filenames above are the actual ones.
