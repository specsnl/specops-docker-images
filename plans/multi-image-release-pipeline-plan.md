# Multi-Image Release Pipeline — Implementation Plan

## Goal

A single repository contains multiple Docker images that must be **released
independently**. Each release is triggered manually, picks one image, and bumps
that image's version (major / minor / patch / prerelease). The computed version
is baked into the image, pushed to GHCR as a multi-arch manifest, and recorded
back in git.

## Context and conventions

- **`ghcr.io/specsnl/utils:latest`** is a public image containing `svu`, `jq`,
  `git`, and `bash`. No credentials block is needed to pull it.
- **`specsnl/github-actions@2.1.0`** is the pinned version of the reusable
  workflows. It provides `build.yml` (native per-arch build, push by digest) and
  `merge.yml` (multi-arch manifest assembly and tagging). Both are already
  extended with the `context`, `raw-tags`, and `version` inputs this workflow
  relies on.
- **Git tags are the single source of truth for versions.** Nothing queries the
  registry. Tags are namespaced per image: `<image>/v<semver>` (e.g.
  `image-a/v1.3.0`, `image-b/v2.0.1-rc.2`). Each image has its own independent
  version line within one repo.
- **Build is native per-arch** (no QEMU): `ubuntu-24.04` for `linux/amd64`,
  `ubuntu-24.04-arm` for `linux/arm64`.
- **Four jobs in order:** `version` → `build` → `merge` → `tag`. The git tag is
  pushed last, only after the image is successfully in GHCR, so a failed build
  never leaves a phantom version claimed in git.

---

## Workflow inputs

```yaml
on:
  workflow_dispatch:
    inputs:
      image:
        type: choice
        options: [image-a, image-b, image-c]   # keep in sync with the repo
      release_type:
        type: choice
        options: [patch, minor, major, prerelease]
        default: patch
      prerelease_id:
        description: Set to START a prerelease train (e.g. rc.1, beta.1). Empty for stable.
        type: string
        required: false
      version:
        description: Explicit version, overrides everything else (e.g. 1.4.0 or 1.4.0-rc.1)
        type: string
        required: false
```

### Input → behaviour

| Current state | `release_type` | `prerelease_id` | Result |
|---|---|---|---|
| `1.2.3` | `patch` | _(empty)_ | `1.2.4` |
| `1.2.3` | `minor` | _(empty)_ | `1.3.0` |
| `1.2.3` | `major` | _(empty)_ | `2.0.0` |
| `1.2.3` | `minor` | `rc.1` | `1.3.0-rc.1` (starts a train) |
| `1.3.0-rc.1` | `prerelease` | _(empty)_ | `1.3.0-rc.2` (auto-increments) |
| `1.3.0-rc.2` | `patch` | _(empty)_ | `1.3.0` (finalizes; drops the prerelease) |

svu behaviours relied on:

- `svu prerelease` auto-increments the trailing counter when the current tag is
  already a prerelease — no identifier needed to continue a train.
- Starting a train from a stable tag requires a bump command plus an identifier:
  `svu minor --prerelease rc.1`.
- `svu patch` on a prerelease tag clears the prerelease without incrementing —
  this is promotion to stable.

`prerelease_id` is what disambiguates a stable bump from a prerelease bump.
The `version` field is the escape hatch for anything the bump logic doesn't
model; when set it is used verbatim.

---

## Version resolution script

Used in the `version` job. svu's `--json` output shape:

```go
type VersionInfo struct {
    Version    string `json:"version"`
    Major      uint64 `json:"major"`
    Minor      uint64 `json:"minor"`
    Patch      uint64 `json:"patch"`
    Prefix     string `json:"prefix,omitempty"`
    Metadata   string `json:"metadata,omitempty"`
    Prerelease string `json:"prerelease,omitempty"`  // "rc", "beta", ...
    Build      string `json:"build,omitempty"`       // "1", "2", ... (counter)
}
```

Semver is reconstructed from the numeric fields (prefix-agnostic and exact):

```bash
PREFIX="${IMAGE}/v"

if [ -n "${EXPLICIT_VERSION}" ]; then
  SEMVER="${EXPLICIT_VERSION}"
  core="${SEMVER%%+*}"
  case "$core" in
    *-*) IS_PRERELEASE=true; pre="${core#*-}"; CHANNEL="${pre%%.*}" ;;
    *)   IS_PRERELEASE=false; CHANNEL="" ;;
  esac
else
  case "${RELEASE_TYPE}" in
    prerelease) SUBCMD="prerelease" ;;
    *)          SUBCMD="${RELEASE_TYPE}" ;;
  esac

  JSON=$(svu "${SUBCMD}" ${PRERELEASE_ID:+--prerelease "${PRERELEASE_ID}"} \
              --tag.prefix="${PREFIX}" --json)

  SEMVER=$(printf '%s' "$JSON" | jq -r '
    "\(.major).\(.minor).\(.patch)"
    + (if (.prerelease // "") != "" then "-\(.prerelease)"
         + (if (.build // "") != "" then ".\(.build)" else "" end)
       else "" end)
    + (if (.metadata // "") != "" then "+\(.metadata)" else "" end)
  ')
  IS_PRERELEASE=$(printf '%s' "$JSON" | jq -r 'if (.prerelease // "") == "" then "false" else "true" end')
  CHANNEL=$(printf '%s' "$JSON" | jq -r '.prerelease // ""')
fi

{
  echo "semver=${SEMVER}"
  echo "git_tag=${PREFIX}${SEMVER}"
  echo "is_prerelease=${IS_PRERELEASE}"
  echo "channel=${CHANNEL}"
} >> "$GITHUB_OUTPUT"
```

