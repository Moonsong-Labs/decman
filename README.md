# dpm-priv showcase: migrating Decentralization Manager to `dpm.yaml`

> Looking for the Decentralization Manager application itself? See
> [READMEdecman.md](READMEdecman.md).

This fork of [Decentralization Manager](https://github.com/DLC-link/decentralization-manager)
(DecMan) shows the proposed `dpm` package-manager design on a real production
codebase rather than a toy example. DecMan's Daml code builds here with
**dpm-priv**, our fork of `dpm` that implements the baseline of the proposed
design.

DecMan is a good stress test: **13 Daml packages in one workspace**, which
upstream builds with **17 third-party `.dar` files committed to the repo**.

## What we want to show

- A new developer experience through the proposed `dpm.yaml` manifest format.
- A cleaner repo, with no committed `.dar` build inputs and no hardcoded
  dependency paths.
- Less manual work:
  - Versions synchronized automatically across packages in the same workspace.
  - Dependency declarations unified through workspace catalogs.
  - Direct references to OCI and Git sources.
- Verifiable, reproducible builds backed by a lockfile.

## Status

| Capability | dpm-priv | Used in this repo |
|---|---|---|
| `dpm.yaml` package and workspace manifests | ✅ | ✅ |
| Workspace members (`members: pkg/*`) and `sdk: workspace` | ✅ | ✅ |
| `member` dependencies between sibling packages | ✅ | ✅ |
| OCI dependencies | ✅ | ✅ |
| Git dependencies (commit-pinned) | ✅ | ✅ |
| Workspace catalogs | 🚧 parsed and validated; package references not implemented | ❌ |
| Lockfile | ❌ | ❌ |

This README is updated as each phase of the [plan](#plan) lands.

## The migration, step by step

The "before" is upstream `main` at
[`b507e55`](https://github.com/DLC-link/decentralization-manager/commit/b507e5519c5d4bd8b8f5d0c6fe7c6548814fbf9f)
(DecMan v1.9.0). The "after" is this branch.

### Where we started

Upstream builds with a `multi-package.yaml` and 13 independent `daml.yaml`
files. Here is `governance-utility-onboarding/daml.yaml`:

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

Across the workspace this pattern adds up to:

- 17 third-party DARs committed under `daml/dars/`, referenced 68 times by
  relative path.
- 24 sibling references that hardcode both a build-output path and the
  sibling's version (`.../governance-action-v1-0.1.0.dar`). Bumping a package
  means editing every package that depends on it.
- `sdk-version: 3.4.11` repeated in 13 files.
- Silent version drift: `governance-token-custody-test` depends on
  `splice-amulet-0.1.17.dar`, while every other package uses `0.1.19`. Nothing
  in this layout flags it.

### Step 1: one workspace

A single workspace manifest at [`daml/dpm.yaml`](daml/dpm.yaml) replaces
`multi-package.yaml`. It owns the member list, the SDK version, and the
registries:

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

Each package declares `sdk: workspace`, so the SDK version lives in one place.

### Step 2: siblings by name

```yaml
data-dependencies:
  governance-action-v1: member
```

replaces `../governance-action-v1/.daml/dist/governance-action-v1-0.1.0.dar`.
The dependency resolves to whatever version the sibling currently declares.
Packages still version independently, because Daml upgrade compatibility is
per package, but bumping one no longer requires edits anywhere else. This is
the "versions synchronized automatically" story.

### Step 3: DARs leave the repo

Third-party DARs now come straight from their source. Splice artifacts come
from Git, pinned to a commit. DA utility packages come from an OCI registry:

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

`daml/dars/` is gone. The OCI registry is anonymously readable, so anyone can
pull the same artifacts.

> **Scope of "no committed DARs".** `releases/` still holds first-party DARs.
> These are runtime assets that the DecMan server uploads to Canton, not build
> inputs, so they are out of scope for this migration.
> `daml/dars-to-publish/` holds the DARs used to seed the OCI registry and will
> be removed.

### Where it's still rough

The migration is honest about one regression: manifests grew from **277 to
484** non-blank lines, because each package spells out its Git and OCI
dependencies in full. The same Splice commit is pinned **40 times**.

Workspace catalogs fix this. The target shape declares each third-party
dependency once:

```yaml
workspace:
  catalogs:
    splice:
      splice-amulet:
        git: https://github.com/canton-network/splice
        commit: d5b780b5b636fe46f6f77e87c7655ddaf53c50fc
        path: daml/dars/splice-amulet-0.1.19.dar
      # ...
```

Packages then reference the catalog entry by name instead of repeating it,
and drift like `splice-amulet` 0.1.17 vs 0.1.19 becomes visible in one place.
Catalogs are parsed and validated by dpm-priv today. Package-side references
are Phase 2.

### Watch it build

<!-- TODO: embed recording of a clean build from a fresh clone with an empty dpm cache. -->
_Recording coming soon:_ a fresh clone with an empty dpm cache, running
`dpm build` from `daml/`, pulling from OCI and Git and compiling all 13
packages.

## By the numbers

| | Before (upstream `main`) | After (this branch) |
|---|---|---|
| Third-party DARs committed (`daml/dars/`) | 17 | 0 |
| Relative-path references to those DARs | 68 | 0 |
| Hardcoded sibling `.daml/dist` paths | 24 | 0 |
| Copies of the SDK version | 13 | 1 |
| Non-blank manifest lines | 277 | 484 (Phase 2 target: below 277) |

## Plan

### Phase 1: showcase what works (this README)

- [x] Migrate all 13 packages to `dpm.yaml` with a workspace root.
- [x] Replace sibling paths with `member` dependencies.
- [x] Replace committed third-party DARs with OCI and Git dependencies.
- [ ] Remove `daml/dars-to-publish/`, documenting how the registry was seeded.
- [ ] Record a clean build and embed it above.

### Phase 2: workspace catalogs

- [ ] dpm-priv: package-side catalog references, resolution, and publish-time
      expansion into standalone declarations.
- [ ] Move Splice Git pins and utility OCI versions into `workspace.catalogs`.
- [ ] Resolve the `splice-amulet` 0.1.17/0.1.19 drift explicitly.
- [ ] Update this README: re-measure, replace "Where it's still rough", re-record.

### Phase 3: lockfile

- [ ] dpm-priv: lockfile recording resolved OCI digests and Git commits.
- [ ] Commit the lockfile here. Show that a changed artifact fails verification
      and that two clean builds produce identical DAR hashes.
- [ ] Update this README with a "Verifiable builds" step and re-record.

### Out of scope for now

- Moving `releases/` DARs to OCI (requires DecMan runtime changes).
- Re-enabling CI on this fork against dpm-priv.
