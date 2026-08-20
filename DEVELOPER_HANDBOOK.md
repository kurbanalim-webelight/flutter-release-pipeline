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

## 3. Hand the credentials to the DevOps team

🤝 Share all three with DevOps so they can load them into the pipeline:

| What | File / value | Secret? |
|---|---|---|
| 📄 **App Store Connect API key** | `AuthKey_<KeyID>.p8` from [step 2](#2-create-an-app-store-connect-api-key) | 🔴 yes |
| 🔑 **Key ID** | `2X9R4HXF34` — from [step 2](#2-create-an-app-store-connect-api-key), also in the `.p8` filename | 🟢 no |
| 🆔 **Issuer ID** | `69a6de7e-1234-4c5b-8f21-9a0bcdef1234` — from [step 2](#2-create-an-app-store-connect-api-key) | 🟢 no |

> ❗ **All three are mandatory**
> The pipeline signs every Apple API call with all three: the Key ID, the Issuer ID and the `.p8`. Miss the Issuer ID and every build fails to authenticate. The Issuer ID cannot be read off the file — it only lives on that App Store Connect page.

> 🔐 **Send the `.p8` privately**
> Use a password manager or a secrets vault — never email, Slack, or a git commit. It is a signing key for your App Store identity. The Key ID and Issuer ID are just identifiers, so those two are fine to paste anywhere.

---
