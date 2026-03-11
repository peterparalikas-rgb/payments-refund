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

# Changelog
## 0.2
Initial attempt involved reviewing documentation regarding integration, workflow function with Postman. Crafted prompts for Cursor for followup understading & forming a checklist to go through to set up this process.
## 0.4
Initial succcess achieved with creation of workspace & relevant data. Issue found that a new workspace is made each run. Updating workflow to properly point to existing workspace. Future development will instead properly swap over to an automated collection of IDs to prevent manual effort. 
## 0.6
Verified functionality and return to repo. Rereviewed documentaiton to ensure no missed steps.
## 0.8
Additional notes on universal vs specific setup added for multiple scenarios. Included updates around customer specific steps where needed.

# Issues Encountered
- Experienced multiple issues attempting to update Postman account. Failure due to Org level restrictions. Adapted and used a separate account and utilized Enterprise Trial to complete process.
- Chained Action failed first go around. Discovered missed URL in payments-refund workflow
