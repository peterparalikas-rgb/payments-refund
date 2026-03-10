# payments-refund
Comprehensive API for managing payment refunds in a financial services platform. This API handles refund creation, status tracking, cancellation, and reporting for payment transactions processed through the authorization and capture workflow. Said workflow triggers followup run to sync the loan-origination-api-openapi.yaml file upon successful sync of the payment-refund-api-openapi.yml

# Universal Considerations

## Inputs

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

Note: The PMAK is a long-lived key tied to your Postman account. It does not require periodic renewal like the postman-access-token.

## Obtaining postman-access-token (Open Alpha)
⚠️ Open-alpha limitation: The postman-access-token input requires a manually-extracted session token. There is currently no public API to exchange a Postman API key (PMAK) for an access token programmatically. This manual step will be eliminated before GA.

The postman-access-token is a Postman session token (x-access-token) required for internal API operations that the standard PMAK API key cannot perform — specifically workspace ↔ repo git sync (Bifrost), governance group assignment, and system environment associations. Without it, those steps are silently skipped during the onboarding pipeline.
To obtain and configure the token:
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

Important: This token is session-scoped and will expire. When it does, operations that depend on it (workspace linking, governance, system environment associations) will silently degrade. You will need to repeat the login and secret update process. There is no automated refresh mechanism.

Note: postman login --with-api-key stores a PMAK — not the session token these APIs require. You must use the interactive browser login.

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
