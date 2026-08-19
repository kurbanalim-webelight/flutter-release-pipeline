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
