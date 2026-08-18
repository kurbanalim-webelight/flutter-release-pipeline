# ⚙️ Pipeline Variables

> Every setting the pipeline reads from **`pipeline.properties`** — what it is, and
> why it lives there.

The `Jenkinsfile` is identical for every project. This file is what makes each
project different.

---

## 📖 At a glance

| # | Variable | Required | In one line |
| :-: | :------- | :------: | :---------- |
| 1 | [`FLUTTER_VERSION`](#-1-flutter_version) | ✅ Always | Which Flutter SDK builds the app |
| 2 | [`GIT_PROTOCOL`](#-2-git_protocol) | ✅ Always | How to reach GitLab |
| 3 | [`GIT_HOST`](#-3-git_host) | ✅ Always | Which GitLab server |
| 4 | [`GIT_REPO_PATH`](#-4-git_repo_path) | ✅ Always | Which project on that server |
| 5 | [`GIT_BRANCH`](#-5-git_branch) | ✅ Always | Which branch to clone |
| 6 | [`GIT_CREDENTIALS_ID`](#-6-git_credentials_id) | ✅ Always | Where the git credential is kept |
| 7 | [`INFISICAL_API_URL`](#-7-infisical_api_url) | ✅ Always | Which Infisical server |
| 8 | [`INFISICAL_PROJECT_ID`](#-8-infisical_project_id) | ✅ Always | Which Infisical project |
| 9 | [`INFISICAL_ENV`](#-9-infisical_env) | ✅ Always | Which Infisical environment |
| 10 | [`INFISICAL_TOKEN_CREDENTIALS_ID`](#-10-infisical_token_credentials_id) | ✅ Always | Where the Infisical token is kept |
| 11 | [`SHOREBIRD_TOKEN_CREDENTIALS_ID`](#-11-shorebird_token_credentials_id) | ⚠️ Shorebird only | Where the Shorebird token is kept |

---

## 🎯 1. `FLUTTER_VERSION`

| | |
| :--- | :--- |
| **Example** | `3.44.6` |
| **Format** | `MAJOR.MINOR.PATCH` |
| **Required** | ✅ Always |

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

| | |
| :--- | :--- |
| **Example** | `https` |
| **Format** | `ssh` or `https` |
| **Required** | ✅ Always |

**What it is** — how the pipeline reaches GitLab.

```properties
GIT_PROTOCOL=https
```

It decides how the clone URL is built:

| Value | URL |
| :---- | :-- |
| `ssh` | `git@HOST:PATH.git` |
| `https` | `https://HOST/PATH.git` |

**Why it exists** — SSH needs port 22 open to the GitLab server. That is normal
inside the office network but often blocked from outside it, where only HTTPS gets
through. One setting lets the same pipeline run in both places.

> [!IMPORTANT]
> `GIT_CREDENTIALS_ID` must match. `ssh` needs an SSH key credential, `https` needs
> a username and password credential.

---

## 🌐 3. `GIT_HOST`

| | |
| :--- | :--- |
| **Example** | `gitlab.webelight.co.in` |
| **Format** | A hostname, no scheme and no trailing slash |
| **Required** | ✅ Always |

**What it is** — the GitLab server the project lives on.

```properties
GIT_HOST=gitlab.webelight.co.in
```

**Why it exists** — it is the same for every project on the server. Keeping it
separate from the project path means the clone URL is assembled in one place
instead of being spelled out again in every project.

---

## 📁 4. `GIT_REPO_PATH`

| | |
| :--- | :--- |
| **Example** | `webelight/ccmt/chinmaya-mission-flutter` |
| **Format** | Group and project, no leading slash, no `.git` suffix |
| **Required** | ✅ Always |

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

| | |
| :--- | :--- |
| **Example** | `main` |
| **Format** | A branch name |
| **Required** | ✅ Always |

**What it is** — the branch to clone.

```properties
GIT_BRANCH=main
```

**Why it exists** — a clone needs a named ref to be repeatable. Stating the branch
means two runs of the same configuration fetch the same thing, instead of
depending on whatever the remote's default happens to be.

---

## 🔑 6. `GIT_CREDENTIALS_ID`

| | |
| :--- | :--- |
| **Example** | `gitlab-token` |
| **Format** | A Jenkins credential ID |
| **Required** | ✅ Always |

**What it is** — the **name** of a Jenkins credential. Never the key or token
itself.

```properties
GIT_CREDENTIALS_ID=gitlab-token
```

The credential is created in Jenkins, and its kind depends on `GIT_PROTOCOL`:

| `GIT_PROTOCOL` | Kind | Username | Secret |
| :------------- | :--- | :------- | :----- |
| `ssh` | SSH Username with private key | `git` | The private key |
| `https` | Username with password | `oauth2` | The access token |

**Why it exists** — cloning a private repository needs credentials, and this file
is not a safe place for them. Storing only the name keeps the secret inside
Jenkins, where the pipeline can read it but git history and build logs never see
it.

> [!WARNING]
> Putting a raw token here instead of a credential ID is rejected by the pipeline.
> It would land in git history and in the build log in plain text.

---

## ☁️ 7. `INFISICAL_API_URL`

| | |
| :--- | :--- |
| **Example** | `https://app.infisical.com` |
| **Format** | A URL, no trailing slash |
| **Required** | ✅ Always |

**What it is** — the Infisical server the secrets are read from.

```properties
INFISICAL_API_URL=https://app.infisical.com
```

**Why it exists** — teams running a self-hosted Infisical need a different address.
Keeping it here means the pipeline works against either without a code change.

---

## 🆔 8. `INFISICAL_PROJECT_ID`

| | |
| :--- | :--- |
| **Example** | `7f3a1c2e-...` |
| **Format** | The project ID from the Infisical dashboard |
| **Required** | ✅ Always |

**What it is** — which Infisical project holds this app's secrets.

```properties
INFISICAL_PROJECT_ID=7f3a1c2e-4b5d-6789-abcd-ef0123456789
```

**Why it exists** — one Infisical account holds many projects. This is the per-app
part, so it changes for every project using this pipeline.

---

## 🏷️ 9. `INFISICAL_ENV`

| | |
| :--- | :--- |
| **Example** | `dev` |
| **Format** | An environment slug from the Infisical project |
| **Required** | ✅ Always |

**What it is** — which set of values to export, e.g. `dev`, `staging`, `prod`.

```properties
INFISICAL_ENV=dev
```

**Why it exists** — the same variable names hold different values per environment.
This decides which set lands in `.env`.

---

## 🎫 10. `INFISICAL_TOKEN_CREDENTIALS_ID`

| | |
| :--- | :--- |
| **Example** | `infisical-token` |
| **Format** | A Jenkins credential ID |
| **Required** | ✅ Always |

**What it is** — the **name** of a Jenkins credential. Never the token.

```properties
INFISICAL_TOKEN_CREDENTIALS_ID=infisical-token
```

The credential is created in Jenkins:

| Field | Value |
| :---- | :---- |
| Kind | Secret text |
| Secret | The Infisical service token |
| ID | `infisical-token` |

**Why it exists** — reading secrets requires a secret of its own. Storing only the
name keeps the token inside Jenkins, and the pipeline hands it to the CLI through
the environment rather than a command line, so it never reaches the build log.

---

## 🐦 11. `SHOREBIRD_TOKEN_CREDENTIALS_ID`

| | |
| :--- | :--- |
| **Example** | `shorebird-token` |
| **Format** | A Jenkins credential ID |
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

## ➕ Adding a variable

| Step | Action |
| :-: | :----- |
| 1 | Add a `KEY=VALUE` line to `pipeline.properties` |
| 2 | Document it here |

> [!TIP]
> No `Jenkinsfile` change is needed — the agent reads the file fresh on every
> build, so a new value takes effect on the next run.

---

<sub>Format: `KEY=VALUE`, one per line. Lines starting with `#` are comments.</sub>
