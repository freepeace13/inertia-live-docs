# Release policy

How Inertia Live is versioned, branched and released. This applies to the client packages (`@freepeace13/inertia-live-core`, `-vue`, `-react`) in [`inertia-live`](https://github.com/freepeace13/inertia-live) and the Composer package in [`inertia-live-laravel`](https://github.com/freepeace13/inertia-live-laravel). The release process itself is implemented in [`inertia-live-tooling`](https://github.com/freepeace13/inertia-live-tooling). Why each rule exists is in the [decision log](decision-log.md).

## Versioning

- **Before 1.0:** a minor bump may contain breaking changes; a patch is a fix or docs only (D-001).
- **Client packages are lockstep:** `core`, `vue` and `react` always share one version and one tag, `vX.Y.Z` (D-002).
- **Client and server minors match until 1.0:** npm `0.N.x` pairs with Composer `0.N.x`. A breaking protocol change bumps the minor on both sides in the same release window, and each changelog links to the other (D-003).
- **After 1.0:** standard semver, with an explicit protocol version for compatibility between client and server (see D-012, open).
- **Prereleases** use `-rc.N` only. They do not touch the changelog; the stable release carries the whole entry.

## Branches

- `main` is the development line.
- `release/0.N` (and `release/1.x` after 1.0) maintain an older line. A branch is cut lazily from the line's last tag when the next minor ships (D-004).
- **Fixes land on `main` first**, then are backported by cherry-pick (`-x`) when the PR carries a `backport/<line>` label. A fix never exists only on an old branch.
- Only **patch** releases are allowed on a release branch.

## Support window

Critical fixes only, for the current minor and the one before it. Older lines are archived (D-005). After 1.0: the current major, plus the previous major for critical fixes for six months.

## npm dist-tags

| Release | Tag |
| --- | --- |
| Highest stable version | `latest` |
| Patch on an older line | `release-<line>`, e.g. `release-0.1` |
| Prerelease (`-rc.N`) | `next` |

## Documentation versioning

- Guides and reference live in a folder per major version: `v1/`, `v2/`. The repository root holds only unversioned project docs (this policy, the roadmap, the decision log) and the index (D-013).
- `v1/` covers the 1.x line. Until 1.0 ships it is the working set and describes 0.1.x.
- When development of the next major starts, create `vN+1/` as a copy of `vN/` and edit it there. Fixes to `vN/` follow the same supported-line rules as code (current plus one previous minor, critical only; after 1.0, the previous major for six months).
- Within a major, a page describes the latest minor and marks newer features with "since 1.x".
- Paths at the repository root that predate the folders are one-line stubs to `v1/`, kept so already-published package READMEs keep working. Remove them after the next release.

## Local specs and plans

Generated specs and plans (`docs/` in each repo) are local working files and are gitignored in every repository (D-014). A decision that matters goes in the [decision log](decision-log.md); anything a contributor needs goes in the repo README or this docs repo.

## Release flow

1. Add changelog entries under `## Unreleased` as part of every user-visible change.
2. Dispatch the **prepare release** workflow with a bump (`patch`, `minor`, `major`, `rc` or an explicit version). It bumps the version files, moves `Unreleased` under the new version and opens a release PR.
3. Merge the release PR. The **publish** workflow tags the commit, publishes with the right dist-tag and creates the GitHub Release. It waits for one approval on the `npm-release` GitHub Environment (D-007).
4. Manual `git tag v*`, `git push --tags` and `npm`/`pnpm publish` are blocked by a hook in the Claude plugin and are not part of the process.

For a breaking change: merge the server and client changes, publish an `rc` of both and test against the demo, then publish the client and the server in the same window.

Until the workflows exist (see the [roadmap](roadmap.md), phase 5) releases are still cut by hand: bump the version files, update the changelog, tag `vX.Y.Z`, and the existing `release.yml` publishes the client packages.

## Per-repo configuration

Each repo has a `.release.json`:

```json
{
  "ecosystem": "npm",
  "versionFiles": ["packages/core/package.json", "packages/vue/package.json", "packages/react/package.json"],
  "changelog": "CHANGELOG.md",
  "tagPrefix": "v"
}
```

For `composer`, `versionFiles` is empty; Composer reads the version from the tag. The release script's rules are documented in the tooling repo's README.
