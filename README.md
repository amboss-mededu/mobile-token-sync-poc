# Mobile Design System Sync POC

Proof of concept for automatically syncing design tokens and icons from
[amboss-design-system](https://github.com/amboss-mededu/amboss-design-system)
to mobile repositories via GitHub Actions.

Two independent flows live side by side:

| Asset | Flow | DS dispatcher | Mobile receiver |
|-------|------|---------------|-----------------|
| **Tokens** | Push-based, sparse checkout | `trigger-token-sync-mobile.yml` (fires on `push: main paths:`) | `design-token-sync.yml` |
| **Icons** | Release-based, tarball download | `dispatch-mobile-on-release.yml` (fires on `release: published`) | `design-icons-sync.yml` |

Tokens are small text files reviewable in mobile-side PRs; icons are large
batches of generated PDFs/XMLs better shipped as immutable release artifacts.

## Tokens flow

```
amboss-design-system                     mobile repo (this POC)
┌─────────────────────┐                  ┌─────────────────────────┐
│ PR merged to main   │                  │  repository_dispatch    │
│ with version bump   │                  │  (design-tokens-update) │
│ in tokens-ios       │ ──dispatch────▶  │  1. checkout DS at SHA  │
│ package.json        │                  │  2. copy token files    │
│                     │                  │  3. open PR             │
└─────────────────────┘                  └─────────────────────────┘
```

### Dispatcher

[`trigger-token-sync-mobile.yml`](https://github.com/amboss-mededu/amboss-design-system/blob/main/.github/workflows/trigger-token-sync-mobile.yml)

- Triggers on push to `main` when `packages/tokens-ios/package.json` or
  `packages/tokens-android/package.json` changes
- Compares the `version` field before/after the merge
- Dispatches a `design-tokens-update` event with payload:
  ```json
  { "version": "0.2.0", "previous_version": "0.1.0", "sha": "abc123..." }
  ```

### Receiver

`.github/workflows/design-token-sync.yml`

- Triggers on `repository_dispatch` event type `design-tokens-update`
- Sparse-checkouts `amboss-design-system` at the exact SHA from the payload
- Copies token files to the target paths
- Creates a PR via `peter-evans/create-pull-request`

## Icons flow

```
amboss-design-system                          mobile repo (this POC)
┌────────────────────────┐                    ┌──────────────────────────────┐
│ PR merged with release │                    │  repository_dispatch         │
│ label → CircleCI bumps │                    │  (design-icons-ios-update)   │
│ DS version, builds     │ ──dispatch event──▶│  1. download tarball         │
│ icons tarballs,        │                    │  2. wipe icons/ios           │
│ uploads them as GH     │                    │  3. unpack into icons/ios    │
│ release assets         │                    │  4. open PR                  │
└────────────────────────┘                    └──────────────────────────────┘
```

### Dispatcher

[`dispatch-mobile-on-release.yml`](https://github.com/amboss-mededu/amboss-design-system/blob/main/.github/workflows/dispatch-mobile-on-release.yml)

- Triggers on `release: published` (created by CircleCI `release-and-publish`
  via `auto release`)
- For each icons sub-package (`icons-ios`, `icons-android`), looks up the
  matching `<package>-<version>.tgz` asset on the release
- Dispatches a `design-icons-<platform>-update` event with payload:
  ```json
  {
    "package": "icons-ios",
    "version": "0.2.0",
    "asset_name": "icons-ios-0.2.0.tgz",
    "asset_url": "https://github.com/.../releases/download/v3.43.3/icons-ios-0.2.0.tgz",
    "ds_tag": "v3.43.3",
    "ds_release_url": "https://github.com/.../releases/tag/v3.43.3"
  }
  ```

### Receiver

`.github/workflows/design-icons-sync.yml`

- Triggers on `repository_dispatch` event type `design-icons-ios-update`
- Downloads the tarball asset directly from the GH release using a PAT
- Wipes `icons/ios/` and unpacks the tarball into it (so removed or renamed
  icons don't leave orphans)
- Opens a PR via `peter-evans/create-pull-request`

## Testing

### Prerequisites

- A GitHub PAT with `repo` scope, stored as `MOBILE_REPOS_ACCESS_TOKEN` secret
  in **both** repos

### Tokens — manual trigger

```bash
gh api repos/amboss-mededu/mobile-token-sync-poc/dispatches \
  -f event_type=design-tokens-update \
  -f 'client_payload[version]=0.2.0' \
  -f 'client_payload[previous_version]=0.1.0' \
  -f 'client_payload[sha]=<commit-sha-from-design-system>'
```

### Icons — manual trigger

Use a real release tag and asset URL from the design-system repo.

```bash
gh api repos/amboss-mededu/mobile-token-sync-poc/dispatches \
  -f event_type=design-icons-ios-update \
  -f 'client_payload[package]=icons-ios' \
  -f 'client_payload[version]=0.2.0' \
  -f 'client_payload[asset_name]=icons-ios-0.2.0.tgz' \
  -f 'client_payload[asset_url]=https://github.com/amboss-mededu/amboss-design-system/releases/download/v3.43.3/icons-ios-0.2.0.tgz' \
  -f 'client_payload[ds_tag]=v3.43.3' \
  -f 'client_payload[ds_release_url]=https://github.com/amboss-mededu/amboss-design-system/releases/tag/v3.43.3'
```

## Adapting for your repo

Copy the workflow that matches the asset you consume:

- Tokens-only: `design-token-sync.yml`. Edit the **file mapping** section to
  match your repo's directory layout.
- Icons-only: `design-icons-sync.yml`. Edit the unpack target (`icons/ios`)
  to match where icons live in your project.
- Both: copy both workflows.

### iOS target paths (example)

| Asset | Source | Target |
|-------|--------|--------|
| Tokens | `packages/tokens-ios/Sources/AmbossDesignTokens/*.swift` | `DesignSystem/Resources/Tokens/` |
| Icons | tarball contents (`*.imageset/*.pdf`, `*.imageset/Contents.json`) | `DesignSystem/Resources/Assets.xcassets/AmbossIcons/` |

### Android target paths (example)

| Asset | Source | Target |
|-------|--------|--------|
| Tokens | `packages/tokens-android/{xml,compose}/*` | `shared-ui/src/main/{res/values,java/.../compose}/` |
| Icons | tarball contents (`ic_*.xml`) | `app/src/main/res/drawable/` |
