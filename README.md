# workspace-update-audit-settings

## Probe metadata

| Field              | Value                                 |
|--------------------|---------------------------------------|
| Pattern            | `workspace-update-audit-settings`     |
| PM                 | pnpm                                  |
| pnpm version       | 11.16                                 |
| Feature categories | `manifest_format`, `install_command`  |
| Lockfile format    | v9 (`lockfileVersion: '9.0'`)         |
| Schema version     | 1.2                                   |

## Feature exercised

This probe exercises the **pnpm 11.16 extension of
`pnpm-workspace.yaml`** that adds two new top-level configuration
sections: `update` and `audit`. These sections are workspace-wide
settings keys that must not be mis-classified by Mend's YAML parser
as package reference globs or dependency declarations.

It also touches the `install_command` category because pnpm 11.16
introduces `pnpm update --changeset`, a flag that changes how
dependency-resolution changes are tracked. The probe provides the
dependency structure against which any changeset-driven lock
re-generation can be compared.

## pnpm 11.16 release note detail

pnpm 11.16 added two new top-level keys to `pnpm-workspace.yaml`:

- **`update`** — workspace-wide update behavior settings:
  - `excludeNewer: boolean` — skip packages newer than current
  - `includeDirect: boolean` — include direct deps in update scope

- **`audit`** — workspace-wide audit filter settings:
  - `ignoreCves: string[]` — CVE IDs to suppress from audit output
  - `ignoreGhsas: string[]` — GHSA IDs to suppress from audit output

- **`pnpm update --changeset`** — new flag that generates a
  changeset file alongside the lock update, enabling changelogs
  for monorepo packages affected by the dependency change.

## What Mend must detect

1. **All workspace importers** are detected via the `importers`
   section of `pnpm-lock.yaml`:
   - Root (`.`) — two devDeps: `prettier@3.3.3`, `typescript@5.5.4`
   - `packages/lib` — two deps: `ms@2.1.3`, `semver@7.6.3`
   - `packages/app` — two deps: `@probe/lib` (local/workspace),
     `debug@4.3.6`

2. **Cross-workspace dependency** is correctly typed as `local`:
   `@probe/app` depends on `@probe/lib` via `workspace:*`, which
   resolves to `link:../lib` in the lockfile.

3. **`pnpm-workspace.yaml` new keys** (`update`, `audit`) are
   silently ignored by Mend's parser. They must not cause:
   - A parse error that aborts detection
   - The strings `update` or `audit` to appear as package names
   - The nested key values to be interpreted as version specifiers

4. **Transitive chain**: `debug@4.3.6` → `ms@2.1.3`. Mend must
   report `ms` as a transitive dep of `debug`, not a direct dep
   of root.

## Mend failure modes to watch

- `update` or `audit` YAML keys in `pnpm-workspace.yaml` parsed as
  package glob entries, causing spurious packages to be reported.
- Parser throws on unrecognized top-level keys and aborts detection.
- `@probe/lib` reported as a registry dependency instead of `local`.
- Workspace `packages/lib` and `packages/app` importers not detected
  at all (Mend sees only the root importer).
- `ms` promoted to a direct dependency of root instead of being
  attributed as a transitive dep of `debug`.

## Mend config

**Bucket A** — `js-pnpm` has no dynamic version detection from the
manifest. This probe ships `.whitesource` with
`scanSettings.versioning` pinning:

- `pnpm: "11.16.0"` — exact pnpm version under test
- `node: "20.11.1"` — matching Node.js LTS

`configMode` is `"AUTO"` because no `whitesource.config` is present
in this probe root. No additional dimensions (resolver-driven
detection, branch scoping, or project tokens) are required.

## Resolver note

The Mend UA pnpm resolver (`PnpmLockCollector` / `PnpmParserV9Impl`)
reads the `importers` section for workspace entries and the `packages`
and `snapshots` sections for the full package graph. The resolver is
not documented as reading `pnpm-workspace.yaml` directly — it relies
on `pnpm-lock.yaml` as the source of truth. The risk is that an
upstream YAML parser used to detect workspace roots might fail on the
new top-level keys before even reaching the lockfile. This is an
exploratory probe since the resolver file does not explicitly mention
`pnpm-workspace.yaml` schema validation.

## Expected dependency tree

```
root (workspace-update-audit-settings@0.0.1)
  devDeps:
    prettier@3.3.3
    typescript@5.5.4

packages/lib (@probe/lib@1.0.0)
  deps:
    ms@2.1.3
    semver@7.6.3

packages/app (@probe/app@1.0.0)
  deps:
    @probe/lib@1.0.0 [local -> ../lib]
    debug@4.3.6
      ms@2.1.3  [transitive]
```
