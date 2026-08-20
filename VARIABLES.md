# ⚙️ Pipeline Variables

> Every setting the pipeline reads from **`pipeline.properties`** — what it is, and
> why it lives there.

The `Jenkinsfile` is identical for every project. This file is what makes each
project different.

---

## 📖 At a glance

|  #  | Variable                                                                |         Required          | In one line                                         |
| :-: | :---------------------------------------------------------------------- | :-----------------------: | :-------------------------------------------------- |
|  1  | [`FLUTTER_VERSION`](#-1-flutter_version)                                |         ✅ Always         | Which Flutter SDK builds the app                    |
|  2  | [`GIT_PROTOCOL`](#-2-git_protocol)                                      |         ✅ Always         | How to reach GitLab                                 |
|  3  | [`GIT_HOST`](#-3-git_host)                                              |         ✅ Always         | Which GitLab server                                 |
|  4  | [`GIT_REPO_PATH`](#-4-git_repo_path)                                    |         ✅ Always         | Which project on that server                        |
|  5  | [`GIT_BRANCH`](#-5-git_branch)                                          |         ✅ Always         | Which branch to clone                               |
|  6  | [`GIT_CREDENTIALS_ID`](#-6-git_credentials_id)                          |         ✅ Always         | Where the git credential is kept                    |
|  7  | [`INFISICAL_API_URL`](#-7-infisical_api_url)                            |         ✅ Always         | Which Infisical server                              |
|  8  | [`INFISICAL_PROJECT_ID`](#-8-infisical_project_id)                      |         ✅ Always         | Which Infisical project                             |
|  9  | [`INFISICAL_ENV`](#-9-infisical_env)                                    |         ✅ Always         | Which Infisical environment                         |
| 10  | [`INFISICAL_TOKEN_CREDENTIALS_ID`](#-10-infisical_token_credentials_id) |         ✅ Always         | Where the Infisical token is kept                   |
| 11  | [`SHOREBIRD_TOKEN_CREDENTIALS_ID`](#-11-shorebird_token_credentials_id) |     ⚠️ Shorebird only     | Where the Shorebird token is kept                   |
| 12  | [`S3_ENDPOINT`](#-12-s3_endpoint)                                       |   ⚠️ With secret files    | Which storage service the files come from           |
| 13  | [`S3_REGION`](#-13-s3_region)                                           |   ⚠️ With secret files    | Which region the bucket is in                       |
| 14  | [`S3_CREDENTIALS_ID`](#-14-s3_credentials_id)                           |   ⚠️ With secret files    | Where the storage keys are kept                     |
| 15  | [`SECRET_FILE.*`](#-15-secret_file)                                     |        ⚠️ Per file        | Which file goes where in the project                |
| 16  | [`ASC_KEY_ID`](#-16-asc_key_id)                                         | ⚠️ iOS builds only        | Which App Store Connect API key signs the upload    |
| 17  | [`ASC_ISSUER_ID`](#-17-asc_issuer_id)                                   | ⚠️ iOS builds only        | Which App Store Connect account that key belongs to |
| 18  | [`ANDROID_PACKAGE_NAME`](#-18-android_package_name)                     | ⚠️ Android builds only    | Which Play listing the build is uploaded to         |

---

## 🎯 1. `FLUTTER_VERSION`

|              |                     |
| :----------- | :------------------ |
| **Example**  | `3.44.6`            |
| **Format**   | `MAJOR.MINOR.PATCH` |
| **Required** | ✅ Always           |

**What it is** — the exact Flutter SDK version to build with.

```properties
FLUTTER_VERSION=3.44.6
```

**Why it exists** — a project is only reliable on the SDK it was written for. Put
the version here and FVM installs that exact SDK on the agent, so the same commit
produces the same build on any machine, today or in six months.

> [!NOTE]
> This value wins over a `.fvmrc` in the project repo. One file decides the
> version, so there is never a second answer to argue with.

---

## 🔌 2. `GIT_PROTOCOL`

|              |                  |
| :----------- | :--------------- |
| **Example**  | `https`          |
| **Format**   | `ssh` or `https` |
| **Required** | ✅ Always        |

**What it is** — how the pipeline reaches GitLab.

```properties
GIT_PROTOCOL=https
```

It decides how the clone URL is built:

| Value   | URL                     |
| :------ | :---------------------- |
| `ssh`   | `git@HOST:PATH.git`     |
| `https` | `https://HOST/PATH.git` |

**Why it exists** — SSH needs port 22 open to the GitLab server. That is normal
inside the office network but often blocked from outside it, where only HTTPS gets
through. One setting lets the same pipeline run in both places.

> [!IMPORTANT]
> `GIT_CREDENTIALS_ID` must match. `ssh` needs an SSH key credential, `https` needs
> a username and password credential.

---

## 🌐 3. `GIT_HOST`

|              |                                             |
| :----------- | :------------------------------------------ |
| **Example**  | `gitlab.webelight.co.in`                    |
| **Format**   | A hostname, no scheme and no trailing slash |
| **Required** | ✅ Always                                   |

**What it is** — the GitLab server the project lives on.

```properties
GIT_HOST=gitlab.webelight.co.in
```

**Why it exists** — it is the same for every project on the server. Keeping it
separate from the project path means the clone URL is assembled in one place
instead of being spelled out again in every project.

---

## 📁 4. `GIT_REPO_PATH`

|              |                                                       |
| :----------- | :---------------------------------------------------- |
| **Example**  | `webelight/ccmt/chinmaya-mission-flutter`             |
| **Format**   | Group and project, no leading slash, no `.git` suffix |
| **Required** | ✅ Always                                             |

**What it is** — which project to clone from that server.

```properties
GIT_REPO_PATH=webelight/ccmt/chinmaya-mission-flutter
```

**Why it exists** — this is the one part of the clone URL that changes per
project. It is combined with `GIT_HOST` into an SSH URL:

```text
git@gitlab.webelight.co.in:webelight/ccmt/chinmaya-mission-flutter.git
```

---

## 🌿 5. `GIT_BRANCH`

|              |               |
| :----------- | :------------ |
| **Example**  | `main`        |
| **Format**   | A branch name |
| **Required** | ✅ Always     |

**What it is** — the branch to clone.

```properties
GIT_BRANCH=main
```

**Why it exists** — a clone needs a named ref to be repeatable. Stating the branch
means two runs of the same configuration fetch the same thing, instead of
depending on whatever the remote's default happens to be.

---

## 🔑 6. `GIT_CREDENTIALS_ID`

|              |                         |
| :----------- | :---------------------- |
| **Example**  | `gitlab-token`          |
| **Format**   | A Jenkins credential ID |
| **Required** | ✅ Always               |

**What it is** — the **name** of a Jenkins credential. Never the key or token
itself.

```properties
GIT_CREDENTIALS_ID=gitlab-token
```

The credential is created in Jenkins, and its kind depends on `GIT_PROTOCOL`:

| `GIT_PROTOCOL` | Kind                          | Username | Secret           |
| :------------- | :---------------------------- | :------- | :--------------- |
| `ssh`          | SSH Username with private key | `git`    | The private key  |
| `https`        | Username with password        | `oauth2` | The access token |

**Why it exists** — cloning a private repository needs credentials, and this file
is not a safe place for them. Storing only the name keeps the secret inside
Jenkins, where the pipeline can read it but git history and build logs never see
it.

> [!WARNING]
> Putting a raw token here instead of a credential ID is rejected by the pipeline.
> It would land in git history and in the build log in plain text.

---

## ☁️ 7. `INFISICAL_API_URL`

|              |                             |
| :----------- | :-------------------------- |
| **Example**  | `https://app.infisical.com` |
| **Format**   | A URL, no trailing slash    |
| **Required** | ✅ Always                   |

**What it is** — the Infisical server the secrets are read from.

```properties
INFISICAL_API_URL=https://app.infisical.com
```

**Why it exists** — teams running a self-hosted Infisical need a different address.
Keeping it here means the pipeline works against either without a code change.

---

## 🆔 8. `INFISICAL_PROJECT_ID`

|              |                                             |
| :----------- | :------------------------------------------ |
| **Example**  | `7f3a1c2e-...`                              |
| **Format**   | The project ID from the Infisical dashboard |
| **Required** | ✅ Always                                   |

**What it is** — which Infisical project holds this app's secrets.

```properties
INFISICAL_PROJECT_ID=7f3a1c2e-4b5d-6789-abcd-ef0123456789
```

**Why it exists** — one Infisical account holds many projects. This is the per-app
part, so it changes for every project using this pipeline.

---

## 🏷️ 9. `INFISICAL_ENV`

|              |                                                |
| :----------- | :--------------------------------------------- |
| **Example**  | `dev`                                          |
| **Format**   | An environment slug from the Infisical project |
| **Required** | ✅ Always                                      |

**What it is** — which set of values to export, e.g. `dev`, `staging`, `prod`.

```properties
INFISICAL_ENV=dev
```

**Why it exists** — the same variable names hold different values per environment.
This decides which set lands in `.env`.

---

## 🎫 10. `INFISICAL_TOKEN_CREDENTIALS_ID`

|              |                         |
| :----------- | :---------------------- |
| **Example**  | `infisical-token`       |
| **Format**   | A Jenkins credential ID |
| **Required** | ✅ Always               |

**What it is** — the **name** of a Jenkins credential. Never the token.

```properties
INFISICAL_TOKEN_CREDENTIALS_ID=infisical-token
```

The credential is created in Jenkins:

| Field  | Value                       |
| :----- | :-------------------------- |
| Kind   | Secret text                 |
| Secret | The Infisical service token |
| ID     | `infisical-token`           |

**Why it exists** — reading secrets requires a secret of its own. Storing only the
name keeps the token inside Jenkins, and the pipeline hands it to the CLI through
the environment rather than a command line, so it never reaches the build log.

---

## 🐦 11. `SHOREBIRD_TOKEN_CREDENTIALS_ID`

|              |                                              |
| :----------- | :------------------------------------------- |
| **Example**  | `shorebird-token`                            |
| **Format**   | A Jenkins credential ID                      |
| **Required** | ⚠️ Only when the build runner is `Shorebird` |

**What it is** — the **name** of a Jenkins credential. Not the token.

```properties
SHOREBIRD_TOKEN_CREDENTIALS_ID=shorebird-token
```

**Why it exists** — Shorebird needs a token to publish a release. Same reasoning as
above: the name lives here, the secret stays in Jenkins.

> [!IMPORTANT]
> A plain `Flutter` build never reads this. Projects that do not release through
> Shorebird can leave the line out entirely.

---

## 🪣 12. `S3_ENDPOINT`

|              |                                              |
| :----------- | :------------------------------------------- |
| **Example**  | `https://a1b2c3d4.r2.cloudflarestorage.com`  |
| **Format**   | A URL, no trailing slash                     |
| **Required** | ⚠️ Only when `SECRET_FILE.*` entries are set |

**What it is** — the API address of the object storage the secret files come from.

```properties
S3_ENDPOINT=https://a1b2c3d4.r2.cloudflarestorage.com
```

Any service that speaks the S3 API works. The endpoint is the only thing that
changes between them:

| Service             | Endpoint                                        |
| :------------------ | :---------------------------------------------- |
| AWS S3              | `https://s3.<region>.amazonaws.com`             |
| Cloudflare R2       | `https://<account-id>.r2.cloudflarestorage.com` |
| MinIO               | `https://minio.example.com:9000`                |
| Backblaze B2        | `https://s3.<region>.backblazeb2.com`           |
| DigitalOcean Spaces | `https://<region>.digitaloceanspaces.com`       |

**Why it exists** — the pipeline downloads with the aws CLI, which talks the S3
protocol to whatever address it is given. Keeping that address here means moving
buckets between providers is a one-line change, with no `Jenkinsfile` edit.

---

## 🌍 13. `S3_REGION`

|              |                                              |
| :----------- | :------------------------------------------- |
| **Example**  | `auto`                                       |
| **Format**   | A region name                                |
| **Required** | ⚠️ Only when `SECRET_FILE.*` entries are set |

**What it is** — the region the bucket lives in.

```properties
S3_REGION=auto
```

| Service       | Value                                       |
| :------------ | :------------------------------------------ |
| AWS S3        | The bucket's real region, e.g. `ap-south-1` |
| Cloudflare R2 | `auto`                                      |
| MinIO         | `us-east-1` unless configured otherwise     |

**Why it exists** — the S3 protocol signs every request with a region, so one is
always required even where the service has no regions. Providers that do have them
reject a wrong value, so it cannot be hardcoded.

---

## 🗝️ 14. `S3_CREDENTIALS_ID`

|              |                                              |
| :----------- | :------------------------------------------- |
| **Example**  | `object-storage-credentials`                 |
| **Format**   | A Jenkins credential ID                      |
| **Required** | ⚠️ Only when `SECRET_FILE.*` entries are set |

**What it is** — the **name** of a Jenkins credential. Never the keys themselves.

```properties
S3_CREDENTIALS_ID=object-storage-credentials
```

The credential is created in Jenkins:

| Field    | Value                        |
| :------- | :--------------------------- |
| Kind     | Username with password       |
| Username | Access Key ID                |
| Password | Secret Access Key            |
| ID       | `object-storage-credentials` |

Read-only access to the one bucket is enough — the pipeline only downloads.

**Why it exists** — same reasoning as every other credential ID here. The keys stay
in Jenkins, and the pipeline hands them to the aws CLI through the environment, so
they never reach a command line or the build log.

> [!WARNING]
> Putting a raw access key here instead of a credential ID is rejected by the
> pipeline, the same as for the other credential IDs.

---

## 📦 15. `SECRET_FILE.*`

|              |                                                                    |
| :----------- | :----------------------------------------------------------------- |
| **Example**  | `SECRET_FILE.android/key.properties=s3://my-bucket/key.properties` |
| **Format**   | `SECRET_FILE.<path in the project>=<s3 uri>`                       |
| **Required** | ⚠️ One line per file, none is fine                                 |

**What it is** — a map. The key is where the file lands in the checked out project,
the value is where it comes from in the bucket.

```properties
SECRET_FILE.android/app/src/prod/google-services.json=s3://my-bucket/google-services.json
SECRET_FILE.android/key.properties=s3://my-bucket/key.properties
SECRET_FILE.android/cm-app.jks=s3://my-bucket/cm-app.jks
SECRET_FILE.ios/Runner/GoogleService-Info.plist=s3://my-bucket/GoogleService-Info.plist
```

Read one line as a sentence:

```text
SECRET_FILE.  android/key.properties  =  s3://my-bucket/key.properties
└─ prefix ─┘  └─ where it goes ────┘     └─ where it comes from ─────┘
```

Destination paths are relative to the project root, and missing directories are
created. The prefix is stripped, so whatever follows it _is_ the path.

> [!NOTE]
> The `s3://` scheme is how the aws CLI addresses a bucket, on every provider. It
> does not mean the storage has to be AWS.

**Why it exists** — a keystore, a `google-services.json` and an iOS plist cannot be
committed, but the build needs them in exact places. Listing them as pairs keeps
both halves in one line, so the bucket can be organised however you like and the
project layout still decides where files land.

> [!TIP]
> Adding a file is one line here. No `Jenkinsfile` change — the stage downloads
> whatever it finds.

> [!NOTE]
> A project with no `SECRET_FILE.*` lines skips the download, and then the `S3_*`
> settings are not required either.

---

## 🔑 16. `ASC_KEY_ID`

|              |                                                      |
| :----------- | :--------------------------------------------------- |
| **Example**  | `2X9R4HXF34`                                         |
| **Format**   | The Key ID from App Store Connect                    |
| **Required** | ⚠️ Only for an iOS build                            |

**What it is** — the short id of the App Store Connect API key that uploads the
build. Not a secret, and it is also the middle of the `.p8` filename.

```properties
ASC_KEY_ID=2X9R4HXF34
SECRET_FILE.private_keys/AuthKey.p8=s3://my-bucket/AuthKey.p8
```

**Why it exists** — `xcrun altool` names the key on the command line and reads the
matching file from `private_keys/` in the project root. The value here has to be the
same id in both places, so the upload stage builds the path from it and fails early
when the file is missing.

> [!IMPORTANT]
> The `.p8` is the secret and travels as a `SECRET_FILE.*` entry like every other
> credential file. Apple fixes its name as `AuthKey_<Key ID>.p8` — rename it and
> `altool` will not find it.

---

## 🆔 17. `ASC_ISSUER_ID`

|              |                                                      |
| :----------- | :--------------------------------------------------- |
| **Example**  | `69a6de7e-1234-4c5b-8f21-9a0bcdef1234`               |
| **Format**   | The Issuer ID from App Store Connect                 |
| **Required** | ⚠️ Only for an iOS build                            |

**What it is** — the id of the App Store Connect account the key was created in. It
sits at the top of **Users and Access → Integrations → App Store Connect API**.

```properties
ASC_ISSUER_ID=69a6de7e-1234-4c5b-8f21-9a0bcdef1234
```

**Why it exists** — Apple authenticates the upload with all three pieces: the Key ID,
the Issuer ID and the `.p8`. The Issuer ID cannot be read off the file, so it has to
be written down here.

---

## 🤖 18. `ANDROID_PACKAGE_NAME`

|              |                                                      |
| :----------- | :--------------------------------------------------- |
| **Example**  | `com.chinmayamission.app`                            |
| **Format**   | The application id, reverse-DNS                       |
| **Required** | ⚠️ Only for an Android build                        |

**What it is** — the package name Google Play lists the app under, the same value as
the `prod` flavour's `applicationId`. Not a secret.

```properties
ANDROID_PACKAGE_NAME=com.chinmayamission.app
SECRET_FILE.private_keys/play-store.json=s3://my-bucket/play-store.json
```

**Why it exists** — `fastlane supply` addresses a Play listing by package name, not by
repository or artifact, so it has to be written down. The upload stage checks the
shape before the build starts, because a typo here otherwise fails an hour later
against Play's API instead of in the first ten seconds.

> [!IMPORTANT]
> The service account JSON is the secret and travels as a `SECRET_FILE.*` entry like
> the Apple `.p8`. The path is fixed at `private_keys/play-store.json` — the upload
> stage looks there and nowhere else.

> [!NOTE]
> Every Android build lands on the **internal testing** track with the release live
> for internal testers. Promotion to closed, open or production testing stays a
> manual step in the Play Console.

---

## ➕ Adding a variable

| Step | Action                                          |
| :--: | :---------------------------------------------- |
|  1   | Add a `KEY=VALUE` line to `pipeline.properties` |
|  2   | Document it here                                |

> [!TIP]
> No `Jenkinsfile` change is needed — the agent reads the file fresh on every
> build, so a new value takes effect on the next run.

---

<sub>Format: `KEY=VALUE`, one per line. Lines starting with `#` are comments.</sub>
