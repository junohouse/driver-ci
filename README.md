# Juno driver CI

The one workflow every certified driver calls. A driver repo's own workflow is this:

```yaml
name: release
on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  driver:
    uses: junohouse/driver-ci/.github/workflows/driver.yml@v1
    permissions:
      contents: write
    secrets: inherit
```

Push to `main` for a beta. Tag `v1.2.0` for a release. That is the whole interface.

The `permissions` block is not optional. Publishing writes releases, a called workflow cannot
request more permission than its caller was granted, and the default `GITHUB_TOKEN` is
read-only — so leaving it out fails the run before any job starts, with an error naming this
workflow rather than the caller that is actually missing a line.

## What you get

| Trigger | Channel | Version | Where it lands |
| --- | --- | --- | --- |
| push to `main` | beta | `1.2.0-beta.41` | rolling `beta` prerelease, `beta.json` |
| tag `v1.2.0` | stable | `1.2.0` | release `v1.2.0`, `index.json` |

A beta version sorts *below* the release it previews, so a controller following stable can
never resolve to one by accident.

## What it actually does

1. Builds the driver as a native library on macOS **and** Linux, with no credentials. Both go
   in one package, so a controller installs the same artifact whichever it runs.
2. Validates the manifest against core's proxy contracts. This is the certification step —
   `junod pack` refuses to write an archive it would not install, so there is no way to
   publish while skipping it.
3. Packages, digests, and publishes to the driver's own releases — which is what a controller
   downloads from. There was a step here that mirrored the artifact into a shared `artifacts`
   repo so a controller had one public host regardless of whether the driver's repo was
   public. Every driver repo is public and already carries the file, so that was a second copy
   of a published artifact, kept in step by hope.
4. Tells the registry what it published. The registry merges what it is handed and scans
   nothing.

## When a driver repo needs to be private

Step 3 assumes the release it writes is one a controller can read. A private repo breaks that
and nothing else: a house has no token and must never need one, so the *download* has to move
somewhere anonymous while the build, the certification and the dispatch stay exactly as they
are.

The one public host already in the picture is `driver.juno.house` — controllers read the index
from it, so serving payloads from it too means core and the catalog need no notion of where a
driver came from. The seam is this step's `url` output: whatever publishes the payload writes
the URL there, and the rest of the pipeline does not care.

Put the payloads in an R2 bucket behind that domain rather than in the registry repo. Object
storage keeps binaries out of git, and the registry's own site is redeployed wholesale on every
publish — a place to serve files from, not to accumulate them.

Restoring the deleted mirror repo would also work and is less setup, but it recreates a second
copy of an artifact for every driver to solve it for the few that are private.

## Inputs

| Input | Default | Why you would change it |
| --- | --- | --- |
| `packages` | `'["."]'` | A repo shipping more than one package: `'["cloud", "hap"]'` |
| `core-ref` | `main` | The core ref to build and validate against |
| `registry-repo` | `junohouse/registry` | |

## Secrets

Set these once as **organization** secrets on `Juno-Certified-Drivers`, not per repo.

| Secret | Needs |
| --- | --- |
| `CORE_TOKEN` | Read access to `junohouse/core`, for the packaging step |
| `REGISTRY_TOKEN` | Dispatch into `junohouse/registry` |

Building a driver needs **no credentials** — it depends only on the public
[`driver-sdk`](https://github.com/junohouse/driver-sdk). `CORE_TOKEN` is used
only by the publish job, which runs `junod` to validate the manifest against the proxy
contracts and to build the package.

## One package or two?

A package with one driver keeps it in `manifest.toml`. A package with several puts them **all**
in `manifests/` — one place to look, and no file privileged by where it sits. Which one leads
is then stated outright: `primary = true` in `[driver]`, or inferred from a `parent`
relationship where one exists.

Bundle drivers into one package when they **share control code** — a Hue bridge and its bulbs,
a Roku TV and a Roku player, an ecobee thermostat and its remote sensors. One payload carries
several manifests, the registry lists each driver separately, and installing any of them
installs the package. That is what stops a bridge and the devices behind it from drifting
apart in version.

Ship two packages when they share only a vendor. Caséta's telnet and LEAP paths are different
protocols for different hardware, and a single payload would have to guess from instance state
which one a command meant. `on_command` is not given a driver id, so that guess is exactly as
fragile as it sounds.

## This repo is public on purpose

A reusable workflow in a private repo cannot be called from another organization, and the
drivers live in `Juno-Certified-Drivers`. There are no secrets in this file — they are passed
in by the caller.
