# Mobile Token Sync POC

Proof of concept for automatically syncing design tokens from
[amboss-design-system](https://github.com/amboss-mededu/amboss-design-system)
to mobile repositories via GitHub Actions.

## How It Works

```
amboss-design-system                     mobile repo (this POC)
┌─────────────────────┐                  ┌─────────────────────────┐
│ PR merged to main   │                  │                         │
│ with version bump   │                  │  repository_dispatch    │
│ in tokens-ios or    │ ──dispatch────▶  │  workflow triggers:     │
│ tokens-android      │                  │  1. checkout DS at SHA  │
│ package.json        │                  │  2. copy token files    │
│                     │                  │  3. open PR             │
└─────────────────────┘                  └─────────────────────────┘
```

### Dispatcher (design-system side)

Workflow: `.github/workflows/trigger-token-sync-mobile.yml`

- Triggers on push to `main` when `packages/tokens-ios/package.json` or
  `packages/tokens-android/package.json` changes
- Compares the `version` field before/after the merge
- If the version changed, dispatches a `design-tokens-update` event to the
  corresponding mobile repo with payload:
  ```json
  {
    "version": "0.2.0",
    "previous_version": "0.1.0",
    "sha": "abc123..."
  }
  ```

### Receiver (mobile repo side)

Workflow: `.github/workflows/design-token-sync.yml`

- Triggers on `repository_dispatch` event type `design-tokens-update`
- Checks out `amboss-design-system` at the exact SHA from the payload
  (uses sparse checkout for speed)
- Copies token files to the target paths defined in the workflow
- Creates a PR via `peter-evans/create-pull-request`

## Testing the POC

### Prerequisites

- A GitHub PAT with `repo` scope, stored as `MOBILE_REPOS_ACCESS_TOKEN` secret
  in the `amboss-design-system` repo

### Steps

1. In `amboss-design-system`, merge the branch `chore/mobile-token-sync-dispatch`
   to `main`
2. Create a PR in `amboss-design-system` that bumps the version in
   `packages/tokens-ios/package.json` (e.g., `0.1.0` -> `0.2.0`)
3. Merge that PR to `main`
4. The dispatcher workflow runs, detects the version change, and dispatches
   to this POC repo
5. This repo's workflow creates a PR with the updated token files

### Manual trigger for testing

You can also trigger the receiver workflow manually using the GitHub CLI:

```bash
gh api repos/amboss-mededu/mobile-token-sync-poc/dispatches \
  -f event_type=design-tokens-update \
  -f 'client_payload[version]=0.2.0' \
  -f 'client_payload[previous_version]=0.1.0' \
  -f 'client_payload[sha]=<commit-sha-from-design-system>'
```

## Adapting for Your Repo

To use this in your actual mobile repo:

1. Copy `.github/workflows/design-token-sync.yml` to your repo
2. Edit the **file mapping** section in the "Copy token files" step to match
   your repo's directory structure
3. Update the `sparse-checkout` to match your platform (`tokens-ios` or
   `tokens-android`)
4. Update the PR body to list your actual target files
5. Create the `design-tokens` and `automated` labels in your repo (optional)

### iOS example paths

```bash
cp design-system/packages/tokens-ios/Sources/AmbossDesignTokens/DesignSystemColorTokens.swift \
   DesignSystem/DesignSystem/Resources/Token/DesignSystemColorTokens.swift
```

### Android example paths

```bash
cp design-system/packages/tokens-android/compose/ColorTokens.kt \
   shared-ui/src/main/java/de/miamed/amboss/shared/ui/compose/ColorTokens.kt

cp design-system/packages/tokens-android/xml/design_system_color.xml \
   shared-ui/src/main/res/values/design_system_color.xml
```
