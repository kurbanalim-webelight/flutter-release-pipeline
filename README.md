# 🚀 Flutter Release Pipeline

> The steps in the [`Jenkinsfile`](Jenkinsfile), and what each one does.

---

## 📑 Table of contents

| # | Step |
| :-: | :--- |
| 1 | [Validate Inputs](#-1-validate-inputs) |
| 2 | [Load Configuration](#-2-load-configuration) |
| 3 | [Setup Flutter](#-3-setup-flutter) |
| 4 | [Verify Flutter Version](#-4-verify-flutter-version) |

---

## ✅ 1. Validate Inputs

Checks the values the user entered before any work starts.

| Check | Rule |
| :---- | :--- |
| `BUILD_VERSION` | Present, and matches `MAJOR.MINOR.PATCH` |
| `APP_BUILD_NUMBER` | Present, digits only, and `≥ 1` |

Then names the run so the build history is readable:

```text
#17 Android/Flutter 1.0.0+32
```

---

## 📄 2. Load Configuration

Reads [`pipeline.properties`](pipeline.properties) and makes its values available
to the rest of the pipeline.

| Action | Detail |
| :----- | :----- |
| Parse | `KEY=VALUE` per line; `#` and `!` start a comment |
| Require | `FLUTTER_VERSION` must exist and look like a version |
| Export | `FLUTTER_VERSION` becomes an environment variable for later steps |

---

## 🐦 3. Setup Flutter

Prepares the toolchain, installing whatever the agent is missing.

| # | Action |
| :-: | :----- |
| 1 | If `fvm` is absent, install it with Homebrew |
| 2 | `fvm install "$FLUTTER_VERSION" --setup` — downloads the SDK if not cached |
| 3 | `fvm use "$FLUTTER_VERSION" --force` — pins this workspace to that SDK |

---

## 🔍 4. Verify Flutter Version

Confirms the SDK that is actually active is the one that was asked for. The
version is read from `fvm flutter --version` and compared as an exact string.

```text
expected: 3.44.6
actual:   3.44.6
Flutter version matches
```
