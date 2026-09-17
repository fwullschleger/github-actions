# github-actions

Shared GitHub Actions for my Grav plugins.

## TL;DR

- **One manual release workflow** for Grav plugins: pick a version, the pipeline
  refuses anything unsafe, lints the PHP, tags, and publishes the GitHub release.
- **`release-guard`** is the gatekeeper: branch, SemVer format, duplicate tag,
  version order, linear history, `blueprints.yaml`, and the `CHANGELOG.md` entry.
- **Pin `@v1`.** It is a moving tag, so fixes reach callers without editing each one.
- This repository is **public**, because a private one cannot be called by public
  repositories at all.

## Use it

Add `.github/workflows/create-release.yml` to your plugin:

```yaml
name: Create Release

on:
  workflow_dispatch:
    inputs:
      version:             { description: 'SemVer (e.g. v1.6.3)', required: true, type: string }
      prerelease:          { type: boolean, default: false }
      draft:               { type: boolean, default: false }
      release_notes:       { type: string, required: false }
      allow_older_version: { description: 'Bypass version order (hotfix)', type: boolean, default: false }

jobs:
  release:
    uses: fwullschleger/github-actions/.github/workflows/grav-plugin-release.yml@v1
    permissions:
      contents: write
    with:
      version:             ${{ inputs.version }}
      prerelease:          ${{ inputs.prerelease }}
      draft:               ${{ inputs.draft }}
      release_notes:       ${{ inputs.release_notes }}
      allow_older_version: ${{ inputs.allow_older_version }}
```

Then run it from **Actions → Create Release → Run workflow**.

## What the guard checks

| # | Check | Fails when |
|---|---|---|
| 0 | Full history | the checkout is shallow (`fetch-depth: 0` missing) |
| 1 | Branch | you release from anything but the default branch |
| 2 | Format | the version is not `vX.Y.Z` or `vX.Y.Z-suffix` |
| 3 | Duplicate | the tag already exists locally or on the remote |
| 4 | Order and history | the version is not greater than the latest tag, or `HEAD` does not descend from it |
| 5 | Plugin version | `version:` in `blueprints.yaml` differs from the requested version |
| — | Changelog | `CHANGELOG.md` has no `# vX.Y.Z` section for the release |

Check 4 is bypassed by `allow_older_version`, for a hotfix on an older line.

A Grav plugin has no build, so the workflow runs `php -l` over every PHP file
instead. The changelog section for the released version becomes the release body.

## Inputs worth knowing

| Input | Default | Notes |
|---|---|---|
| `version_file` | `blueprints.yaml` | read as YAML |
| `changelog_file` | `CHANGELOG.md` | set to `''` to skip the check |
| `changelog_heading` | `# v{version_no_v}` | the Grav convention; supports `{version}` too |

## Maintaining `release-guard`

`actions/release-guard/action.yml` exists as **two independent copies**, here and
in `sanetics/github-actions` (private, for .NET projects). They are byte-identical
by intent and the two repositories never reference each other, so neither account
depends on the other. A change to one must be applied to the other.

## Releasing this repository

Callers pin `@v1`, a moving tag. After a backwards-compatible change, retarget it
at the new commit:

```bash
git tag --force v1
```

Publishing the moved tag overwrites a remote ref, so it is a human step — the
workspace push guard blocks agents from doing it.