Notes for the implementer:

- `>> "$GITHUB_OUTPUT"` works normally inside a container job.
- The container default shell is `sh`; `bash` is set explicitly at the job level
  via `defaults.run.shell`.
- `${PRERELEASE_ID:+--prerelease "${PRERELEASE_ID}"}` is POSIX conditional
  expansion: the flag is only included when `PRERELEASE_ID` is non-empty.

---

## Tagging rules (applied by `merge.yml` via `create-manifest`)

- **`semver`** — always; passed as `raw-tag`.
- **`latest`** — only when `is_prerelease == 'false'`; passed as a `raw-tags`
  line with `enable=`.
- **`channel`** (`rc`, `beta`, …) — only when `is_prerelease == 'true'`; passed
  as a `raw-tags` line with `enable=`. Lets consumers opt into a prerelease
  track without ever pulling `latest`.
- **OCI `org.opencontainers.image.version`** label and annotation — passed via
  the `version` input.

---

## Full workflow

```yaml
name: Release image

on:
  workflow_dispatch:
    inputs:
      image:
        type: choice
        options: [image-a, image-b, image-c]   # keep in sync with the repo
      release_type:
        type: choice
        options: [patch, minor, major, prerelease]
        default: patch
      prerelease_id:
        description: Set to START a prerelease train (e.g. rc.1, beta.1). Empty for stable.
        type: string
        required: false
      version:
        description: Explicit version, overrides everything else (e.g. 1.4.0 or 1.4.0-rc.1)
        type: string
        required: false

permissions:
  packages: write
  contents: write
  actions: write

# Prevent two concurrent releases of the same image from computing the same
# next version and colliding on the git tag push.
concurrency:
  group: release-${{ inputs.image }}
  cancel-in-progress: false

jobs:
  version:
    runs-on: ubuntu-24.04
    container:
      image: ghcr.io/specsnl/utils:latest   # public; no credentials needed
    defaults:
      run:
        shell: bash                          # container default is sh
    outputs:
      semver:        ${{ steps.v.outputs.semver }}
      git_tag:       ${{ steps.v.outputs.git_tag }}
      is_prerelease: ${{ steps.v.outputs.is_prerelease }}
      channel:       ${{ steps.v.outputs.channel }}
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0     # svu needs full history to walk tags
          fetch-tags: true
      - id: v
        env:
          IMAGE:            ${{ inputs.image }}
          RELEASE_TYPE:     ${{ inputs.release_type }}
          PRERELEASE_ID:    ${{ inputs.prerelease_id }}
          EXPLICIT_VERSION: ${{ inputs.version }}
        run: |
          # <-- version resolution script from above -->

  build:
    needs: version
    uses: specsnl/github-actions/.github/workflows/build.yml@2.1.0
    strategy:
      fail-fast: false
      matrix:
        runs-on:
          - { os: ubuntu-24.04,     platform: linux/amd64 }
          - { os: ubuntu-24.04-arm, platform: linux/arm64 }
    with:
      runs-on:    ${{ matrix.runs-on.os }}
      platform:   ${{ matrix.runs-on.platform }}
      image-name: ghcr.io/${{ github.repository }}/${{ inputs.image }}
      dockerfile: images/${{ inputs.image }}/Dockerfile
      context:    images/${{ inputs.image }}

  merge:
    needs: [version, build]
    uses: specsnl/github-actions/.github/workflows/merge.yml@2.1.0
    with:
      runs-on:    ubuntu-24.04
      image-name: ghcr.io/${{ github.repository }}/${{ inputs.image }}
      raw-tag:    ${{ needs.version.outputs.semver }}
      raw-tags: |
        type=raw,value=latest,enable=${{ needs.version.outputs.is_prerelease == 'false' }}
        type=raw,value=${{ needs.version.outputs.channel }},enable=${{ needs.version.outputs.is_prerelease == 'true' }}
      version:    ${{ needs.version.outputs.semver }}

  # Tag git LAST — only after a successful build and merge. A failed build
  # never leaves a version claimed in git. No force-push: if the tag already
  # exists the push fails loudly, which is the duplicate-release guard.
  tag:
    needs: [version, merge]
    if: success()
    runs-on: ubuntu-24.04
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v6
      - name: Push git tag
        run: |
          git tag -a "${{ needs.version.outputs.git_tag }}" \
            -m "Release ${{ inputs.image }} ${{ needs.version.outputs.semver }}"
          git push origin "${{ needs.version.outputs.git_tag }}"
```

---

## Implementation checklist

- [ ] Confirm per-image directory layout. The workflow assumes each image lives
      under `images/<image>/` with a `Dockerfile` at that path. Adjust
      `dockerfile` and `context` in the `build` job if the layout differs.
- [ ] Update the `image` choice list to match the actual image names in the repo.
- [ ] Add the workflow file at `.github/workflows/release.yml` (or similar).
- [ ] Seed an initial namespaced git tag for each image if none exist yet
      (e.g. `git tag image-a/v0.0.0 && git push origin image-a/v0.0.0`).
      Without a baseline tag svu has nothing to increment from.
- [ ] Verify the GHCR image path (`ghcr.io/<owner>/<repo>/<image>`) matches
      the intended pull paths.
- [ ] Test each release path in order:
      - `patch` (stable bump)
      - `minor` (stable bump)
      - `major` (stable bump)
      - `minor` + `prerelease_id: rc.1` (start a prerelease train)
      - `prerelease` with empty `prerelease_id` (iterate the train)
      - `patch` with empty `prerelease_id` (finalize to stable)
      - explicit `version` override
