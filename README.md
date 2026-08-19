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
ios/fastlane/Fastfile       # iOS lanes     — no relative imports, see below
android/fastlane/Fastfile   # Android lanes
*/actions/.gitkeep          # import_from_git checks out an actions dir per Fastfile
.github/workflows/          # reusable workflows (workflow_call)
```

## Using it in an app

### 1. Fastlane

Each app's Fastfiles become one line:

```ruby
# ios/fastlane/Fastfile
WSM_CI = "https://github.com/whilesmartflutter/wsm_ci.git".freeze
WSM_CI_VERSION = "1.0.0".freeze

# Shared helpers first — the platform lanes call them at parse time.
import_from_git(url: WSM_CI, path: "Fastfile", version: WSM_CI_VERSION)
import_from_git(url: WSM_CI, path: "ios/fastlane/Fastfile", version: WSM_CI_VERSION)
```

Android is the same with `path: "android/fastlane/Fastfile"`.

**Both imports are required.** A Fastfile fetched from git cannot use a
relative `import` to pull in a sibling: fastlane resolves that path against
the *calling* app's directory, not the clone, so
`import "../../Fastfile"` inside a shared file looks for a Fastfile in the app
and fails with `Could not find Fastfile at path`. The shared files therefore
contain no relative imports, and each app imports both pieces explicitly.

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
    uses: whilesmartflutter/wsm_ci/.github/workflows/staging-distribute.yml@1.0.0
    with:
      platforms: ${{ inputs.platforms }}
      release_notes: ${{ inputs.release_notes }}
    secrets:
      ASC_KEY_ID: ${{ secrets.ASC_KEY_ID }}
      ASC_ISSUER_ID: ${{ secrets.ASC_ISSUER_ID }}
      ASC_KEY_P8_BASE64: ${{ secrets.ASC_KEY_P8_BASE64 }}
      APPLE_TEAM_ID: ${{ secrets.APPLE_TEAM_ID }}
      MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
      MATCH_GIT_DEPLOY_KEY: ${{ secrets.MATCH_GIT_DEPLOY_KEY }}
      ANDROID_KEYSTORE_BASE64: ${{ secrets.ANDROID_KEYSTORE_BASE64 }}
      ANDROID_KEYSTORE_PASSWORD: ${{ secrets.ANDROID_KEYSTORE_PASSWORD }}
      ANDROID_KEY_PASSWORD: ${{ secrets.ANDROID_KEY_PASSWORD }}
      FIREBASE_SERVICE_ACCOUNT_JSON: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_JSON }}
      FIREBASE_APP_ID_ANDROID_STAGING: ${{ secrets.FIREBASE_APP_ID_ANDROID_STAGING }}
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
    uses: whilesmartflutter/wsm_ci/.github/workflows/release.yml@1.0.0
    with:
      platform: ${{ inputs.platform || 'all' }}
    secrets:
      # …the release set; see the stubs in whilesmartdesk/mobile
```

**Pass secrets explicitly — `secrets: inherit` does not work here.** The apps
live in `whilesmartdesk`, `WhilesmartPay` and `krewcore`, while this repo is
in `whilesmartflutter`. `inherit` does not carry secrets across
organisations: they arrive empty and the lane fails with a confusing
`ENV "X" is not set`, even though the secret is set on the calling repo.
Repository *variables* (`vars`) do resolve, which makes the failure look
stranger than it is.

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

Tags are **plain semver — `1.0.0`, not `v1.0.0`**. That is not a style choice:
fastlane's `import_from_git` parses `version:` as a `Gem::Requirement`, so a
`v`-prefixed tag fails with `Illformed requirement ["v1"]`. GitHub's `uses:`
accepts any ref, so one semver tag serves both and they move together.

Callers pin an exact tag. **Never point an app at `@main`** — a change here
would reach four release pipelines at once with no review.

Breaking changes (a renamed input, a new required secret) get a new major
version, and apps migrate one at a time.
