# 🚀 Flutter Release Pipeline

> The steps in the [`Jenkinsfile`](Jenkinsfile), and what each one does.

---

## 📑 Table of contents

| # | Step |
| :-: | :--- |
| 1 | [Validate & Load Configuration](#-1-validate--load-configuration) |
| 2 | [Setup Flutter](#-2-setup-flutter) |
| 3 | [Clone Repository](#-3-clone-repository) |

---

## ✅ 1. Validate & Load Configuration

Checks the user's input, then reads [`pipeline.properties`](pipeline.properties)
and makes its values available to the rest of the pipeline.

| Check | Rule |
| :---- | :--- |
| `BUILD_VERSION` | Present, and matches `MAJOR.MINOR.PATCH` |
| `APP_BUILD_NUMBER` | Present, digits only, and `≥ 1` |

Names the run so the build history is readable:

```text
#17 Android/Flutter 1.0.0+32
```

| Action | Detail |
| :----- | :----- |
| Parse | `KEY=VALUE` per line; `#` and `!` start a comment |
| Require | `FLUTTER_VERSION`, `GIT_PROTOCOL`, `GIT_HOST`, `GIT_REPO_PATH`, `GIT_BRANCH`, `GIT_CREDENTIALS_ID` |
| Build | The clone URL from `GIT_PROTOCOL`, `GIT_HOST` and `GIT_REPO_PATH` |
| Reject | A `GIT_CREDENTIALS_ID` that looks like a raw token instead of an ID |
| Export | Each value becomes an environment variable for later steps |

---

## 🐦 2. Setup Flutter

Prepares the toolchain, installs the SDK, and confirms the right one is active.

| # | Action |
| :-: | :----- |
| 1 | If `fvm` is absent, install it with Homebrew |
| 2 | `fvm install "$FLUTTER_VERSION" --setup` — downloads the SDK if not cached |
| 3 | `fvm use "$FLUTTER_VERSION" --force` — pins the workspace to that SDK |
| 4 | Compare the active version against `FLUTTER_VERSION` as an exact string |

```text
expected: 3.44.6
actual:   3.44.6
Flutter version matches
```

---

## 📥 3. Clone Repository

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
