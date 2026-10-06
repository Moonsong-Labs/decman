# Case study: building Decentralization Manager with `dpm.yaml`

We migrated the Daml build of Decentralization Manager, a production Canton
application with 13 packages, from `daml.yaml` to `dpm.yaml` using dpm-priv,
our fork of `dpm`. The migration removes all 17 committed third-party DARs, 24
versioned sibling paths, and 12 duplicate SDK declarations. Version mismatches,
longer manifests, and the loss of byte-identical inputs remain open until
workspace catalogs and a lockfile land.

## The problem

The
[Canton Network Developer Experience and Tooling Survey (2026)](https://forum.canton.network/t/canton-network-developer-experience-and-tooling-survey-analysis-2026/8412)
describes Daml dependency management as "a manual, file-based process."
Developers download DAR files by hand, move them between folders, and
struggle to resolve version mismatches.

[Decentralization Manager](https://github.com/DLC-link/decentralization-manager)
(DecMan), a production Canton application, shows all three symptoms. This case
study migrates DecMan's Daml build to `dpm.yaml` and measures what changes.

## The problem in DecMan

The baseline is upstream `main` at commit
[`b507e55`](https://github.com/DLC-link/decentralization-manager/commit/b507e5519c5d4bd8b8f5d0c6fe7c6548814fbf9f)
(DecMan v1.9.0). DecMan has 13 Daml packages that depend on each other and on
17 third-party DARs, most of them from Splice and the Digital Asset (DA)
utility packages.

- **DARs moved by hand between folders.** All 17 third-party DARs are
  committed under `daml/dars/`, and the manifests reference them 68 times by
  relative path. A DAR's origin and version exist only in its filename.
- **Version mismatches.** 24 references to sibling packages hardcode a build
  path that includes the sibling's version, so a version bump means editing
  every dependent. Nothing checks consistency: `governance-token-custody-test`
  uses `splice-amulet` 0.1.17, and every other package uses 0.1.19.
- **No standard workflow.** The build is a `multi-package.yaml` plus 13
  `daml.yaml` files, each with its own copy of `sdk-version: 3.4.11`.

## What we built

dpm-priv is a Moonsong Labs fork of `dpm` that adds a `dpm dario` command
set. It reads a declarative manifest, `dpm.yaml`, resolves each dependency
from where it lives, generates the `daml.yaml` that `damlc` expects, and calls
the standard `damlc`. The schema is specified in
`pkg/dario/dpm-yaml-schema.md` in dpm-priv.

| Capability | Status in dpm-priv |
|---|---|
| `dpm.yaml` manifests for packages and workspaces | Done |
| Workspaces: member globs, shared SDK version, and registries | Done |
| Dependencies from OCI registries, by semantic-versioning requirement | Done |
| Dependencies from Git, by commit, branch, tag, or release artifact | Done |
| Shared global cache for fetched artifacts | Done |
| `member` dependencies between workspace packages | On branch `feat/member-dep-support`, not merged |
| Workspace catalogs | Parsed and validated, not usable from packages |
| Lockfile with artifact checksums | Not started |

## Before and after

### DARs moved by hand between folders

**Before.** Each package lists vendored DARs by path. This excerpt is from
`governance-utility-onboarding/daml.yaml`:

```yaml
data-dependencies:
  - ../dars/splice-amulet-0.1.19.dar
  - ../dars/splice-api-token-holding-v1-1.0.0.dar
  - ../dars/utility-registry-v0-0.6.0.dar
  # ... 10 more
```

**After.** Each dependency names its source. Splice DARs come from the Splice
Git repository at a pinned commit, and DA utility DARs come from an OCI
registry. `dpm` fetches them into its cache:

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

**Result:** no third-party DARs in the repository, and every dependency's
origin is in its manifest.

### Version mismatches

**Before.** A package depends on its sibling's build output, with the
sibling's version in the path:

```yaml
data-dependencies:
  - ../governance-action-v1/.daml/dist/governance-action-v1-0.1.0.dar
```

**After.** A package names its sibling:

```yaml
data-dependencies:
  governance-action-v1: member
```

`dpm` builds the sibling first, and the dependent compiles against the
sibling's current sources. Packages keep independent versions, because Daml
upgrade compatibility is per package, but a version bump no longer requires
edits in dependents.

**Result:** the 24 versioned sibling paths are gone. The `splice-amulet`
0.1.17 mismatch is not: it carried over unchanged, because each package still
declares its own copy of each third-party dependency. Workspace catalogs fix
this; see [What's left](#whats-left).

### No standard workflow

**Before.** `multi-package.yaml` lists the packages, and each of the 13
`daml.yaml` files repeats the SDK version.

**After.** One workspace manifest, [`daml/dpm.yaml`](daml/dpm.yaml), declares
the members, the SDK version, and the registries:

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

Each package sets `sdk: workspace`. To build everything, run `dpm dario build`
from `daml/` with a dpm-priv build that includes `member` support.

**Result:** one SDK declaration instead of 13, and one command to build the
workspace.

## The trade-off we haven't closed

Committing DARs had one real benefit: every checkout built against the same
bytes. Fetching from a registry gives that up. Git dependencies pin a commit,
but OCI dependencies resolve to a tag, and a registry can move a tag to
different content. The schema lets you pin an OCI DAR with `digest`, but this
repository doesn't, because maintaining digests by hand across 13 packages is
error-prone.

A lockfile restores the guarantee. It records the checksum of every resolved
artifact and fails the build when a fetched artifact doesn't match. dpm-priv
doesn't have one yet, and it's the next thing to build.

## Scorecard

| Metric | Before (`daml.yaml`) | After (`dpm.yaml`) |
|---|---|---|
| Third-party DARs committed to the repository | 17 | 0 |
| Relative-path references to those DARs | 68 | 0 |
| Sibling references that hardcode a build path and version | 24 | 0 |
| Copies of the SDK version | 13 | 1 |
| Known version mismatches | 1 | 1 |
| Non-blank manifest lines | 277 | 484 |

Line counts cover every `daml.yaml` plus `multi-package.yaml` before, and
every `dpm.yaml` after.

## What's left

### Longer manifests and surviving mismatches

Manifests grew from 277 to 484 non-blank lines, because each package declares
its Git and OCI dependencies in full. The same Splice commit appears 40 times.
Workspace catalogs let a workspace declare each third-party dependency once,
which also removes the place where mismatches hide:

```yaml
workspace:
  catalogs:
    splice:
      splice-amulet:
        git: https://github.com/canton-network/splice
        commit: d5b780b5b636fe46f6f77e87c7655ddaf53c50fc
        path: daml/dars/splice-amulet-0.1.19.dar
```

dpm-priv parses and validates `catalogs`, but packages can't reference them
yet.

### DARs that stay in the repository

`releases/` holds first-party DARs that the DecMan server uploads to Canton at
runtime. They aren't build inputs, so this migration leaves them in place.
`daml/dars-to-publish/` holds the DARs used to seed the OCI registry.

### Questions for `dpm` maintainers

- How should a package reference a catalog entry? One option is a `catalog:`
  key on the dependency that names the catalog.
- What should the lockfile record, and where should it live in a workspace?

### Next steps

Phase 1: migration

- [x] Migrate all 13 packages to `dpm.yaml` with a workspace manifest.
- [x] Replace sibling paths with `member` dependencies.
- [x] Replace committed third-party DARs with OCI and Git dependencies.
- [ ] Merge `member` resolution into dpm-priv `dario`.
- [ ] Rename the workspace from `token-trading`, a leftover from an example.
- [ ] Remove `daml/dars-to-publish/` and document how the registry was seeded.
- [ ] Record a clean build from a fresh clone with an empty cache.

Phase 2: workspace catalogs

- [ ] Implement package-side catalog references in dpm-priv.
- [ ] Move Splice and DA utility declarations into `workspace.catalogs`.
- [ ] Resolve the `splice-amulet` 0.1.17 and 0.1.19 mismatch.
- [ ] Remeasure manifest size.

Phase 3: lockfile

- [ ] Implement a lockfile in dpm-priv that records artifact checksums.
- [ ] Commit the lockfile and show that a changed artifact fails the build.

Out of scope: moving `releases/` to OCI, which requires DecMan runtime
changes, and re-enabling CI on this fork.
