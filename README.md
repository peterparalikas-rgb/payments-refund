# payments-refund
Comprehensive API for managing payment refunds in a financial services platform. This API handles refund creation, status tracking, cancellation, and reporting for payment transactions processed through the authorization and capture workflow. Said workflow triggers followup run to sync the loan-origination-api-openapi.yaml file upon successful sync of the payment-refund-api-openapi.yml

# Workflow Creation Process
Creating the workflow, I reviewed the documentation provided in the [Postman API Onboarding Action](https://github.com/postman-cs/postman-api-onboarding-action). I took the format and relevant information, and added the following:
- Added the boilerplate 'name' and 'on' sections to the workflow
- Ensured there were push/pull/manual options for the workflow to trigger
- Experimented with additional permissions, before removing them (pull-requests, deployments)
- Updating the spec-url to point toward a gist github endpoint, due to issues with running it from within my repo
- Added the following variables under the 'with' statement for the payments-redund workflow:
```
          workspace-id: ${{ vars.POSTMAN_WORKSPACE_ID }}
          spec-id: ${{ vars.POSTMAN_SPEC_UID }}
          baseline-collection-id: ${{ vars.POSTMAN_BASELINE_COLLECTION_UID }}
          smoke-collection-id: ${{ vars.POSTMAN_SMOKE_COLLECTION_UID }}
          contract-collection-id: ${{ vars.POSTMAN_CONTRACT_COLLECTION_UID }}
```
- These variables were put in place to ensure that the same workspace, spec and collections would be referenced upon running the Actions in the future, ensuring the same workspace and data would be updated upon the workflow running, as opposed to an entirely new workspace/spec/collection being created.

- I also placed the workspace-id variable in the workflow for my second API, the loan-origination-api. I made sure that the variable pointed to the correct workspace ID that had been created by the payment-refund workflow. I left out the other variables to test if they were required or not, and they appear to not be required; the workflow properly updated the repo variables accordingly. 

# Run Instructions 

## Automatic
1. Push an update to the branch noted in the workflow (by default, 'main').

## Manual
1. Go to **Actions -> FunctioningAPIWorkflow** (or name of your workflow) **-> Run Workflow**

## 1. Pre-Run Validation

### 1.1 Repository and layout

| Check | Expected | How to verify |
|-------|----------|----------------|
| Workflow files present | `FunctioningAPIWorkflow.yml` under `.github/workflows/` | `ls .github/workflows/` |
| Spec output path | `specs/` exists or will be created by the generate workflow | Generate workflow creates `specs/` if missing |

### 1.2 Secrets (GitHub: Settings → Secrets and variables → Actions)

| Secret | Used by | Validation |
|--------|---------|------------|
| `POSTMAN_API_KEY` | Postman onboarding; generate-spec (if source = postman_collection) | Must be a valid Postman API key; test with a simple Postman API call |
| `POSTMAN_ACCESS_TOKEN` | Postman onboarding action | Must be current; `postman login` and update if expired |
| `GH_PAT` or `GH_FALLBACK_TOKEN` | Generate-spec (commit/push); trigger downstream | Must have `contents: write`; for downstream, must be able to trigger `repository_dispatch` on target repo |
| `GITHUB_TOKEN` | Provided by Actions; used by onboarding for repo operations | No setup; ensure job has `contents: write` if it writes to repo |

**Quick check:** Run the generate-spec workflow manually with minimal inputs; if it fails on push, the PAT/fallback token or branch permissions are the usual cause.

### 1.3 Repository variables (optional but recommended after first run)

| Variable | Purpose |
|----------|---------|
| `POSTMAN_WORKSPACE_ID` | Reuse existing workspace on subsequent onboarding runs |
| `POSTMAN_SPEC_UID` | Reuse spec in Spec Hub |
| `POSTMAN_BASELINE_COLLECTION_UID`, `POSTMAN_SMOKE_COLLECTION_UID`, `POSTMAN_CONTRACT_COLLECTION_UID` | Reuse collections instead of creating new ones |

Leave unset on first run; after first successful onboarding, capture from job output and set as repository variables.

### 1.4 Placeholders in workflow files

| File | Placeholder | Action |
|------|-------------|--------|
| `FunctioningAPIWorkflow.yml` | `<owner>`, `<repo>`, `<branch>` in `spec-url` | Replace with real GitHub owner, repo name, and branch (e.g. `main`) |
| `FunctioningAPIWorkflow.yml` | `<owner>` in `repos/<owner>/loan-origination-api/dispatches` | Replace with the owner of the loan-origination-api repo |
| `FunctioningAPIWorkflow.yml` | `env-runtime-urls-json` and `environments-json` | Replace with real environment names and base URLs for your API |

**Validation:** Raw spec URL must be reachable in a browser (or `curl`) when not using a private repo; for private repos, the runner and Postman must have access (token in URL or Postman’s access context).

## 2. Workflow 1: Generate OpenAPI Spec and Commit (Optional)

For scenario where no specification exists

### 2.1 Input validation

| Source | Required inputs | Validation |
|--------|------------------|------------|
| `runtime_url` | `runtime_url` | URL must be reachable from GitHub’s runner (public or runner-accessible network). If auth required, set `runtime_url_auth_header` (e.g. Bearer token from secret). |
| `runtime_url` | Optional: `runtime_url_auth_header` | Use a secret reference in manual run if needed; for push-triggered, config cannot hold secrets—use a workflow secret and pass via input if supported. |
| `postman_collection` | `collection_path` **or** `postman_collection_uid` + `POSTMAN_API_KEY` | Path must exist in repo; UID + API key must be valid for Postman API. |
| Any | `spec_output_name` | Must be a filename only (e.g. `payment-refund-api-openapi.yaml`), no path. Will be written under `specs/`. |

### 2.2 Push-triggered run

| Check | Expected |
|-------|----------|
| Config file | `.github/spec-generation-config.yaml` exists and has correct `source`, `spec_output_name`, and either `runtime_url` or `collection_path`. |
| Trigger paths | Workflow triggers on changes to `src/`, `postman/collection.json`, `package.json`, workflow file, or config. It does **not** trigger on `specs/**` (to avoid loops). |

### 2.3 Post-run validation (Generate Spec)

| Check | How to verify |
|-------|----------------|
| Spec file created | New file under `specs/<spec_output_name>`. |
| Valid OpenAPI | Open the file; valid YAML/JSON with `openapi`, `info`, `paths` (or use a validator). |
| Commit and push | Latest commit on the branch contains the new/changed spec file; push succeeded (no 403 from PAT). |

If the workflow runs but there is “No changes to commit,” the generated spec matched the existing file (no diff).

## 3. Workflow 2: Postman API Onboarding

### 3.1 Pre-run validation

| Check | Expected |
|-------|----------|
| Spec URL | `spec-url` in the workflow is the **raw** URL of the spec (e.g. `https://raw.githubusercontent.com/<owner>/<repo>/main/specs/payment-refund-api-openapi.yaml`). Not a local path. |
| Environments | `environments-json` and `env-runtime-urls-json` match (same keys); URLs are the real base URLs for each environment. |
| Trigger paths | If using push trigger, path filter includes the spec file you commit (e.g. `specs/payment-refund-api-openapi.yaml`) and/or the workflow file. |

### 3.2 Post-run validation (Onboarding)

| Check | How to verify |
|-------|----------------|
| Postman workspace | New (or updated) workspace in Postman; name/domain reflects `project-name` and `domain-code`. |
| Spec in Spec Hub | Spec appears in Spec Hub linked to the workspace. |
| Collections | Baseline, smoke, and contract collections created or updated. |
| Environments | Environments (e.g. prod, stage) created with correct base URLs. |
| Repo exports | Exported collections/environments under `postman/` (or configured artifact dir) if the action is configured to export. |
| IDs for reuse | Job output or action logs contain workspace ID, spec UID, collection UIDs—capture and set as repository variables for next run. |

### 3.3 Downstream: Loan Origination API

| Check | Expected |
|-------|----------|
| Trigger job | `trigger-loan-origination` runs only after `onboarding` succeeds. |
| Token | `GH_FALLBACK_TOKEN` has permission to trigger `repository_dispatch` on the target repo. |
| Target repo | `repos/<owner>/loan-origination-api/dispatches` uses the correct `<owner>`. |
| Target workflow | The loan-origination-api repo has a workflow that listens for `repository_dispatch` with the event type used (e.g. `postman-refund-success`). |

# 4. Full Chain Validation

### 4.1 No spec → Generate → Onboarding → Downstream

1. **Before:** No file in `specs/` for the service; generate-spec config or manual inputs set correctly; PAT/fallback token set.
2. **After Generate:** File in `specs/<name>`; commit on default branch.
3. **After Onboarding:** Postman has workspace, spec, collections, environments; repo has exports if configured.
4. **After Downstream:** Loan-origination workflow run appears in the loan-origination-api Actions tab (if applicable).

### 4.2 Spec already in repo

1. **Before:** Spec exists at `specs/<name>`; raw URL is correct; `spec-url` in postman-onboarding.yml matches.
2. **After Onboarding:** Same as in section 3.2.

### 4.3 Shell script (no Actions UI)

When using `postman-onboarding-gh.sh`:

| Mode | Validation |
|------|------------|
| `trigger` | `GITHUB_REPO`, `SPEC_URL`, `ENVIRONMENTS_JSON`, `ENV_RUNTIME_URLS_JSON`, `PROJECT_NAME` set; `gh` installed and authenticated; workflow exists at `.github/workflows/postman-onboarding.yml`. |
| `local` | `POSTMAN_API_KEY`, `SPEC_URL`, environments, `PROJECT_NAME` set; script can reach Postman API and spec URL. |
| `downstream` | `GH_FALLBACK_TOKEN` (or `gh` auth); target repo and event type correct. |

---

## 5. Environment-Specific Validation

Use **SETUP_ADDITIONAL_REQUIREMENTS.md** for full checklists. Summary:

| Environment | Key validation |
|-------------|----------------|
| **AWS / ECS + ALB** | Runtime URL is the client-visible base URL (ALB or custom domain). Runner (and Postman, if used) can reach it; internal ALB requires self-hosted runner or VPN. |
| **GitLab** | Equivalent secrets/variables; raw spec URL format; PAT with `write_repository` for push. |
| **Kubernetes** | Ingress/base URL reachable from runner and Postman; TLS/CA if internal CA. |
| **mTLS** | Client cert and key available to the step that fetches the spec (and to Postman/Newman if they call the API). |

---

## 6. Common Failure Points and Checks

| Symptom | What to validate |
|---------|------------------|
| Generate spec doesn’t push | PAT or `GH_FALLBACK_TOKEN` has `contents: write`; default branch allows pushes from the token; no branch protection blocking. |
| Onboarding fails on spec | `spec-url` is a URL (raw GitHub or other host), not a local path; URL is reachable (and from Postman’s perspective if applicable). |
| Postman token errors | `POSTMAN_ACCESS_TOKEN` / `POSTMAN_API_KEY` valid and not expired; update secrets after `postman login` if needed. |
| Downstream not triggered | `GH_FALLBACK_TOKEN` set; correct `<owner>` and repo name; target workflow has `repository_dispatch` and matching `event_type`. |
| Onboarding workflow not found | Workflow file is under `.github/workflows/` (e.g. `.github/workflows/postman-onboarding.yml`). |
| Push trigger doesn’t run | Changed paths match the workflow’s `on.push.paths`; for generate-spec, changes are not under `specs/` only. |
| Postman IDs not reused | After first run, set repository variables; workflow uses `vars.POSTMAN_*` so subsequent runs pass these IDs. |

# Universal Considerations

## Inputs

| Input | Default | Notes |
| --- | --- | --- |
| `workspace-id` | | Reuse an existing Postman workspace through the bootstrap step. |
| `spec-id` | | Update an existing Postman spec instead of creating a new one. |
| `baseline-collection-id` | | Reuse an existing baseline collection. |
| `smoke-collection-id` | | Reuse an existing smoke collection. |
| `contract-collection-id` | | Reuse an existing contract collection. |
| `generate-ci-workflow` | `true` | Pass through to repo sync; set `false` for repos that already manage CI. |
| `ci-workflow-path` | `.github/workflows/ci.yml` | Pass through to repo sync to redirect generated workflow output. |
| `project-name` | | Service name used across bootstrap and repo sync. |
| `domain` | | Business domain used for governance assignment. |
| `domain-code` | | Short prefix used in workspace naming. |
| `requester-email` | | Optional workspace invite target. |
| `workspace-admin-user-ids` | | Optional comma-separated workspace admin IDs. |
| `spec-url` | | Required registry-backed OpenAPI document URL. |
| `environments-json` | `["prod"]` | Environment slugs to materialize downstream. |
| `system-env-map-json` | `{}` | Map of environment slug to system environment ID. |
| `governance-mapping-json` | `{}` | Map of domain to governance group name. |
| `env-runtime-urls-json` | `{}` | Map of environment slug to runtime base URL. |
| `postman-api-key` | | Required for bootstrap and repo sync Postman operations. |
| `postman-access-token` | | Enables governance assignment and Bifrost integration work. |
| `github-token` | | Enables repository variable persistence and generated commits. |
| `gh-fallback-token` | | Optional fallback token for workflow and variable APIs. |
| `github-auth-mode` | `github_token_first` | GitHub auth mode for repository APIs. |
| `repo-write-mode` | `commit-and-push` | Repo mutation mode passed to repo sync. |
| `current-ref` | | Optional explicit ref override for detached checkouts. |
| `committer-name` | `Postman FDE` | Commit author name for generated sync commits. |
| `committer-email` | `fde@postman.com` | Commit author email for generated sync commits. |
| `integration-backend` | `bifrost` | Current public open-alpha backend. |

## Obtaining postman-api-key

The postman-api-key is a Postman API key (PMAK) used for all standard Postman API operations — creating workspaces, uploading specs, generating collections, exporting artifacts, and managing environments.

To generate one:
1. Open the Postman desktop app or web UI.
2. Go to Settings (gear icon) -> Account Settings -> API Keys.
3. Click Generate API Key, give it a label, and copy the key (Starts with PMAK- ).
4. Set it as a GitHub secret:

```
gh secret set POSTMAN_API_KEY --repo <owner>/<repo>
```

> **Note:** The PMAK is a long-lived key tied to your Postman account. It does not require periodic renewal like the `postman-access-token`.

## Obtaining postman-access-token (Open Alpha)
> **⚠️ Open-alpha limitation:** The `postman-access-token` input requires a manually-extracted session token. There is currently no public API to exchange a Postman API key (PMAK) for an access token programmatically. This manual step will be eliminated before GA.

The postman-access-token is a Postman session token (x-access-token) required for internal API operations that the standard PMAK API key cannot perform — specifically workspace ↔ repo git sync (Bifrost), governance group assignment, and system environment associations. Without it, those steps are silently skipped during the onboarding pipeline.
**To obtain and configure the token:**

1. Log in via the Postman CLI (requires a browser):
```
postman login
```
This opens a browser window for Postman’s PKCE OAuth flow. Complete the sign-in.
2. Extract the access token from the CLI credential store:
```
cat ~/.postman/postmanrc | jq -r '.login._profiles[].accessToken'
```
3. Set it as a GitHub secret on your repository or organization:
```
# Repository-level secret
gh secret set POSTMAN_ACCESS_TOKEN --repo <owner>/<repo>

# Organization-level secret (recommended for multi-repo use)
gh secret set POSTMAN_ACCESS_TOKEN --org <org> --visibility selected --repos <repo1>,<repo2>
```
Paste the token when prompted.

> **Important:** This token is session-scoped and will expire. When it does, operations that depend on it (workspace linking, governance, system environment associations) will silently degrade. You will need to repeat the login and secret update process. There is no automated refresh mechanism.

> **Note:** `postman login --with-api-key` stores a PMAK — **not** the session token these APIs require. You must use the interactive browser login.

## Output Mapping

The composite action wires:
* workspace-id, workspace-url, spec-id, and collections-json from bootstrap.
* environment-uids-json, mock-url, monitor-id, repo-sync-summary-json, and commit-sha from repo_sync.
* Existing-service passthrough inputs to bootstrap: workspace-id, spec-id, baseline-collection-id, smoke-collection-id, and contract-collection-id.
* Existing-repo passthrough inputs to repo_sync: generate-ci-workflow and ci-workflow-path.
See action.yml for exact step mappings.

# Unique Considerations 

In the event the user has alternative setups or restrictions, the following can be reviewed.

## Github CLI is Utilized Instead of Actions

Instead of creating a workflow, developers utilizing Github CLI can write a script (utilizing Node or Python, for instance) that utilizing the Postman API with the same inputs. This script can be run through the GitHub CLI at the expected point in the workflow. Blow are the fields that would be added (and edited) to accomodate for the correct varaibles. (see other for file?)

```
{
  "project_name": "payment-refund-api",
  "domain": "payments",
  "domain_code": "PAY",
  "spec_url": "https://raw.githubusercontent.com/<owner>/<repo>/main/specs/payment-refund-api-openapi.yaml",
  "environments_json": "[\"prod\",\"stage\"]",
  "env_runtime_urls_json": "{\"prod\":\"https://api.payments.example.com/v2\",\"stage\":\"https://api-uat.payments.example.com/v2\"}",
  "github_repo": "<owner>/<repo>",
  "ref": "main",
  "workspace_id": "",
  "postman_api_key": "",
  "gh_fallback_token": ""
}
```

## Differing Authentication Patterns

In the event that different authentication patterns are utilized (i.e. API Keys, OAuth, JWT, mTLS, etc.):
- Ensure the README.md contains which authtype is utilized
- Which Postman environment variables need to be set
- That the spec must define 'security' so gernetated requests utilzied the right authentication
- If using Newman, ensure the runner can provider secrets for each authentication type (i.e. API Key, OAuth client credentials, etc.)

## Mixed Infrastructure

In the event that infrastructre is handled differently (i.e. such as Lambda, Kubernetes, etc.), treat it as an implementation detail. For each service and environment, define one base URL and ptu it in env-runtime-urls-json.
If one repo hosts multiple services, each should have its own job or workflow with its own set of the following variables:
* project-name
* spec-url
* env-runtime-urls-json
* (Optional) a shared workspace-id

## Service Missing an OpenAPI Spec 

In the event there is no OpenaAPI specification in the repository, you can add a spec-generation step that runs in GitHub Actions, produces an OpenAPI file and commits it back to the repo. The newly made specification can then be referenced in Postman via a saved spec-url.

The workflow will need to be updated to accomodate for the missing specification and also either set a source, runtime_url or collection_path for the trigger to run properly OR a manual choosing of the source/URL/collection path to be run. Some additional notes:

- **Secrets:** `GH_PAT` or `GH_FALLBACK_TOKEN` with `contents: write` so the workflow can push the new file. Optionally `POSTMAN_API_KEY` when fetching a collection from the Postman API.
- **Trigger:** Does **not** run on changes under `specs/` (to avoid loops). Runs on `workflow_dispatch` or on push to source paths (e.g. `src/`, `postman/collection.json`, or the config file).

### Guidance

- The workflow_dispatch should include the following:
```
spec_output_name:
        description: 'Output filename (under specs/) without path, e.g. payment-refund-api-openapi.yaml'
        required: true
        default: 'payment-refund-api-openapi.yaml'
      runtime_url:
        description: '(source=runtime_url) URL of the OpenAPI endpoint, e.g. https://api.example.com/v3/api-docs'
        required: false
      runtime_url_auth_header:
        description: '(optional) Auth header value, e.g. Bearer TOKEN (store token in a secret and pass here)'
        required: false
      collection_path:
        description: '(source=postman_collection) Repo path to collection JSON, e.g. postman/collection.json'
        required: false
      postman_collection_uid:
        description: '(source=postman_collection) Postman collection UID if fetching via API (set POSTMAN_API_KEY secret)'
        required: false
```

- The push->paths_ignore should include the following:
```
paths-ignore:
      - 'specs/**'
      - '**.md'
      - 'docs/**'
```

- Within the generate-and-commit section, there should bewaccomodations to account for creating the Specification, fetching it, and commiting it. See [the example](https://github.com/peterparalikas-rgb/payments-refund/blob/main/.github/EXAMPLE_workflowMissingSpec.yml) for further information.

# Specific Accomodations

## AWS Utilization
* Store following secrets in Github Action Secrets or Gitlab CI variables (masked):
```
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
(Optional) AWS_SESSION_TOKEN
```
* Configure GitHub/GitLab and AWS IAM to allow workflow to assume a role. Use that role in the 'Configure AWS Credentials' step
* Put non-sensitive data (i.e. account ID, region, URLs) in config or workflow files so operators know which account/region is in use

> If multiple accounts exist, you may need separate credentials or role assumptions for each
> For API Gateway/ALB URLs: Ensure the URL put into env-runtime-urls-json is one the clients will use

## ECS behind Amazon ALB

* Set env-runtime-urls-json (and runtime_url for spec generation) to the full base URL of the API as seen by the client: e.g. https://api.payments.example.com/v2 or https://payments-alb-xxx.elb.amazonaws.com/v2. Include any path prefix the ALB routes to this service.
* If the ALB uses a private CA, store the CA bundle (e.g. ALB_CA_BUNDLE or MTLS_CA_BUNDLE) and use it in the "Fetch spec from runtime URL" step and in Newman (e.g. NODE_EXTRA_CA_CERTS or --cacert).
* If the API behind the ALB requires auth (API key, Bearer token, etc.), provide it via runtime_url_auth_header or Postman environment variables; the ALB itself does not add auth unless you use a custom auth Lambda or Cognito.

> Postman Cloud and public GitHub/GitLab runners cannot reach an internal ALB. Use a self-hosted runner in the VPC, a Postman agent in the same network, or a bastion/proxy that exposes the API (with appropriate security).
> If the ALB has sticky sessions or AWS WAF in front, ensure health checks and the OpenAPI/docs path are allowed by WAF rules.
> If one ALB routes to several ECS services by path or host, each service has a different base URL; use separate spec-url and env-runtime-urls-json entries per service (or separate workflows).


## GitLab Access

* Create a Project or Group Action Token with write_repository; store in GitLab CI/CD variables as masked and optionally protected
* Recreted the GitHub Actions steps in .gitlab-ci.yml: e.g. One job to generate the spec (if needed) and push, another to run Postman Onboarding (Script or API calls). Reuse the same inputs
* The existing postman-onboarding-gh.sh can be adapted for GitLab by swapping gh for GitLab API/CLI and using CI_PROJECT_PATH, CI_COMMIT_REF_NAME, and GitLab variables.

> Self-managed GitLab: Raw file URLs may require authentication or be internal-only; Postman must be able to reach the spec URL (or use a public mirror / artifact URL).
> Protected branches: Pushing from CI often requires a token with "maintainer" or equivalent rights; branch protection rules must allow pushes from the CI user.

## Kubernetes

* Put the client-visible base URL (what Postman/Newman would use) into env-runtime-urls-json and, for spec generation, into runtime_url. That might be the Ingress hostname or an internal URL if the runner is in the same network.
* If the API is only reachable inside the cluster, use a self-hosted runner (or GitLab runner) that runs inside the cluster or in a network that can reach the Ingress/internal service.
* If using an internal CA, provide the CA bundle (e.g. as a secret or file) and use it in the "Fetch spec from runtime URL" step (e.g. curl --cacert ca.pem) and in any Newman run.

> Postman Cloud cannot reach internal URLs; use a self-hosted Postman agent or run collections from a runner that has network access.
> If URLs change per deployment (e.g. new Ingress each release), consider a stable DNS or gateway in front and document how to discover the current URL (e.g. from K8s or a config service).

## 1. Chained Workflows vs. Single Monolithic Workflow

| Aspect | This approach (chained) | Single monolithic workflow |
|--------|-------------------------|----------------------------|
| **Clarity** | Each workflow has one job; generate vs. onboarding vs. downstream are separate. | One file does everything; easier to “run once” mentally. |
| **Reuse** | Generate-spec can be reused by other repos; onboarding can run without generate when spec already exists. | Logic is tied together; harder to reuse “just onboarding” or “just generate.” |
| **Trigger control** | Path-based triggers avoid loops (e.g. no trigger on `specs/` for generate). | Must carefully design path filters and job ordering in one file. |
| **Debugging** | Failures are isolated to one workflow; logs are scoped. | One long run; need to scroll to find which step failed. |
| **Maintenance** | Two (or more) workflow files to keep in sync (e.g. branch, tokens). | One file to edit; but that file can become large and brittle. |

**Trade-off:** Chaining favors **reuse, clarity, and isolation** at the cost of **more files and ensuring triggers (push vs. manual) are coordinated**. A single workflow is simpler to “run one thing” but less flexible when only part of the pipeline is needed.

## 2. Spec Generation in CI vs. Spec-Only / Manual Spec

| Aspect | This approach (generate in CI, then commit) | Spec already in repo / manual only |
|--------|--------------------------------------------|------------------------------------|
| **No spec today** | Can start from runtime URL or Postman collection; spec is produced and committed automatically. | Team must add a spec by hand or via another process before onboarding runs. |
| **Single source of truth** | Spec lives in repo; onboarding reads from raw URL. Repo is the source. | Same if they already commit specs; no gain/loss. |
| **Secrets and auth** | Generate step may need PAT (push), optional auth for runtime URL, optional Postman API key. | Fewer secrets; onboarding only needs Postman tokens and spec URL. |
| **Loops** | Must avoid triggering generate on `specs/` changes; path filters and config required. | No generate step, so no loop risk. |
| **Operational load** | More moving parts: config file for push-triggered generate, PAT, branch permissions. | Lower; “just point at a spec URL.” |

**Trade-off:** In-repo spec generation **unblocks teams that don’t have a spec yet** and keeps the spec in version control, at the cost of **extra workflow logic, tokens, and loop prevention**. If the spec already exists or is maintained elsewhere, a “spec-only” onboarding workflow is simpler.

## 3. GitHub Actions vs. Other CI (GitLab, Jenkins, etc.)

| Aspect | This approach (GitHub Actions) | GitLab CI / Jenkins / other |
|--------|--------------------------------|-----------------------------|
| **Ecosystem** | Native to GitHub; `workflow_dispatch`, `repository_dispatch`, Actions marketplace. | Same logic can be reimplemented; no direct reuse of YAML. |
| **Secrets** | GitHub Secrets and variables; well documented for this flow. | GitLab CI variables, Jenkins credentials; need to map and document. |
| **Triggers** | Push paths, manual run, workflow_run. | GitLab: rules, changes; Jenkins: triggers, pipeline params. |
| **Portability** | Shell script (`postman-onboarding-gh.sh`) supports “trigger” and “local” so non-Actions users can still run onboarding. | Full portability requires rewriting workflows and possibly the script for each system. |

**Trade-off:** This design is **optimized for GitHub**. It’s **easier to adopt** for GitHub-centric teams and **harder to drop in** for GitLab/Jenkins without reimplementing the same steps and secret handling (see `SETUP_ADDITIONAL_REQUIREMENTS.md` for GitLab notes).

## 4. Push-Triggered vs. Manual-Only Workflows

| Aspect | This approach (push + manual) | Manual-only |
|--------|--------------------------------|-------------|
| **Automation** | Spec change or config change can trigger generate; spec push can trigger onboarding. | Every run is explicit (Actions UI or `gh workflow run`). |
| **Predictability** | Runs can happen on any qualifying push; need path filters and branch rules. | No surprise runs; full control over when things run. |
| **First-time / one-off** | Manual run still supported; push is optional. | Only option is manual (or scheduled). |
| **Config** | Push-triggered generate needs `.github/spec-generation-config.yaml` and correct paths. | No config file needed for triggers; inputs only when running. |

**Trade-off:** Supporting **both** push and manual gives **“spec in repo → auto onboarding”** but adds **trigger and config discipline**. Manual-only is simpler and safer for teams that want strict control and fewer accidental runs.

## 5. Multiple Environments (prod/stage) in One Workflow vs. One Workflow per Env

| Aspect | This approach (envs in JSON in one run) | Separate workflow/job per environment |
|--------|------------------------------------------|----------------------------------------|
| **Setup** | One set of inputs: `environments-json`, `env-runtime-urls-json`; one run creates/updates all. | One run per env; clearer “this run is prod only.” |
| **Safety** | Same workflow can touch prod and non-prod; approval gates must be in the workflow or branch rules. | Can restrict “prod” workflow to protected branch or manual only. |
| **Flexibility** | Add/remove envs by editing JSON; no new workflow file. | New env often means new job or workflow. |

**Trade-off:** **One workflow, multiple envs** is **simpler to configure and extend** (add an env = update JSON). **One workflow per env** gives **stronger separation** and easier approval/audit (e.g. “prod workflow” only on main with manual approval).

# Changelog
## 0.2
Initial attempt involved reviewing documentation regarding integration, workflow function with Postman. Crafted prompts for Cursor for followup understading & forming a checklist to go through to set up this process.
## 0.4
Initial succcess achieved with creation of workspace & relevant data. Issue found that a new workspace is made each run. Updating workflow to properly point to existing workspace. Future development will instead properly swap over to an automated collection of IDs to prevent manual effort. 
## 0.6
Verified functionality and return to repo. Rereviewed documentaiton to ensure no missed steps.
## 0.8
Additional notes on universal vs specific setup added for multiple scenarios. Included updates around customer specific steps where needed.
## 1.0
Documentation finalized; variant notes, onboarding notes, verification and trade-offs added.

# Issues Encountered
- Experienced multiple issues attempting to update Postman account. Failure due to Org level restrictions. Adapted and used a separate account and utilized Enterprise Trial to complete process.
- Chained Action failed first go around. Discovered missed URL in payments-refund workflow
