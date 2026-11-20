# ios-build-ci

Shared GitHub Actions workflows that turn an exported Unity Xcode project into a TestFlight build on a free macOS runner.
This repository contains no project code and no secrets.

## How it works

1. The project's own tooling exports the Unity Xcode project, packs it (`tar.gz`) and encrypts it with AES-256:
   `openssl enc -aes-256-cbc -pbkdf2 -iter 200000 -md sha256 -salt -pass env:IOS_CI_ZIP_PASSWORD`.
2. The encrypted file is attached as `xcode.tar.gz.enc` to a **draft** release with a neutral random tag `ci-<32 hex>`. Drafts are visible only to users with write access.
3. The tooling starts the workflow:
   `gh workflow run ios-testflight.yml -R <owner>/ios-build-ci -f tag=ci-... -f game=<label> -f bundle_id=<bundle id> [-f scheme=Unity-iPhone] [-f upload=true]`
4. `ios-testflight` steps:
   - Checks that the release is a draft, downloads it and deletes the draft immediately, then decrypts it.
   - Runs `pod install` if there is a `Podfile` and uses the `.xcworkspace`.
   - Archives **unsigned**, so the API key is never present while project build scripts run.
   - Signs and uploads with `xcodebuild -exportArchive -allowProvisioningUpdates` (cloud signing with the App Store Connect API key).
5. A `cleanup` job always deletes the draft and its tag. The `ci-janitor` workflow (every 6 h) removes `ci-` drafts older than 3 h.

## Inputs (`ios-testflight`)

| Input | Required | Meaning |
|---|---|---|
| `tag` | yes | draft release tag `ci-<32 hex>` holding `xcode.tar.gz.enc` |
| `game` | yes | project label; concurrency group (one build per label at a time) |
| `bundle_id` | yes | must match `PRODUCT_BUNDLE_IDENTIFIER` in the project |
| `scheme` | no | default `Unity-iPhone` |
| `marketing_version` | no | overrides `CFBundleShortVersionString` |
| `build_number` | no | overrides `CFBundleVersion`; default `<exported build number>.<run number>` |
| `upload` | no | `true` (default) uploads to TestFlight, `false` only archives and signs |
| `sign_in_archive` | no | fallback if unsigned archive + cloud signing fails: signs during the archive (key present during the build) |

## Repository secrets (team-wide)

`APPLE_TEAM_ID`, `ASC_KEY_ID`, `ASC_ISSUER_ID`, `ASC_KEY_P8_B64` (App Store Connect API key `.p8`, base64, role Admin), `IOS_CI_ZIP_PASSWORD` (at least 32 random characters).

## Security (public repository)

- `workflow_dispatch` / `schedule` only; no `push` or `pull_request` triggers, so forks never get the secrets.
- Jobs run only for the repository owner, or users listed in the repository variable `IOS_CI_ALLOWED_ACTORS` (comma-separated).
- No third-party actions. Inputs and values read from the package are validated and passed through environment variables.
- `GH_TOKEN` is set only in the steps that read or delete the release. The API key is decoded only in the export step, masked, and deleted right after.
- No artifacts. On failure the log shows only `error:` lines (max 40). The logs of a public repository are public.
