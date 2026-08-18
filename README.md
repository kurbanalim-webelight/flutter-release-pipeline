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
| Require | `FLUTTER_VERSION`, `GIT_REPO_URL`, `GIT_BRANCH`, `GIT_CREDENTIALS_ID` |
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
| 1 | Clone `GIT_REPO_URL` at branch `GIT_BRANCH`, authenticating with `GIT_CREDENTIALS_ID` |
| 2 | Shallow clone (`depth 1`, no tags) so only the commit being built is fetched |
| 3 | Print the commit that was checked out |
