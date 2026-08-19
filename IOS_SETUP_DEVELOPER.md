# 🍎 iOS Setup Guide

> 🚀 Everything you need to get iOS release builds signed and shipping.

---

## 1. Check if a distribution certificate already exists

🔍 First, look before you create — duplicate certs cause pain later.

```bash
security find-identity -v -p codesigning | grep "Apple Distribution"
```

✅ **Something printed?** You're good — the cert is already on this Mac.

⚠️ **Nothing printed?** Either a teammate has it, or nobody has made one yet. **Ask first.** 🙋

🆕 If truly nobody has one:

> Xcode → ⚙️ Settings → 👤 Accounts → pick the team → 🔐 Manage Certificates → **➕** → **Apple Distribution**

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

## 3. Export the certificate as a `.p12` file

🔐 Keychain Access app · Development Mac

> 🖱️ This is the last time you use a window with buttons for signing.

1. Open **Keychain Access**
2. Click **My Certificates** in the sidebar
3. Find `Apple Distribution: Your Company (TEAMID)`
4. Click the small **▸** beside it — a 🗝️ private key must appear underneath. **No key means this Mac cannot export it**
5. Right click → **Export**
6. Format: **Personal Information Exchange (.p12)**
7. Save as `dist.p12` and set a 🔒 password

---

## 4. Create the provisioning profile in Apple's portal

🌐 Browser · [developer.apple.com](https://developer.apple.com)

1. **Certificates, Identifiers & Profiles → Profiles**
2. Press **➕**
3. Choose **App Store Connect** under **Distribution**
4. Pick the **App ID** — the bundle id of your app
5. Pick the certificate from [step 3](#3-export-the-certificate-as-a-p12-file)
6. Name it clearly, for example `CM App AppStore`, and **generate** it

> 🚫 **Do not download it**
> The profile only needs to exist. From now on the build pulls it from Apple every single time, so it can never go stale on you. ♻️

---

## 5. Hand the credentials to the DevOps team

🤝 Share all five with DevOps so they can load them into the pipeline:

| What | File / value | Secret? |
|---|---|---|
| 📄 **App Store Connect API key** | `AuthKey_<KeyID>.p8` from [step 2](#2-create-an-app-store-connect-api-key) | 🔴 yes |
| 🔐 **Distribution certificate** | `dist.p12` from [step 3](#3-export-the-certificate-as-a-p12-file) | 🔴 yes |
| 🔒 **`.p12` password** | the password you set when exporting | 🔴 yes |
| 🔑 **Key ID** | `2X9R4HXF34` — from [step 2](#2-create-an-app-store-connect-api-key), also in the `.p8` filename | 🟢 no |
| 🆔 **Issuer ID** | `69a6de7e-1234-4c5b-8f21-9a0bcdef1234` — from [step 2](#2-create-an-app-store-connect-api-key) | 🟢 no |

> ⚠️ **The `.p8` on its own is not enough**
> The pipeline signs every Apple API call with all three: the Key ID, the Issuer ID and the `.p8`. Miss the Issuer ID and every build fails to authenticate. The Issuer ID cannot be read off the file — it only lives on that App Store Connect page.

> 🔐 **Send the secrets privately**
> Use a password manager or a secrets vault — never email, Slack, or a git commit. These are signing keys for your App Store identity. The Key ID and Issuer ID are just identifiers, so those two are fine to paste anywhere.

---
