# Mobile Design System Sync POC

Proof of concept for automatically syncing design tokens and icons from
[amboss-design-system](https://github.com/amboss-mededu/amboss-design-system)
to mobile repositories via GitHub Actions and tarball release assets.

## How It Works

```
amboss-design-system                          mobile repo (this POC)
┌────────────────────────┐                    ┌──────────────────────────────┐
│ PR merged with release │                    │                              │
│ label → CircleCI bumps │                    │   repository_dispatch        │
│ DS version, builds 4   │                    │   (design-<package>-update)  │
│ tarballs, uploads them │ ──dispatch event──▶│                              │
│ as GH release assets   │                    │   1. download tarball        │
│ (auto release)         │                    │   2. wipe target dir         │
│                        │                    │   3. unpack into target      │
│ release: published →   │                    │   4. open PR                 │
│ dispatch-mobile-on-    │                    │                              │
│ release.yml            │                    │                              │
└────────────────────────┘                    └──────────────────────────────┘
```

### Dispatcher (design-system side)

Workflow: [`.github/workflows/dispatch-mobile-on-release.yml`](https://github.com/amboss-mededu/amboss-design-system/blob/main/.github/workflows/dispatch-mobile-on-release.yml)

- Triggers on `release: published` (created by CircleCI `release-and-publish`
  via `auto release`)
- For each sub-package matrix entry (`tokens-ios`, `tokens-android`,
  `icons-ios`, `icons-android`), looks up the matching `<package>-<version>.tgz`
  asset on the release
- Dispatches a `design-<package>-update` event with payload:
  ```json
  {
    "package": "tokens-ios",
    "version": "0.2.0",
    "asset_name": "tokens-ios-0.2.0.tgz",
    "asset_url": "https://github.com/.../releases/download/v3.43.3/tokens-ios-0.2.0.tgz",
    "ds_tag": "v3.43.3",
    "ds_release_url": "https://github.com/.../releases/tag/v3.43.3"
  }
  ```

### Receiver (mobile repo side)

Workflow: `.github/workflows/design-system-sync.yml`

- Triggers on `repository_dispatch` events of type `design-tokens-ios-update`
  or `design-icons-ios-update` (one workflow handles all sub-packages this
  repo consumes)
- Resolves the target directory based on the `package` field in the payload
- Downloads the tarball asset directly from the GH release using a PAT
- Wipes the target directory and unpacks the tarball into it (so removed or
  renamed files don't leave orphans)
- Opens a PR via `peter-evans/create-pull-request`

## Testing the POC

### Prerequisites

- A GitHub PAT with `repo` scope, stored as `MOBILE_REPOS_ACCESS_TOKEN` secret
  in **both** repos (the dispatcher needs it to send events to private mobile
  repos; the receiver needs it to download release assets from the private
  design-system repo)

### Steps

1. In `amboss-design-system`, open a PR that:
   - Touches `assets/icons/*.svg` (auto-bumps `packages/icons-*/package.json`)
     and/or `packages/design-tokens/src/**` (auto-bumps `packages/tokens-*/package.json`)
   - Has a release label (`patch` / `minor` / `major`)
2. Merge the PR
3. CircleCI `release-and-publish` runs, creates the DS release, builds 4
   tarballs, and uploads them as release assets
4. `dispatch-mobile-on-release.yml` fires on `release: published`, finds the
   matching tarballs, dispatches one event per sub-package
5. This POC's `design-system-sync.yml` runs once per dispatched event, opens a
   PR with the updated files

### Manual trigger for testing

You can dispatch the event manually using the GitHub CLI. Use a real release
tag and asset URL from the design-system repo.

```bash
gh api repos/amboss-mededu/mobile-token-sync-poc/dispatches \
  -f event_type=design-tokens-ios-update \
  -f 'client_payload[package]=tokens-ios' \
  -f 'client_payload[version]=0.2.0' \
  -f 'client_payload[asset_name]=tokens-ios-0.2.0.tgz' \
  -f 'client_payload[asset_url]=https://github.com/amboss-mededu/amboss-design-system/releases/download/v3.43.3/tokens-ios-0.2.0.tgz' \
  -f 'client_payload[ds_tag]=v3.43.3' \
  -f 'client_payload[ds_release_url]=https://github.com/amboss-mededu/amboss-design-system/releases/tag/v3.43.3'
```

## Adapting for Your Repo

To use this in your actual mobile repo:

1. Copy `.github/workflows/design-system-sync.yml`
2. Edit the **target-path mapping** in the "Resolve target paths" step so each
   sub-package lands in the right directory of your project (e.g.
   `DesignSystem/Resources/Tokens/` for iOS, `app/src/main/res/drawable/` for
   Android)
3. Add or remove `repository_dispatch` event types in the `on:` block depending
   on which sub-packages you consume (e.g. drop `design-icons-ios-update` if
   you don't ship icons)
4. Create the labels referenced in the PR body (`design-tokens`, `design-icons`,
   `automated`) in your repo, or replace with labels you already use

### iOS target paths (example)

| Package | Tarball contents | Suggested target |
|---------|------------------|------------------|
| `tokens-ios` | `Sources/AmbossDesignTokens/*.swift`, `package.json` | `DesignSystem/Resources/Tokens/` |
| `icons-ios` | `*.imageset/Contents.json`, `*.imageset/*.pdf`, `package.json` | `DesignSystem/Resources/Assets.xcassets/AmbossIcons/` |

### Android target paths (example)

| Package | Tarball contents | Suggested target |
|---------|------------------|------------------|
| `tokens-android` | `xml/*.xml`, `compose/*.kt`, `package.json` | `shared-ui/src/main/` (split into `res/values/` and `java/.../compose/`) |
| `icons-android` | `ic_*.xml`, `package.json` | `app/src/main/res/drawable/` |
