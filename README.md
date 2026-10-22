# poc-cloudruns

PoC of Monorepo of Google Cloud Run Functions.

## Overview

- Manage resources using Terraform
- Deploy function code using gcloud

## Directory Structure

```
functions/
  <function>/
    Makefile
    function.yaml
    <env>/
      env.yaml
    code/
      <function code>
```

Example:

```
functions/
  hello/
    Makefile
    function.yaml
    prd/env.yaml
    stg/env.yaml
    code/
      main.py # A function implemented in Python
```

### function.yaml

Defines environment-independent settings.  
However, most settings depend on the environment, so they are defined in env.yaml instead.

```
slack_channel_id:
  contact: Contact for review requests and so on
```

### env.yaml

Contains the metadata required for deployment.  
It is also used as a marker indicating the root directory of a function.

```
function_name: function name
auth: # Inputs of google-github-actions/auth
  service_account: my-service-account@my-project.iam.gserviceaccount.com # Service Account for deployment
  workload_identity_provider: projects/123456789/locations/global/workloadIdentityPools/my-pool/providers/my-provider # Workload Identity Provider
sha: The commit SHA of the function to deploy. Ignored in staging
deploy: # Arguments of gcloud run deploy
  function: The value of --function
  region: The value of --region
slack_channel_id:
  deploy: Slack Channel notified on deployment
```

`service_account` and `workload_identity_provider` are passed as inputs of `google-github-actions/auth` to impersonate the SA for deployment.

- CI fails if service_account is not unique across the Monorepo
- CI fails if the pair of the project number of workload_identity_provider and function_name is not unique across the Monorepo

https://github.com/google-github-actions/auth#inputs-workload-identity-federation

The documentation of `google-github-actions/auth` recommends Direct Workload Identity Federation, but Cloud Run doesn't support it, so we use SA impersonation.

https://docs.cloud.google.com/iam/docs/federated-identity-supported-services#cloud-run-functions

> Cloud Run doesn't support Workload Identity Federation. To grant access, use [service account impersonation](https://docs.cloud.google.com/iam/docs/workload-identity-federation#impersonation).

### Makefile

Defines tasks such as test and format run in CI.  
Since commands differ per function, they can be freely defined with Make.  
Makefile is adopted because it is language-agnostic.  
Only the test task is required, and the others are optional. At least minimal unit tests should exist, but if there are none, write dummy code such as `echo test`.

- `make fix`: Code formatting. Fixes are applied automatically in CI.
- `make pre-test`: Run in CI before tests. e.g. installing dependencies
- `make dep`: dependency
- `make test`: test. If you run integration tests, linters, and so on in addition to unit tests, include them as well

Since Makefile can run arbitrary code, jobs that run it in CI are separated and not granted unnecessary permissions.

## Environments

There are two environments for functions: production and staging.  
Environments such as development and integration are not considered.

- production: Production environment
- staging: Environment for testing. Since there is only one per function, you can't deploy and test multiple versions of code at the same time

### Can we add more environments for testing?

As described above, there is only one environment for testing: staging.  
The following additional environments could be considered, but they are not adopted.

1. An environment per PR
    1. It's hard to dynamically create dependent resources per PR and destroy them when the PR is merged
2. A dedicated environment for production release PRs
    1. It's acceptable to deploy to staging and occupy it until the PR is merged
3. An environment for the main branch
    1. It's possible to prepare, but it's doubtful that it would be used effectively

To make a testing environment meaningful, we need to create not only the function but also its dependent resources and test them end-to-end.  
So it would be hard to prepare one per PR.  
Environments for the main branch and for production release PRs could be prepared, but if functions aren't developed that actively, one environment should be enough.  
Preparing these environments uniformly for all functions would be very wasteful.  
We start with only one environment, and for functions that need environments 2 or 3, we will support them by setting a flag in env.yaml.

## Resources

- Function: Created per environment
    - SA for execution: Created per function
    - SA for deployment: Created per function
- Workload Identity Pool: Created per project
- Workload Identity Provider: Created per project
    - For production, impersonation is allowed only from the main branch
    - For staging, impersonation is allowed only from workflows on the main branch via job_workflow_ref, which prevents abuse of the SA for staging

## Branching Strategy

- default branch
- feature branch
- release branch: `release/<project id>/<function name>` Created by workflow_dispatch when releasing to production.
    - Project ID is a globally unique, immutable, and human-readable identifier. It is different from project number and project name

## How to Deploy

Run workflow_dispatch on GitHub Actions to deploy explicitly.

