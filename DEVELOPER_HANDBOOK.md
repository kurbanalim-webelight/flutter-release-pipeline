# 📘 Developer Handbook

---

# 🍎 iOS

> 🚀 Everything you need to get iOS release builds signed and shipping.

---

## 1. Sign in to Xcode on the pipeline runner Mac

💻 Xcode · pipeline runner Mac

Log in with your Apple account so Xcode can pull down and install the signing pieces on its own:

> Xcode → ⚙️ Settings → 👤 Accounts → **➕** → sign in → pick the team

✅ Certificates and provisioning profiles now install **automatically** on this Mac — nothing to export, nothing to download. ♻️

> 🖥️ **It has to be the runner Mac**
> Signing happens where the build happens. Logging in on your own laptop does nothing for the pipeline.

---

## 2. Create an App Store Connect API key

🌐 Browser · [appstoreconnect.apple.com](https://appstoreconnect.apple.com)

Go to **Users and Access → Integrations → App Store Connect API** and press **➕**. Name it `CI Pipeline` and set the role to **App Manager**.

📝 Write down three things from that page:

| What | Where | Looks like |
|---|---|---|
| 🆔 **Issuer ID** | the long id at the top | `69a6de7e-1234-4c5b-8f21-9a0bcdef1234` |
| 🔑 **Key ID** | the short id in the row | `2X9R4HXF34` |
| 📄 **The `.p8` file** | press **Download** | `AuthKey_2X9R4HXF34.p8` |

> ⚠️ **One chance only**
> The `.p8` downloads exactly once. Lose it and you must revoke the key and start again. 💾 Save it somewhere safe before you close the page.

---

# 🤖 Android

> 🚀 Everything you need to let the pipeline talk to the Google Play Console.

---

## 1. Create the service account

🌐 Browser · [console.cloud.google.com](https://console.cloud.google.com)

**Google Cloud Console → select the project associated with YOUR APP → IAM & Admin → Service Accounts → Create service account**

📝 Fill it in exactly like this:

| Field | Value |
|---|---|
| 🏷️ **Name** | `play-store-ci` |
| 🆔 **ID** | `play-store-ci` |
| 💬 **Description** | `Service account for Google Play Console API access` |

Leave **all** optional Google Cloud permissions and roles **empty** → **Continue** → **Done**.

> 🚫 **No roles here**
> Access is granted later inside the Play Console, not in Google Cloud. Adding roles here does nothing for the pipeline. 🙅

---

## 2. Create a JSON key

🔑 Same page · the account you just made

Open the new **`play-store-ci`** service account → **Keys → Add key → Create new key** → pick **JSON** → **Create**.

📄 The file downloads on its own and contains fields such as:

`type` · `project_id` · `private_key` · `client_email` · `client_id` …

> ⚠️ **One chance only**
> Google never shows the key again. Lose the file and you must create a new key. 💾 Save it somewhere safe.

---

# 🤝 Hand the credentials to the DevOps team

> 📦 One handover for both platforms. Share these so DevOps can load them into the pipeline.

---

## 🍎 iOS — all three are mandatory

| What | File / value | Secret? |
|---|---|---|
| 📄 **App Store Connect API key** | `AuthKey_<KeyID>.p8` from [iOS step 2](#2-create-an-app-store-connect-api-key) | 🔴 yes |
| 🔑 **Key ID** | `2X9R4HXF34` — from [iOS step 2](#2-create-an-app-store-connect-api-key), also in the `.p8` filename | 🟢 no |
| 🆔 **Issuer ID** | `69a6de7e-1234-4c5b-8f21-9a0bcdef1234` — from [iOS step 2](#2-create-an-app-store-connect-api-key) | 🟢 no |

> ❗ **The `.p8` on its own is not enough**
> The pipeline signs every Apple API call with all three: the Key ID, the Issuer ID and the `.p8`. Miss the Issuer ID and every build fails to authenticate. The Issuer ID cannot be read off the file — it only lives on that App Store Connect page.

---

## 🤖 Android

| What | File / value | Secret? |
|---|---|---|
| 📄 **Service account key** | the JSON downloaded in [Android step 2](#2-create-a-json-key) | 🔴 yes |
| 📧 **`client_email`** | read it out of that JSON, e.g. `play-store-ci@your-project.iam.gserviceaccount.com` | 🟢 no |

> 📧 **What the `client_email` is for**
> It goes into the **Google Play Console** to grant YOUR APP access for CI/CD. Without that grant the key exists but can do nothing.

---

## 🔐 How to send them

> 🔒 **Secrets go through a password manager or a secrets vault — never email, Slack, or a git commit.**
> The `.p8` signs your App Store identity and the JSON can publish to your Play Store listing. The Key ID, Issuer ID and `client_email` are just identifiers, so those are fine to paste anywhere.

---
