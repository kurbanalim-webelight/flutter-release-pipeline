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
| 2 | [`SHOREBIRD_TOKEN_CREDENTIALS_ID`](#-2-shorebird_token_credentials_id) | ⚠️ Shorebird only | Where the Shorebird token is kept |

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

## 🐦 2. `SHOREBIRD_TOKEN_CREDENTIALS_ID`

| | |
| :--- | :--- |
| **Example** | `shorebird-token` |
| **Format** | A Jenkins credential ID |
| **Required** | ⚠️ Only when the build runner is `Shorebird` |

**What it is** — the **name** of a Jenkins credential. Not the token.

```properties
SHOREBIRD_TOKEN_CREDENTIALS_ID=shorebird-token
```

**Why it exists** — Shorebird needs a token to publish a release. The token is a
secret; this file is not. Storing only the name keeps the secret inside Jenkins,
where the pipeline can read it but git and the build log never see it.

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
