# ⚙️ Pipeline Variables

Everything project-specific lives in **`pipeline.properties`**.
The `Jenkinsfile` never changes between projects — only this file does. 🎯

---

## 📋 The variables

| 🔑 Key | Example | What it does |
|---|---|---|
| `FLUTTER_VERSION` | `3.35.4` | Which Flutter SDK to build with. FVM installs and pins it. |
| `SHOREBIRD_TOKEN_CREDENTIALS_ID` | `shorebird-token` | **Name** of the Jenkins credential holding the Shorebird token. Not the token. |

---

## 🔐 Why the token is not in this file

`pipeline.properties` is committed to git, and the pipeline prints every key it
loads. A real token here would end up in **two** public places. 😬

So the token goes in Jenkins, and only its **ID** goes here:

```
Manage Jenkins → Credentials → System → Global
  Kind: Secret text
  ID:   shorebird-token      ← this is what goes in pipeline.properties
  Secret: <your token>       ← this stays in Jenkins
```

---

## ➕ Adding a new variable

1. Add a line to `pipeline.properties`:
   ```properties
   PACKAGE_NAME=com.example.app
   ```
2. Done. ✅

No `Jenkinsfile` edit needed — the loader reads whatever it finds and prints it.

> 💡 Only `FLUTTER_VERSION` is validated, because it is the only one a stage uses
> so far. Others get checked as the stages that need them are added.

---

## 🚦 Rules of thumb

| ✅ Belongs here | ❌ Does not |
|---|---|
| Versions, IDs, package names, flags | Passwords, tokens, API keys |
| Anything safe to read in a build log | Anything you would not paste in Slack |

**Secrets → Jenkins Credentials. Settings → this file.** 🔒
