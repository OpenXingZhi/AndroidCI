# OpenXingZhi Android CI

Versioned reusable GitHub Actions workflows for XingZhi Android applications.

## Android build and release

Call `.github/workflows/android-release.yml` from an application repository:

```yaml
name: Android CI

on:
  push:
    branches: [main]
    tags: ["v*.*.*"]
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      action:
        required: true
        default: build
        type: choice
        options: [build, release]

jobs:
  android:
    permissions:
      contents: write
    uses: OpenXingZhi/AndroidCI/.github/workflows/android-release.yml@v1
    with:
      java-version: "21"
      test-task: test
      build-task: assembleRelease
      release: ${{ github.event_name == 'workflow_dispatch' && inputs.action == 'release' }}
      keystore-file-property: signingKeystoreFile
      keystore-password-property: signingKeystorePassword
    secrets: inherit
```

Pin callers to a major tag such as `@v1`. Breaking interface changes require a new major tag.

## Required repository secrets

- `KEYSTORE`: ASCII-armored, symmetrically encrypted keystore content.
- `KEYSTORE_PASSPHRASE`: passphrase used to decrypt `KEYSTORE`.
- `KEYSTORE_PASSWORD`: Android keystore/key password.
- `GRADLE_PROPERTIES` (optional): newline-delimited application-specific Gradle properties.

The workflow automatically exposes GitHub Packages credentials as Gradle properties named `GitHubPackagesUsername` and `GitHubPackagesPassword`.

## Repository access

Because this repository is private, its Actions access must be set to **Accessible from repositories in the OpenXingZhi organization**. Callers pass secrets with `secrets: inherit`; only secrets declared by this workflow are consumed.
