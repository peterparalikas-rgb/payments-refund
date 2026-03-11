# payments-refund
Comprehensive API for managing payment refunds in a financial services platform. This API handles refund creation, status tracking, cancellation, and reporting for payment transactions processed through the authorization and capture workflow. Said workflow triggers followup run to sync the loan-origination-api-openapi.yaml file upon successful sync of the payment-refund-api-openapi.yml

# Workfllow Creation Process
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


# Changelog
## 0.2
Initial attempt involved reviewing documentation regarding integration, workflow function with Postman. Crafted prompts for Cursor for followup understading & forming a checklist to go through to set up this process.
## 0.4
Initial succcess achieved with creation of workspace & relevant data. Issue found that a new workspace is made each run. Updating workflow to properly point to existing workspace. Future development will instead properly swap over to an automated collection of IDs to prevent manual effort. 
## 0.6
Verified functionality and return to repo. Rereviewed documentaiton to ensure no missed steps.


# Issues Encountered
- Experienced multiple issues attempting to update Postman account. Failure due to Org level restrictions. Adapted and used a separate account and utilized Enterprise Trial to complete process.
- Chained Action failed first go around. Discovered missed URL in payments-refund workflow
