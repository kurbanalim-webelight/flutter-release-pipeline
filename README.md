# 🚀 Flutter Release Pipeline

> The steps in the [`Jenkinsfile`](Jenkinsfile), and what each one does.

---

## 📑 Table of contents

| # | Step |
| :-: | :--- |
| 1 | [Validate & Load Configuration](#-1-validate--load-configuration) |
| 2 | [Clone Repository](#-2-clone-repository) |
| 3 | [Setup Flutter](#-3-setup-flutter) |
| 4 | [Load Secret Files](#-4-load-secret-files) |
| 5 | [Download Secret Files](#-5-download-secret-files) |
| 6 | [Install Dependencies](#-6-install-dependencies) |
| 7 | [Build Release](#-7-build-release) |
| 8 | [Upload Release](#-8-upload-release) |

---

## ✅ 1. Validate & Load Configuration

Checks the user's input, then reads [`pipeline.properties`](pipeline.properties)
and makes its values available to the rest of the pipeline.

| Input | Rule |
| :---- | :--- |
| `PLATFORM` | `Android` or `iOS` |
| `BUILD_RUNNER` | `Flutter` for a normal build, `Shorebird` for a code push release |
| `BUILD_VERSION` | Present, and matches `MAJOR.MINOR.PATCH` |
| `APP_BUILD_NUMBER` | Present, digits only, and `≥ 1` |
| `SET_RELEASE_NAME` | Android only. Ticked means `RELEASE_NAME` is present and `≤ 50` characters |
| `SET_RELEASE_NOTES` | Android only. Ticked means `RELEASE_NOTES` is present and `≤ 500` characters |

Names the run so the build history is readable:

```text
#17 Android/Flutter 1.0.0+32
```

| Action | Detail |
| :----- | :----- |
| Parse | `KEY=VALUE` per line; `#` and `!` start a comment |
| Require | The `FLUTTER_*`, `GIT_*` and `INFISICAL_*` keys |
| Build | The clone URL from `GIT_PROTOCOL`, `GIT_HOST` and `GIT_REPO_PATH` |
| Reject | Any `*_CREDENTIALS_ID` that looks like a raw token instead of an ID |
| Require | `S3_ENDPOINT`, `S3_REGION` and `S3_CREDENTIALS_ID`, but only when `SECRET_FILE.*` entries exist |
| Require | `SHOREBIRD_TOKEN_CREDENTIALS_ID`, but only when `BUILD_RUNNER` is `Shorebird` |
| Export | Each value becomes an environment variable for later steps |

The flavour is not an input. Release builds always ship `prod`, so `BUILD_FLAVOR`
and `DART_ENTRYPOINT` are fixed in the pipeline's `environment` block.

### Shorebird is checked up front

A `Shorebird` run ends this stage by proving the token is non-empty and installing
the CLI if the agent does not have it, so a bad token fails in seconds instead of
after the clone, the secrets and the whole dependency install.

---

## 📥 2. Clone Repository

Clones the GitLab project into the workspace.

| # | Action |
| :-: | :----- |
| 1 | Clone at branch `GIT_BRANCH`, authenticating with `GIT_CREDENTIALS_ID` |
| 2 | Print the commit that was checked out |

### Both SSH and HTTPS are supported

`GIT_PROTOCOL` in [`pipeline.properties`](pipeline.properties) decides which one is
used. Anything other than `ssh` or `https` fails validation.

| `GIT_PROTOCOL` | URL built | Credential kind `GIT_CREDENTIALS_ID` must point to |
| :------------- | :-------- | :------------------------------------------------- |
| `ssh` | `git@HOST:PATH.git` | SSH Username with private key — username `git` |
| `https` | `https://HOST/PATH.git` | Username with password — username `oauth2`, password is the access token |

```text
ssh    git@gitlab.webelight.co.in:webelight/ccmt/chinmaya-mission-flutter.git
https  https://gitlab.webelight.co.in/webelight/ccmt/chinmaya-mission-flutter.git
```

> [!IMPORTANT]
> The credential must match the protocol. `GIT_CREDENTIALS_ID` names a single
> credential, so switching `GIT_PROTOCOL` also means pointing it at a credential of
> the matching kind. Create both once, then change the two lines together.

> [!NOTE]
> `ssh` needs port 22 open to the GitLab host. That holds inside the office network
> but is usually blocked outside it, where the clone times out after about 75
> seconds. Use `https` when working from outside.

---

## 🐦 3. Setup Flutter

Prepares the toolchain, installs the SDK, and confirms the right one is active.

| # | Action |
| :-: | :----- |
| 1 | If `fvm` is absent, install it with Homebrew |
| 2 | `fvm install "$FLUTTER_VERSION" --setup` — downloads the SDK if not cached |
| 3 | `fvm use "$FLUTTER_VERSION" --force` — pins the workspace to that SDK |
| 4 | Compare the active version against `FLUTTER_VERSION` as an exact string |

> [!IMPORTANT]
> This step runs **after** the clone, not before it. `fvm use` pins a version by
> writing `.fvmrc` into the project it is standing in, so pinning an empty
> workspace has no effect on the build and the version check passes against
> nothing. Cloning first means the pin lands in the app, and every later
> `fvm flutter` call gets the SDK named in `FLUTTER_VERSION`.

```text
expected: 3.44.6
actual:   3.44.6
Flutter version matches
```

---

## 🔐 4. Load Secret Files

Fetches secrets that are not in the repository and writes them into the project.

| # | Action |
| :-: | :----- |
| 1 | If the Infisical CLI is absent, install it with Homebrew |
| 2 | Export the `INFISICAL_ENV` environment of project `INFISICAL_PROJECT_ID` |
| 3 | Write the result to `.env` in the project root |
| 4 | Fail if `.env` came back empty |

The token is read from the Jenkins credential named by
`INFISICAL_TOKEN_CREDENTIALS_ID` and passed to the CLI through the environment, so
it never appears in a command line.

```text
.env written to /Users/…/workspace/flutter-release-pipeline with 24 variable(s)
```

> [!NOTE]
> Only the number of variables is logged, never their names or values.

---

## 📦 5. Download Secret Files

Pulls the files that cannot be committed — the keystore, `google-services.json`,
`key.properties`, the iOS plist — out of object storage and into the checkout. It is
the second half of the `Load Secret Files` stage, documented on its own because it
is configured separately.

Each one is a line in [`pipeline.properties`](pipeline.properties):

```properties
SECRET_FILE.android/key.properties=s3://my-bucket/key.properties
└─ prefix ─┘└─ where it goes ───┘   └─ where it comes from ────┘
```

| # | Action |
| :-: | :----- |
| 1 | If the aws CLI is absent, install it with Homebrew |
| 2 | Collect every `SECRET_FILE.*` entry into destination → source pairs |
| 3 | Create the destination directory, then `aws s3 cp` the file into it |
| 4 | Fail if a downloaded file is empty |

```text
android/key.properties <- s3://my-bucket/key.properties
android/cm-app.jks <- s3://my-bucket/cm-app.jks
ios/Runner/GoogleService-Info.plist <- s3://my-bucket/GoogleService-Info.plist
Downloaded 3 secret file(s)
```

### Any S3-compatible storage works

The download is a plain `aws s3 cp` aimed at `S3_ENDPOINT` with `S3_REGION`, so
AWS S3, Cloudflare R2, MinIO, Backblaze B2 and DigitalOcean Spaces are all just a
different endpoint. The keys come from the Jenkins credential named by
`S3_CREDENTIALS_ID` and are passed through the environment, never on a command
line.

> [!NOTE]
> The stage runs after the clone, so destinations land inside the project. Running
> it earlier would only get the files wiped by `git`.

> [!TIP]
> Adding a file is one line in [`pipeline.properties`](pipeline.properties). A
> project with no `SECRET_FILE.*` lines skips the stage entirely.

---

## 🧩 6. Install Dependencies

Resolves the packages the build needs, before anything tries to compile.

| # | Action |
| :-: | :----- |
| 1 | `fvm flutter clean` — drop any output left by an earlier build |
| 2 | `fvm flutter pub get` — resolve the Dart packages with the pinned SDK |
| 3 | iOS only: if the CocoaPods CLI is absent, install it with Homebrew |
| 4 | iOS only: `fvm flutter precache --ios` — download the iOS engine artifacts |
| 5 | iOS only: `cd ios && pod install --repo-update` |
| 6 | `easy_localization:generate` — build `locale_keys.g.dart` from `assets/l10n` |
| 7 | `build_runner build --delete-conflicting-outputs` — everything else `*.g.dart` |
| 8 | Fail if no `*.g.dart` exists under `lib/` |

```text
Got dependencies!
pod: 1.16.2
Pod installation complete!
🧬  38 generated file(s) under lib/
```

> [!IMPORTANT]
> Codegen is not optional and it cannot move earlier. `*.g.dart` is git-ignored, so
> a fresh clone ships none of it and the build will not compile — and Envied reads
> the `.env` written by [step 4](#-4-load-secret-files), so this has to run after it.

> [!NOTE]
> Always `fvm flutter`, never plain `flutter`. Plain `flutter` uses whatever SDK is
> on the machine's `PATH`, which is how a build silently stops matching
> `FLUTTER_VERSION`.

> [!TIP]
> `precache` is not optional on a freshly installed SDK. The Podfile's post-install
> hook needs `Flutter.xcframework` from the SDK's artifact cache, and `pub get` does
> not fetch it — `flutter build ios` normally does it implicitly, which is why a
> hand-run `pod install` can pass on a machine that has already built once.

> [!TIP]
> The pods step is skipped for an Android build. `flutter build ios` runs
> `pod install` on its own too, so this step mostly buys an earlier and clearer
> failure. `--repo-update` refreshes the CocoaPods spec repo on every run — drop it
> if the extra minute costs more than it saves.

---

## 🏗️ 7. Build Release

Compiles the release artifact. `PLATFORM` picks the platform, `BUILD_RUNNER` picks
the tool.

| `PLATFORM` | `BUILD_RUNNER` | What runs |
| :--------- | :------------- | :-------- |
| `Android` | `Flutter` | `fvm flutter build appbundle --release` |
| `Android` | `Shorebird` | `shorebird release -p android --artifact aab` |
| `iOS` | either | Work in progress |

Both Android paths get the same version and flavour arguments, because both CLIs
spell them the same way:

```text
--flavor prod  -t lib/config/flavours/prod/prod.dart
--build-name=$BUILD_VERSION  --build-number=$APP_BUILD_NUMBER
```

The Flutter path then locates the `.aab` under `build/app/outputs/bundle` and fails
if there is none, so a build that "succeeded" without producing an artifact is
still a red build.

```text
📦  build/app/outputs/bundle/prodRelease/app-prod-release.aab (52M)
```

### The jitpack fix

Every Android build first writes `~/.gradle/init.d/jitpack-www.gradle`, which
rewrites `https://www.jitpack.io` to `https://jitpack.io` for every repository
Gradle resolves.

> [!NOTE]
> `image_cropper` 9.1.0 declares the `www` host, whose IPv4 address serves a GitHub
> certificate, so Gradle cannot fetch `com.github.yalantis:ucrop`. The apex host is
> healthy and is what `image_cropper` itself moved to in 11.0.0. Doing it in an init
> script fixes every Gradle build on the agent without patching the app repo — delete
> `ensureGradleJitpackFix` once the app upgrades past 11.0.0.

---

## 🚀 8. Upload Release

Ships the artifact the previous stage produced. `PLATFORM` picks the destination;
`BUILD_RUNNER` makes no difference, because a Shorebird release is a full build too.

| `PLATFORM` | Destination | Tool |
| :--------- | :---------- | :--- |
| `Android` | Play **internal testing** track, released to testers | `fastlane supply` |
| `iOS` | TestFlight | `xcrun altool --upload-app` |

The Android path:

| # | Action |
| :-: | :----- |
| 1 | If the fastlane CLI is absent, install it with Homebrew |
| 2 | Write the release notes file, if `SET_RELEASE_NOTES` is ticked |
| 3 | Locate the `.aab` under `build/app/outputs/bundle`, fail if there is none |
| 4 | Fail if `private_keys/play-store.json` did not download |
| 5 | `fastlane supply --track internal --release_status completed` |

### Naming the release and telling testers what changed

Both are off by default and each has its own checkbox, so a normal run needs neither.

| Ticked | Field | What the pipeline does |
| :----- | :---- | :--------------------- |
| `SET_RELEASE_NAME` | `RELEASE_NAME` | Passes it as `--version_name`. Unticked sends `VERSION+BUILD`, e.g. `1.0.5+13` |
| `SET_RELEASE_NOTES` | `RELEASE_NOTES` | Writes the text to `metadata/android/en-US/changelogs/<APP_BUILD_NUMBER>.txt` and points `--metadata_path` at it. Unticked passes `--skip_upload_changelogs` |

A ticked box with an empty or over-long field fails in
[step 1](#-1-validate--load-configuration), before anything is built.

> [!NOTE]
> `supply` has no release-notes argument — the file path *is* the interface, which is
> why the version code has to be in the filename. Notes go up in `en-US` only.

> [!NOTE]
> Release notes are the one piece of listing text CI writes. Store listing, images
> and screenshots stay skipped, so a run can never overwrite them.

```text
📤  Uploading build/app/outputs/bundle/prodRelease/app-prod-release.aab (52M) to com.chinmayamission.app
✅  Uploaded to internal testing - Google processes the build before testers see it
```

> [!IMPORTANT]
> `ANDROID_PACKAGE_NAME` must match the application id of the listing exactly, and
> the service account's `client_email` needs release permission on that app inside
> the Play Console. The key existing in Google Cloud grants nothing on its own —
> see the [Developer Handbook](DEVELOPER_HANDBOOK.md#-android).

> [!NOTE]
> Only metadata upload is skipped, not the release itself: store listing, images,
> screenshots and changelogs stay whatever the Play Console already has, so CI never
> overwrites text somebody edited by hand.

> [!NOTE]
> The first upload of a package cannot come from the API — Play requires one manual
> `.aab` upload to create the app entry before the service account can add builds.

> [!TIP]
> A build number Play has already seen is rejected. `APP_BUILD_NUMBER` has to go up
> on every run that reaches the same track.
