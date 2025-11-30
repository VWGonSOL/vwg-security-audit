# VWG Security Transparency

**Verification repository for [vwg.rip](https://vwg.rip) - Vanity Wallet Generator**

> ⚠️ **This repository is for security verification ONLY**
> Code snippets are provided to prove we don't store private keys.
> Not for reuse or redistribution.

---

## 🔒 What We Prove

1. **Private keys are generated 100% in your browser**
   - No server communication during generation
   - Keys never leave your device
   - Verifiable in browser DevTools

2. **Telegram login is legitimate**
   - No private keys involved in authentication
   - Standard OAuth-style flow
   - Only stores: Telegram ID, username (for rankings)

---

## ✅ How to Verify (3 Minutes)

### Step 1: Browser DevTools Test

1. Visit [vwg.rip](https://vwg.rip)
2. Open DevTools (F12) → **Network** tab
3. Click "Generate Now" and start generation
4. **Observe: ZERO network requests during generation**

✅ This proves wallet generation happens entirely in your browser.

### Step 2: View Source Code

The wallet generation worker is publicly accessible:
```
https://vwg.rip/vanity-worker.js
```

Right-click → View Page Source to see the exact code running.

### Step 3: Check Critical Lines

See [code-snippets/wallet-generation.md](./code-snippets/wallet-generation.md) for the key security-critical sections with explanations.

---

## 📊 What We DON'T Store

| Data | Server Storage | Why |
|------|---------------|-----|
| Private Keys | ❌ NEVER | Generated in browser only |
| Seed Phrases | ❌ NEVER | Derived client-side from private key |
| Passwords | ❌ NEVER | No passwords used |
| Public Keys | ✅ Optional | Only if you save wallet for rankings |
| Telegram ID | ✅ Yes | For authentication only |

---

## 🔍 Code Snippets

We provide **key security-critical code sections** (not full implementation):

### [Wallet Generation Snippets](./code-snippets/wallet-generation.md)
- Line 78: Where keypairs are generated (`Keypair.generate()`)
- Line 88-98: How private keys are handled
- Proof: No network calls in worker

### [Telegram Auth Flow](./code-snippets/telegram-auth.md)
- How one-time tokens work
- What data we store (spoiler: no private keys)
- OAuth-style security model

---

## 📖 Detailed Verification Guides

Step-by-step instructions for security researchers:

1. [Browser DevTools Verification](./verification-guides/devtools-check.md)
2. [Source Code Inspection](./verification-guides/source-inspection.md)
3. [Network Traffic Analysis](./verification-guides/network-analysis.md)

---

## 🛡️ Security Architecture

```
User Browser
    │
    ├─ Web Worker (isolated thread)
    │  └─ Keypair.generate() ◄── @solana/web3.js
    │     └─ Private key stays in RAM
    │
    └─ Main Thread
       └─ Display private key to user
          └─ User chooses to save or not

❌ NO server storage
❌ NO external API calls
❌ NO database writes (for private keys)
```

---

## 🤔 FAQ

### Why not show full code?

**A**: We show the security-critical parts to prove safety. Full code would enable copycats, hurting our business while not adding security value.

### Can I trust the code snippets are real?

**A**: Yes! Compare with production:
1. View source: `view-source:https://vwg.rip/vanity-worker.js`
2. Match line numbers with our snippets
3. Code is identical

### What about Telegram auth?

**A**: See [telegram-auth flow](./code-snippets/telegram-auth.md). No private keys are involved. We only store your Telegram ID for login.

### Could you change the code after I verify?

**A**:
- This repo is timestamped on GitHub
- Check `Last Updated` date below
- Compare production file hash with our documented hash
- Community can monitor for changes

---

## 📞 Security Contact

Found a vulnerability? **Please report responsibly:**

- Twitter/X: [@walgenrip](https://x.com/walgenrip)
- GitHub Issues: For non-critical questions only

**Please DO NOT** open public issues for critical security vulnerabilities. Contact us privately via DM first.

---

## 📅 Transparency Log

| Date | Update |
|------|--------|
| 2025-01-30 | Initial transparency repository created |

**Production Code Hash** (vanity-worker.js):
`SHA256: [will be calculated after deployment]`

---

## ⚖️ Legal

This repository contains **code snippets for security verification purposes ONLY**.

- ❌ NOT licensed for commercial or personal reuse
- ❌ NOT open source
- ✅ Provided for transparency and security audit only
- ✅ Full application remains proprietary to VWG

**Use vwg.rip** - Don't copy our work.

---

**Last Updated**: 2025-01-30
**Repository Purpose**: Security transparency and verification
**Production Site**: [vwg.rip](https://vwg.rip)
