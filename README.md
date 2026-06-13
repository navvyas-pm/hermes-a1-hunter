# hermes-a1-hunter

A scheduled GitHub Actions workflow that hunts for free **Oracle Cloud Ampere A1**
capacity across **multiple regions** and launches the instance the moment a slot
opens.

Each run sweeps a list of regions in rapid succession using a **direct compute
launch** (`oci compute instance launch`) — one independent attempt per region,
per availability domain. The first region with capacity wins. On success it
attaches a public IP, opens a GitHub issue with the `ssh` line, and disables its
own schedule.

Runs entirely in the cloud — no local machine needs to be on. cron fires every
5 minutes as a backstop, and each run re-dispatches itself for continuous cover.

## Regions

Hunted in order (best A1 availability first), configurable via the `REGIONS`
env var in [`hunt.yml`](.github/workflows/hunt.yml):

```
us-phoenix-1  us-ashburn-1  ca-toronto-1  uk-london-1  ap-mumbai-1  us-sanjose-1
```

> This is a **Pay As You Go** account, where the A1 free monthly allowance
> (3000 OCPU-hours / 18000 GB-hours) applies in any region — so it hunts several
> at once. On a pure *Always Free* account, A1 can only be created in the
> tenancy's **home region**; trim `REGIONS` to just that one if you downgrade.

## How it works

- **No Resource Manager stack.** The previous version re-applied a saved stack,
  which holds one shared Terraform state and can't hunt regions in parallel.
  This version launches instances directly, so each region is independent.
- **Sequential, not a parallel matrix** — on purpose. The Always Free allowance
  is **4 OCPU / 24 GB total per tenancy**; sweeping regions one at a time (plus
  the `concurrency` guard) guarantees we never accidentally land two instances.
- **Self-provisioning networking.** If a region has no `hermes-vcn`, the workflow
  creates a VCN + internet gateway + route table + security list (SSH/22 open) +
  public subnet on the fly. Idempotent — reused on later runs.
- **Dynamic image lookup.** The latest Canonical Ubuntu 22.04 **aarch64** image
  OCID is resolved per region at run time.
- **Shape fallback.** Each AD first tries the full **4 OCPU / 24 GB**
  `VM.Standard.A1.Flex`; on out-of-capacity it immediately retries **2 OCPU /
  12 GB** in the same AD before moving on. It never goes lower — 2/12 is the
  floor for running local Ollama inference. Tiers are set via `SHAPE_CONFIGS`.
- **Off-peak weighting.** During US off-peak (02:00–06:00 Pacific = 09:00–13:00
  UTC) it runs more sweeps with a tighter gap (`OFFPEAK_SWEEPS` / `OFFPEAK_GAP`),
  since free capacity is likeliest to open up then. Peak hours stay on the
  relaxed `SWEEPS` / `GAP` cadence. The `*/5` cron + self-re-dispatch are
  unchanged — coverage still comes from frequent runs.

## Secrets (repo → Settings → Secrets and variables → Actions)
- `OCI_CLI_USER` — user OCID
- `OCI_CLI_TENANCY` — tenancy OCID
- `OCI_CLI_FINGERPRINT` — API key fingerprint
- `OCI_CLI_KEY_CONTENT` — API **private** key PEM contents
- `SSH_PUBLIC_KEY` — your SSH **public** key, injected so you can log in as `ubuntu`

> `OCI_STACK_ID` is no longer used and can be removed.

## Manual run
Actions tab → **hermes-a1-hunt** → **Run workflow**.

> Security note: secrets are encrypted and never printed in logs. For least
> privilege, use a dedicated IAM user scoped to **compute + virtual-network-family**
> rather than a full-access key.
