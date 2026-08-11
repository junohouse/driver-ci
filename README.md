# Juno driver CI

Build, validate and package a Juno driver. Public, and needs no credentials — anyone can call
this on their own driver and get exactly the build and exactly the validation a first-party
driver gets.

A driver repo's own workflow is this:

```yaml
name: release
on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  driver:
    uses: junohouse/driver-ci/.github/workflows/driver.yml@main
    permissions:
      contents: write
```

Push to `main` for a beta. Tag `v1.2.0` for a release. That is the whole interface.

The `permissions` block is not optional while `release` is on. Publishing writes releases, a
called workflow cannot request more permission than its caller was granted, and the default
`GITHUB_TOKEN` is read-only — so leaving it out fails the run before any job starts, with an
error naming this workflow rather than the caller that is actually missing a line.

## What you get

| Trigger | Channel | Version | Where it lands |
| --- | --- | --- | --- |
| push to `main` | beta | `1.2.0-beta.41` | rolling `beta` prerelease on your repo |
| tag `v1.2.0` | stable | `1.2.0` | release `v1.2.0` on your repo |

A beta version sorts *below* the release it previews, so a controller following stable can
never resolve to one by accident.

## What it actually does

1. Builds the driver as a native library on macOS **and** Linux, x86-64 and ARM, with no
   credentials. All of them go in one package, so a controller installs the same artifact
   whichever it runs.
2. Validates the manifest against the real proxy contracts. `junodrv pack` refuses to write an
   archive it would not install, so the build step and the validation step are the same step
   and there is no way to skip one.
3. Packages and digests it, and uploads the result as a workflow artifact — `pkg-<package>`,
   holding the `.junodrv`, its `.sha256`, and a `meta.json` describing the build.
4. Publishes it to your repository's own releases, unless you turn `release` off.

## Inputs

| Input | Default | Why you would change it |
| --- | --- | --- |
| `packages` | `'["."]'` | A repo shipping more than one package: `'["cloud", "hap"]'` |
| `sdk-ref` | `main` | Pin one build to a particular driver-sdk branch |
| `release` | `true` | Off when something downstream publishes the payload instead |
| `stamp-property` | `''` | A vendor integration key — see below |

`sdk-ref` is a branch, not a tag. The proxy contracts live in driver-sdk and move with it, so
validating against a pinned tag means a capability that plainly exists on `main` is reported as
unknown — which is the trade core stopped making when it dropped tags from its own dependency.

There are no required secrets. Building a driver depends only on the public
[driver-sdk](https://github.com/junohouse/driver-sdk), which depends on nothing of Juno's —
which is what makes a driver something an outsider can build, check, and ship for themselves.

## A vendor integration key

Some vendors issue a key per *integration* rather than per house — Sonos does. It has to reach
the driver, and it must not be committed to a public repo.

Set `stamp-property` to the name of a `[[property]]` in the manifest, and `VENDOR_KEY` as an
organization secret. The publish job writes the key into that property's `default` just before
packaging:

```yaml
with:
  stamp-property: Sonos API Key
secrets:
  VENDOR_KEY: ${{ secrets.SONOS_API_KEY }}
```

The manifest, not the source. The build job stays credential-free — which is what lets anyone
build a driver without secrets — rotating the key is a repackage rather than a recompile, and
nothing lands in git. A following step fails the build if the key also appears inside a
compiled library, because a second copy is one nobody can rotate.

This is **obfuscation, not secrecy**. The artifact is downloadable and anyone can read a
string out of it. Declare the property so an installer can set their own key instead, and
treat a shipped key as something that will eventually need rotating.

## What this workflow will not do

It does not touch the registry. Being listed at [driver.juno.house](https://driver.juno.house)
as **certified** is a claim about where an artifact came from, and a workflow anyone may call
cannot make that claim about itself.

Indexing lives in `certified.yml` in the private registry repo. It calls this one for the build
and the release, then dispatches the index row. A reusable workflow in a private repository can
only be called from inside its own organisation, so the provenance claim is enforced by GitHub
rather than by everybody agreeing not to.

That is a claim about provenance, **not a safety audit**, and it is worth being precise about
because the controller UI shows it to residents.

## Third-party drivers

Nothing above is required to write or ship a Juno driver. A controller will install any
`.junodrv` handed to it — it labels the driver *Third-party*, and never auto-updates it. Use
this workflow to build one, publish it on your own releases, and tell people the URL.

The documentation for writing one is at [docs.juno.house](https://docs.juno.house).

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

A reusable workflow in a private repo cannot be called from another organisation, and the point
of this one is that it can. There are no secrets in it — the private half passes its own in.
