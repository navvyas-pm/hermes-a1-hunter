# hermes-a1-hunter

A scheduled GitHub Actions workflow that hunts for free **Oracle Cloud Ampere A1**
capacity in `us-sanjose-1` and launches the instance the moment a slot opens.

It re-applies a saved OCI **Resource Manager stack** (`hermes-vps-stack`) every ~10
minutes. When it succeeds it attaches a public IP, opens a GitHub issue with the
`ssh` line, and disables its own schedule.

Runs entirely in the cloud — no local machine needs to be on.

## Secrets (repo → Settings → Secrets and variables → Actions)
- `OCI_CLI_USER` — user OCID
- `OCI_CLI_TENANCY` — tenancy OCID
- `OCI_CLI_FINGERPRINT` — API key fingerprint
- `OCI_CLI_KEY_CONTENT` — API **private** key PEM contents
- `OCI_STACK_ID` — the Resource Manager stack OCID

Region is hardcoded to `us-sanjose-1` in the workflow.

## Manual run
Actions tab → **hermes-a1-hunt** → **Run workflow**.

> Security note: the OCI key in secrets is encrypted and never printed in logs.
> For least privilege, replace it with a dedicated IAM user scoped to compute +
> resource-manager rather than a full-access key.
