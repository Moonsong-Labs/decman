# Case study: building Decentralization Manager with `dpm.yaml`

This repository is a fork of
[Decentralization Manager](https://github.com/DLC-link/decentralization-manager)
(DecMan) whose Daml code builds with dpm-priv, the Moonsong Labs fork of `dpm`
that implements the baseline of the proposed `dpm.yaml` design. This document
compares the upstream build setup with the migrated one, so that `dpm`
maintainers can evaluate the design on a production workspace. The design
itself is specified in `pkg/dario/dpm-yaml-schema.md` in dpm-priv.

DecMan has 13 Daml packages that depend on each other and on 17 third-party
DARs, most of them from Splice and the Digital Asset (DA) utility packages.
Upstream commits those DARs to the repository and references them by relative
path.

## Summary

| Metric | Upstream (`daml.yaml`) | This fork (`dpm.yaml`) |
|---|---|---|
| Third-party DARs committed to the repository | 17 | 0 |
| Relative-path references to those DARs | 68 | 0 |
| Sibling references that hardcode a build path and version | 24 | 0 |
| Copies of the SDK version | 13 | 1 |
| Non-blank manifest lines | 277 | 484 |

Line counts cover every `daml.yaml` plus `multi-package.yaml` upstream, and
every `dpm.yaml` in this fork.

The migration removes committed build inputs and hardcoded paths, but makes the
manifests longer because each package declares its Git and OCI dependencies in
full. Workspace catalogs, planned for Phase 2, address that.

## Before: `daml.yaml` and `multi-package.yaml`

The baseline is upstream `main` at commit
[`b507e55`](https://github.com/DLC-link/decentralization-manager/commit/b507e5519c5d4bd8b8f5d0c6fe7c6548814fbf9f)
(DecMan v1.9.0). A `multi-package.yaml` lists 13 packages, each with its own
`daml.yaml`. This is `governance-utility-onboarding/daml.yaml`:

```yaml
sdk-version: 3.4.11
name: governance-utility-onboarding-v1
version: 0.4.0
source: daml
dependencies:
  - daml-prim
  - daml-stdlib
data-dependencies:
  - ../governance-action-v1/.daml/dist/governance-action-v1-0.1.0.dar
  - ../dars/splice-amulet-0.1.19.dar
  - ../dars/splice-api-featured-app-v1-1.0.0.dar
  - ../dars/splice-api-token-burn-mint-v1-1.0.0.dar
  - ../dars/splice-api-token-holding-v1-1.0.0.dar
  - ../dars/splice-api-token-metadata-v1-1.0.0.dar
  - ../dars/splice-api-token-transfer-instruction-v1-1.0.0.dar
  - ../dars/splice-util-0.1.4.dar
  - ../dars/utility-commercials-v0-0.4.1.dar
  - ../dars/utility-credential-app-v0-0.4.1.dar
  - ../dars/utility-credential-v0-0.1.0.dar
  - ../dars/utility-registry-app-v0-0.7.0.dar
  - ../dars/utility-registry-holding-v0-0.2.1.dar
  - ../dars/utility-registry-v0-0.6.0.dar
```

Four problems repeat across the workspace:

- **The SDK version is repeated.** All 13 files declare `sdk-version: 3.4.11`.
- **Sibling references encode a version.** The path
  `governance-action-v1-0.1.0.dar` includes the sibling's version, so bumping a
  package means editing every package that depends on it.
- **Third-party DARs are vendored.** Their origin and version live only in the
  filename.
- **Nothing detects drift.** `governance-token-custody-test` depends on
  `splice-amulet-0.1.17.dar`, while every other package uses `0.1.19`.

## After: `dpm.yaml`

The migration fixes the first three problems. Drift remains; see
[Manifest size and drift](#manifest-size-and-drift).

### A workspace manifest

[`daml/dpm.yaml`](daml/dpm.yaml) replaces `multi-package.yaml`. It declares the
members, the SDK version, and the registries:

```yaml
schema: 1

workspace:
  name: token-trading
  sdk: 3.4.11
  members:
    - pkg/*

  registries:
    query:
      main: oci://zot.oceano.2networks.tech
    publish:
      main: oci://zot.oceano.2networks.tech
```

Each package sets `sdk: workspace` and inherits the SDK version.

### Sibling dependencies by name

Each sibling reference becomes a single line:

```yaml
data-dependencies:
  governance-action-v1: member
```

A `member` dependency names another package in the same workspace. `dpm`
builds the sibling first, so the dependent compiles against the sibling's
current sources instead of a versioned build output. Packages keep independent
versions, because Daml upgrade compatibility is per package, but a version bump
no longer requires edits in dependents.

`member` resolution is implemented on the dpm-priv branch
`feat/member-dep-support` and isn't merged into `dario` yet.

### Third-party dependencies by source

Splice DARs come from the Splice Git repository, pinned to a commit. DA utility
DARs come from an OCI registry:

```yaml
data-dependencies:
  splice-amulet:
    git: https://github.com/canton-network/splice
    commit: d5b780b5b636fe46f6f77e87c7655ddaf53c50fc
    path: daml/dars/splice-amulet-0.1.19.dar

  utility-registry:
    registry: main
    version: 0.6.0
```

For this case study, we republished the DA utility DARs and `testlib` to an
OCI registry that we host. The registry allows anonymous reads.

An OCI `version` is a semantic-versioning requirement. `dpm` resolves it to the
highest published tag that satisfies it.

`daml-script` and `daml-stdlib` are declared as `builtin`, which means that the
SDK supplies them.

To build the workspace, run `dpm dario build` from `daml/` with a dpm-priv
build that includes `member` support.

## Open gaps

### Manifest size and drift

Manifests grew from 277 to 484 non-blank lines, and the same Splice commit
appears 40 times. The `splice-amulet` 0.1.17 drift carried over unchanged,
because each package still declares its own copy. Workspace catalogs let a
workspace declare each third-party dependency once:

```yaml
workspace:
  catalogs:
    splice:
      splice-amulet:
        git: https://github.com/canton-network/splice
        commit: d5b780b5b636fe46f6f77e87c7655ddaf53c50fc
        path: daml/dars/splice-amulet-0.1.19.dar
```

dpm-priv parses and validates `catalogs`. The syntax for referencing a catalog
entry from a package isn't defined yet.

### Mutable OCI tags

Git dependencies pin a commit. OCI dependencies resolve to a tag, and a
registry can move a tag to different content. The schema lets you pin an OCI
DAR with `digest`, but this repository doesn't, because maintaining digests by
hand across 13 packages is error-prone. A lockfile that records the resolved
OCI digest and Git commit for every dependency closes the gap without manual
pinning.

### Committed DARs outside the build

`releases/` holds first-party DARs that the DecMan server uploads to Canton at
runtime. They aren't build inputs, so this migration leaves them in place.
`daml/dars-to-publish/` holds the DARs used to seed the OCI registry.

## Open design questions

- How does a package reference a catalog entry? The schema reserves the
  feature but doesn't define the syntax.
- What does the lockfile record, and where does it live in a workspace?

## Plan

### Phase 1: migration

- [x] Migrate all 13 packages to `dpm.yaml` with a workspace manifest.
- [x] Replace sibling paths with `member` dependencies.
- [x] Replace committed third-party DARs with OCI and Git dependencies.
- [ ] Merge `member` resolution into dpm-priv `dario`.
- [ ] Rename the workspace from `token-trading`, a leftover from an example.
- [ ] Remove `daml/dars-to-publish/` and document how the registry was seeded.
- [ ] Record a clean build from a fresh clone with an empty cache.

### Phase 2: workspace catalogs

- [ ] Define and implement package-side catalog references in dpm-priv.
- [ ] Move Splice and DA utility declarations into `workspace.catalogs`.
- [ ] Resolve the `splice-amulet` 0.1.17 and 0.1.19 drift.
- [ ] Remeasure manifest size.

### Phase 3: lockfile

- [ ] Implement a lockfile in dpm-priv that records OCI digests and Git commits.
- [ ] Commit the lockfile and show that a changed artifact fails verification.

Out of scope: moving `releases/` to OCI, which requires DecMan runtime changes,
and re-enabling CI on this fork.
