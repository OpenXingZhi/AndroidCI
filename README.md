# OpenXingZhi Android CI

Versioned reusable GitHub Actions workflows for XingZhi Android applications and Gradle libraries.

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
      test-task: testDebugUnitTest
      debug-build-task: assembleDebug
      build-task: assembleRelease
      release: ${{ github.event_name == 'workflow_dispatch' && inputs.action == 'release' }}
      keystore-file-property: signingKeystoreFile
      keystore-password-property: signingKeystorePassword
    secrets: inherit
```

Pin callers to a major tag such as `@v1`. Breaking interface changes require a new major tag.

Gradle dependency/distribution caching is enabled through `gradle/actions/setup-gradle`. Pull requests restore caches read-only; trusted branch and tag builds update them after successful cleanup.

Branch pushes, pull requests, and manual `build` runs execute `test-task` and `debug-build-task` together in one Gradle invocation. The signed `build-task` and build-output upload run only for a `v*.*.*` tag or a manual dispatch with `release: true`, so ordinary commits still compile a Debug APK without paying the R8/resource-shrinking cost of a Release build. Set `sign-debug-builds: false` when the app can use Android's default Debug keystore; this keeps the release signing material out of ordinary and pull-request jobs.

Applications without Gradle version/changelog tasks can use a static release version:

```yaml
with:
  release: ${{ github.event_name == 'workflow_dispatch' && inputs.action == 'release' }}
  release-version: ${{ inputs.version || '' }}
  version-task: ""
  release-version-task: ""
  patch-changelog-task: ""
  get-changelog-task: ""
```

## Gradle library build and publish

JVM or Android libraries published with Gradle can call `.github/workflows/gradle-library.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
    tags: ["v*.*.*"]
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  library:
    permissions:
      contents: read
      packages: write
    uses: OpenXingZhi/AndroidCI/.github/workflows/gradle-library.yml@v1
```

Branch pushes and pull requests run `build`. A `v*.*.*` tag first builds, verifies that the tag matches the Gradle project version, then runs `publish` with `packages: write`. Inputs can override `java-version`, `build-task`, and `publish-task`.

## Required Android repository secrets

- `KEYSTORE`: ASCII-armored, symmetrically encrypted keystore content.
- `KEYSTORE_PASSPHRASE`: passphrase used to decrypt `KEYSTORE`.
- `KEYSTORE_PASSWORD`: Android keystore/key password.
- `GRADLE_PROPERTIES` (optional): newline-delimited application-specific Gradle properties.

The workflow automatically exposes GitHub Packages credentials as Gradle properties named `GitHubPackagesUsername` and `GitHubPackagesPassword`.

## Optional Sentry secrets

`android-release.yml` also accepts optional `workflow_call` secrets. Callers that omit them are unchanged: the values are empty, and mapping upload stays off unless `SENTRY_AUTH_TOKEN` is non-empty.

- `SENTRY_AUTH_TOKEN`: Sentry auth token. When it is non-empty, the Sentry Android Gradle Plugin can upload R8 mappings during the signed `assembleRelease` (`build-task`) step.
- `SENTRY_DSN`: exported as `ORG_GRADLE_PROJECT_sentryDsn` so the app's `providers.gradleProperty("sentryDsn")` resolves without a checked-in DSN.
- `SENTRY_ORG`: Sentry organization slug.
- `SENTRY_PROJECT`: Sentry project slug.
- `SENTRY_URL`: optional self-hosted Sentry base URL.

The build job passes `SENTRY_AUTH_TOKEN`, `SENTRY_ORG`, `SENTRY_PROJECT`, `SENTRY_URL`, and `ORG_GRADLE_PROJECT_sentryDsn` through to Gradle. `secrets: inherit` forwards whichever of these exist on the caller. Explicit `secrets:` maps may omit them because each one is `required: false`.

Application `settings.gradle.kts` files can replace the repeated GitHub Packages
`maven { }` blocks with the `com.xingzhi.github-packages` settings plugin
published from [android-bootstrap](https://github.com/OpenXingZhi/android-bootstrap).
`pluginManagement` still needs one authenticated
`https://maven.pkg.github.com/OpenXingZhi/android-bootstrap` repository to resolve
the plugin itself.

## Repository access

Because this repository is private, its Actions access must be set to **Accessible from repositories in the OpenXingZhi organization**. Callers pass secrets with `secrets: inherit`; only secrets declared by this workflow are consumed.
