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
| 2 | [`GIT_REPO_URL`](#-2-git_repo_url) | ✅ Always | Which GitLab repository to build |
| 3 | [`GIT_BRANCH`](#-3-git_branch) | ✅ Always | Which branch to clone |
| 4 | [`GIT_CREDENTIALS_ID`](#-4-git_credentials_id) | ✅ Always | Where the GitLab token is kept |
| 5 | [`SHOREBIRD_TOKEN_CREDENTIALS_ID`](#-5-shorebird_token_credentials_id) | ⚠️ Shorebird only | Where the Shorebird token is kept |

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

## 🌐 2. `GIT_REPO_URL`

| | |
| :--- | :--- |
| **Example** | `https://gitlab.com/your-group/your-project.git` |
| **Format** | Must start with `https://` or `git@` |
| **Required** | ✅ Always |

**What it is** — the GitLab repository the pipeline clones and builds.

```properties
GIT_REPO_URL=https://gitlab.com/your-group/your-project.git
```

**Why it exists** — this is what makes one pipeline serve many projects. Point it
at a different repository and the same `Jenkinsfile` builds a different app.

---

## 🌿 3. `GIT_BRANCH`

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

## 🔑 4. `GIT_CREDENTIALS_ID`

| | |
| :--- | :--- |
| **Example** | `gitlab-token` |
| **Format** | A Jenkins credential ID |
| **Required** | ✅ Always |

**What it is** — the **name** of a Jenkins credential. Not the token.

```properties
GIT_CREDENTIALS_ID=gitlab-token
```

The credential itself is created in Jenkins:

| Field | Value |
| :---- | :---- |
| Kind | Username with password |
| Username | `oauth2` |
| Password | The GitLab access token |
| ID | `gitlab-token` |

**Why it exists** — cloning a private repository needs a token. The token is a
secret; this file is not. Storing only the name keeps the secret inside Jenkins,
where the pipeline can read it but git and the build log never see it.

---

## 🐦 5. `SHOREBIRD_TOKEN_CREDENTIALS_ID`

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
