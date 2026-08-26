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

> 🎯 **Which role?**
> The **Access** box offers Admin, App Manager, Developer, Finance, Sales and Reports, Customer Support and Marketing. Pick **App Manager** — the smallest role that can upload builds and manage App Store versions. **Admin** works too but hands CI far more than it needs, and **Developer** cannot manage versions, so the upload step fails.

📝 Write down three things from that page:

| What | Where | Looks like |
|---|---|---|
| 🆔 **Issuer ID** | the long id at the top | `69a6de7e-1234-4c5b-8f21-9a0bcdef1234` |
| 🔑 **Key ID** | the short id in the row | `2X9R4HXF34` |
| 📄 **The `.p8` file** | press **Download** | `AuthKey_2X9R4HXF34.p8` |

> ⚠️ **One chance only**
> The `.p8` downloads exactly once. Lose it and you must revoke the key and start again. 💾 Save it somewhere safe before you close the page.

---

## 3. Write the `ExportOptions.plist`

📝 Text editor · anywhere

Copy this, replace **`teamID`** with your Apple Developer Team ID, save it as `ExportOptions.plist` and hand it to DevOps. 📤

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>destination</key>
    <string>export</string>
    <key>method</key>
    <string>app-store-connect</string>
    <key>signingStyle</key>
    <string>automatic</string>
    <key>stripSwiftSymbols</key>
    <true/>
    <key>teamID</key>
    <string>ABCDE12345</string>
    <key>thinning</key>
    <string>&lt;none&gt;</string>
</dict>
</plist>
```

> ✏️ **Two things not to touch**
> `&lt;none&gt;` is escaped on purpose — copy it exactly. And leave `method` as `app-store-connect`: any other value signs the app for something other than the App Store and TestFlight rejects the build. 🚫

---

# 🤖 Android

> 🚀 Everything you need to let the pipeline talk to the Google Play Console.

---

## 1. Enable the Google Play Android Developer API

🌐 Browser · [console.cloud.google.com](https://console.cloud.google.com)

**Google Cloud Console → select the project associated with YOUR APP → APIs & Services → Library** → search **Google Play Android Developer API** → **Enable**.

> ⚠️ **Do this first**
> Without this API enabled the service account key authenticates fine but every upload fails. 🚫

---

## 2. Create the service account

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

## 3. Create a JSON key

🔑 Same page · the account you just made

Open the new **`play-store-ci`** service account → **Keys → Add key → Create new key** → pick **JSON** → **Create**.

📄 The file downloads on its own and contains fields such as:

`type` · `project_id` · `private_key` · `client_email` · `client_id` …

> ⚠️ **One chance only**
> Google never shows the key again. Lose the file and you must create a new key. 💾 Save it somewhere safe.

---

## 4. Give the service account access in Play Console

🌐 Browser · [play.google.com/console](https://play.google.com/console)

**Setup → API access** — the Cloud project must be linked and `play-store-ci` listed.

**Users and permissions → Invite new user** — paste the **`client_email`** from the JSON, not your own email. Under **App permissions** pick YOUR APP and tick:

| Tick | Permission |
|---|---|
| ✅ | **Release apps to testing tracks** |
| ✅ | **View app information (read-only)** |

**Apply**, then **Save changes**. 🔐

> 👑 **Owner or admin only**
> Nobody else can grant this.

---

# 🤝 Hand the credentials to the DevOps team

> 📦 One handover for both platforms. Share these so DevOps can load them into the pipeline.

---

## 🍎 iOS — all four are mandatory

| What | File / value | Secret? |
|---|---|---|
| 📄 **App Store Connect API key** | `AuthKey_<KeyID>.p8` from [iOS step 2](#2-create-an-app-store-connect-api-key) | 🔴 yes |
| 🔑 **Key ID** | `2X9R4HXF34` — from [iOS step 2](#2-create-an-app-store-connect-api-key), also in the `.p8` filename | 🟢 no |
| 🆔 **Issuer ID** | `69a6de7e-1234-4c5b-8f21-9a0bcdef1234` — from [iOS step 2](#2-create-an-app-store-connect-api-key) | 🟢 no |
| 📄 **`ExportOptions.plist`** | the plist from [iOS step 3](#3-write-the-exportoptionsplist) — one Team ID away from the template | 🟢 no |

> ❗ **The `.p8` on its own is not enough**
> The pipeline signs every Apple API call with all three: the Key ID, the Issuer ID and the `.p8`. Miss the Issuer ID and every build fails to authenticate. The Issuer ID cannot be read off the file — it only lives on that App Store Connect page.

> 📄 **What the `ExportOptions.plist` is for**
> The build step hands it to Flutter as `flutter build ipa --export-options-plist=<path>`, replacing the plain `--export-method app-store` the pipeline uses today. That pins the export to your team and your signing style instead of leaving it to Flutter's defaults.

---

## 🤖 Android

| What | File / value | Secret? |
|---|---|---|
| 📄 **Service account key** | the JSON downloaded in [Android step 3](#3-create-a-json-key) | 🔴 yes |
| 📧 **`client_email`** | read it out of that JSON, e.g. `play-store-ci@your-project.iam.gserviceaccount.com` | 🟢 no |

> 📧 **What the `client_email` is for**
> It goes into the **Google Play Console** to grant YOUR APP access for CI/CD, as in [Android step 4](#4-give-the-service-account-access-in-play-console). Without that grant the key exists but can do nothing.

---

## 🔐 How to send them

> 🔒 **Secrets go through a password manager or a secrets vault — never email, Slack, or a git commit.**
> The `.p8` signs your App Store identity and the JSON can publish to your Play Store listing. The Key ID, Issuer ID, `client_email` and `ExportOptions.plist` are just identifiers and settings, so those are fine to paste anywhere — the plist can even be committed to the repo.

---