Impersonate a Service Account with Workload Identity Federation.

- https://docs.cloud.google.com/iam/docs/workload-identity-federation
- https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-google-cloud-platform

WIF parameters are read from env.yaml at deployment.

At deployment, the SHA is recorded as a label of the function, so that the SHA of the currently deployed code is known and the diff can be seen when releasing to production.

### staging

Running workflow_dispatch with a branch specified deploys it.  
If an open release PR exists, the deployment fails without deploying.  
It fails if env.yaml doesn't exist on the main branch, so you need to create env.yaml on the main branch before the first deployment.

### production

Running workflow_dispatch updates the sha in env.yaml and creates a PR from the release branch to the main branch.  
If an open PR already exists, it fails without creating a PR.  
If the PR is closed but the branch remains, the branch is deleted and recreated.  
A PR is used because it would be a problem if we couldn't see the diff when releasing.  
The PR description contains the diff (a list of PRs) compared with the SHA in the function's label.  
When the PR is merged and env.yaml is updated on the main branch, the function is deployed.

CI verifies whether the SHA is on the main branch and fails otherwise, preventing code not on the main branch from being deployed.

## GitHub Actions Workflows

1. deploy_staging.yaml: Deploys a function to staging via workflow_dispatch. It only calls the reusable workflow `wc_deploy_staging.yaml` on main
2. wc_deploy_staging.yaml: The reusable workflow that implements deploy_staging.yaml
3. create_release_pr.yaml: Creates a PR for releasing to production via workflow_dispatch and deploys to staging
4. release.yaml: Deploys functions to production on push events to the main branch
5. test.yaml: Runs tests of changed functions on pull_request events

## Dependency Updates

Dependency updates are done with Renovate.  
Dependabot is also an option, but using both causes conflicts and Renovate is more flexible to configure, so we adopt Renovate.  
Since this is a Monorepo, PRs are created per service.  
Each service's owners handle them.

## Security Considerations

- Separate the execution SA per Function and grant only the minimum permissions
- Obtain deployment permissions by impersonating an SA with WIF, and don't grant permissions to self-hosted runner Nodes
- Restrict WIF conditions to prevent unauthorized impersonation
    - Restrict production releases to the main branch only
        - `ref == "refs/heads/main"`
        - `event_name == "push"`
        - `workflow_ref == "<org>/<repo>/.github/workflows/release.yaml@refs/heads/main"`
    - Restrict staging releases to the reusable workflow on the main branch with job_workflow_ref
        - `job_workflow_ref == "<repo>/.github/workflows/<reusable>.yaml@refs/heads/main"`
- Separate the deployment SA per Function and grant only the minimum permissions
- Separate jobs that run Makefile and grant only the minimum permissions
    - `id-token:write` is normally unnecessary. Granting it allows impersonation via WIF
    - Code fixed by `make fix` is uploaded to GitHub Artifacts and pushed with Securefix Action in a subsequent job, so that `id-token:write` isn't granted to the job running `make fix`
    - If a function depends on another internal repository, it is checked out in a separate job and shared via GitHub Artifacts, so that `id-token:write` isn't granted to the job running make
- Validate the GitHub Actions actor at deployment so that only codeowners can deploy
    - Refer to CODEOWNERS on the main branch
    - Getting members from a team requires a GitHub App token with the members:read permission
- When deploying, read env.yaml from the main branch to prevent unauthorized deployments by rewriting function_name, service_account, or workload_identity_provider on a feature branch
- In production releases, fail without deploying if the sha isn't on the main branch
- Set codeowners per function
- Set codeowners for GitHub Actions Workflows too
- Introduce validate-pr-review-app to prevent unauthorized merges of PRs into the main branch
- Branch Ruleset
    - main branch
        - Restrict deletions
        - Require signed commits
        - Require a pull request before merging
            - Dismiss stale pull request approvals when new commits are pushed
            - Require review from Code Owners
            - Require approval of the most recent reviewable push
        - Require status checks to pass
        - Block force pushes

## Implementation Notes

- Run `gcloud run deploy`. Implementing it with the Go SDK is quite tedious
    - https://pkg.go.dev/cloud.google.com/go/run@v1.23.0/apiv2
    - https://docs.cloud.google.com/sdk/gcloud/reference/run/deploy

```
gcloud run deploy "${NAME}" --region "${REGION}" \
  --source code/ --function "${ENTRY_POINT}" \
  --labels "commit-sha=${SHA}" \
  --revision-suffix "${SHA:0:7}-${GITHUB_RUN_NUMBER}"
```
