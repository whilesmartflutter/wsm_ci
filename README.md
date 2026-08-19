# wsm_ci

Shared release pipeline for WhileSmart mobile apps: reusable GitHub workflows
and the Fastlane lanes they drive.

Before this repo, each app carried ~426 lines of workflow YAML and ~505 lines
of Fastfile. krewcore's root Fastfile was byte-identical to Desk's and Pay's
differed by two lines — so the same 900 lines were maintained in triplicate,
and they had already drifted: a hardcoded keystore alias and a missing
TestFlight changelog were fixed in one app and nowhere else.

Only **nine lines per app** are genuinely app-specific (`Appfile`,
`Matchfile`). Everything else lives here.

## Layout

```
Fastfile                    # shared helper lanes (build_flutter_app, versioning, …)
ios/fastlane/Fastfile       # iOS lanes  — imports ../../Fastfile
android/fastlane/Fastfile   # Android lanes
.github/workflows/          # reusable workflows (workflow_call)
```

The tree deliberately mirrors a Flutter app so the Fastfiles' relative
`import "../../Fastfile"` resolves inside a clone.

## Using it in an app

### 1. Fastlane

Each app's Fastfiles become one line:

```ruby
# ios/fastlane/Fastfile
import_from_git(
  url: "https://github.com/whilesmartflutter/wsm_ci.git",
  path: "ios/fastlane/Fastfile",
  version: "v1",
)
```

```ruby
# android/fastlane/Fastfile
import_from_git(
  url: "https://github.com/whilesmartflutter/wsm_ci.git",
  path: "android/fastlane/Fastfile",
  version: "v1",
)
```

`Appfile` and `Matchfile` stay in the app — they hold the bundle id, package
name and certificates repo.

Lanes resolve paths from the runtime working directory, not from where the
Fastfile lives, so importing from git does not change how they find
`pubspec.yaml` or build outputs.

### 2. Workflows

Keep a thin stub in the app that owns the trigger:

```yaml
# .github/workflows/staging-distribute.yml
name: Staging Distribution (Android + iOS)

on:
  workflow_dispatch:
    inputs:
      platforms:
        description: 'Platforms to distribute'
        required: true
        default: 'both'
        type: choice
        options: [android, ios, both]
      release_notes:
        description: 'Release notes'
        required: true
        type: string

jobs:
  distribute:
    uses: whilesmartflutter/wsm_ci/.github/workflows/staging-distribute.yml@v1
    with:
      platforms: ${{ inputs.platforms }}
      release_notes: ${{ inputs.release_notes }}
    secrets: inherit
```

```yaml
# .github/workflows/release.yml
on:
  push:
    tags: ["v*", "v*-rc.*"]
  workflow_dispatch:
    inputs:
      platform:
        type: choice
        options: [all, android, ios]

jobs:
  release:
    uses: whilesmartflutter/wsm_ci/.github/workflows/release.yml@v1
    with:
      platform: ${{ inputs.platform || 'all' }}
    secrets: inherit
```

`secrets: inherit` passes the calling repo's secrets through. That works
because every app uses the **same secret names** — see below.

## Contract: secrets and variables the caller must define

| Secret | Used by |
|---|---|
| `ASC_KEY_ID`, `ASC_ISSUER_ID`, `ASC_KEY_P8_BASE64` | all iOS lanes |
| `APPLE_TEAM_ID` | iOS signing |
| `MATCH_PASSWORD`, `MATCH_GIT_DEPLOY_KEY` | certificate sync |
| `ANDROID_KEYSTORE_BASE64`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_PASSWORD` | Android signing |
| `FIREBASE_SERVICE_ACCOUNT_JSON`, `FIREBASE_APP_ID_ANDROID_STAGING` | Android staging |
| `PLAYSTORE_SERVICE_ACCOUNT_BASE64` | production release |

| Variable | Value |
|---|---|
| `ANDROID_KEY_ALIAS` | keystore alias, e.g. `release` |
| `APP_PACKAGE_NAME` | Android application id |
| `APP_BUNDLE_ID` | iOS bundle id |

An app missing one of these fails inside the shared workflow, so keep the
names identical — that consistency is what makes `secrets: inherit` viable.

## Versioning

Tagged `v1`, `v1.1.0`, …; callers pin a tag. **Never point an app at `@main`**
— a change here would reach four release pipelines at once with no review.

Breaking changes (a renamed input, a new required secret) get a new major tag,
and apps migrate one at a time.
