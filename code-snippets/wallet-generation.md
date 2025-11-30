# Wallet Generation - Critical Security Code

> **Note**: These are EXCERPTS from production code for security verification.
> Not the complete implementation. Not for reuse.

---

## 🔑 The Critical Line: Where Your Wallet is Generated

**File**: `vanity-worker.js` (Web Worker)
**Line 78** (approximate):

```javascript
// Generate a new random keypair
const keypair = Keypair.generate();
const publicKey = keypair.publicKey.toBase58();
```

### What This Means:

- `Keypair.generate()` is from `@solana/web3.js` (official Solana library)
- Uses browser's `crypto.getRandomValues()` (cryptographically secure)
- **Happens in Web Worker** (isolated browser thread)
- **NO network communication** (you can verify in DevTools)

---

## 📤 How Private Keys Are Handled

**Lines 88-98** (approximate):

```javascript
if (matchResult && matchResult.matched) {
  // Found a match! Send it back to the main thread
  self.postMessage({
    type: 'found',
    workerId: workerId,
    result: {
      publicKey: publicKey,
      secretKey: Array.from(keypair.secretKey), // ← Private key
      attempts: attempts,
      pattern: matchResult.pattern,
    }
  });
  // Don't stop - keep searching for more results
}
```

### What This Means:

- `secretKey` is sent via `postMessage()` to main browser thread
- **Still in your browser** - not sent to server
- Main thread displays it to YOU
- YOU decide whether to save it or not

---

## 🔒 Zero Network Communication

**Proof Point 1**: Worker Imports

```javascript
// Line 13 (approximate)
importScripts('https://cdn.jsdelivr.net/npm/@solana/web3.js@1.95.8/lib/index.iife.min.js');
```

- Only external call: Loading Solana library from CDN
- Happens ONCE when worker starts
- **Not during generation** (verify in Network tab)

**Proof Point 2**: No Fetch/XHR Calls

Search the entire worker file for:
- `fetch(` - ❌ None
- `XMLHttpRequest` - ❌ None
- `axios` - ❌ None
- `$.ajax` - ❌ None

**The worker has NO code to send data to a server.**

---

## ✅ How to Verify This Yourself

### Method 1: Browser DevTools

1. Go to [vwg.rip](https://vwg.rip)
2. F12 → Network tab
3. Filter: "Fetch/XHR"
4. Start wallet generation
5. **Result**: EMPTY (no requests)

### Method 2: View Full Worker Source

```
view-source:https://vwg.rip/vanity-worker.js
```

- Find line 78: `const keypair = Keypair.generate();`
- Confirm it matches what we show here
- Search for network calls: None exist

### Method 3: Run Offline

1. Visit vwg.rip
2. Disconnect internet
3. Generate wallet
4. **It still works** (because it's 100% client-side)

---

## 🛠️ Technical Details

### Entropy Source

```javascript
// What Keypair.generate() does internally:
const randomBytes = crypto.getRandomValues(new Uint8Array(32));
// Then derives Ed25519 keypair from these bytes
```

- `crypto.getRandomValues()` is browser's built-in secure RNG
- Same quality as hardware wallets
- CSPRNG (Cryptographically Secure Pseudo-Random Number Generator)

### Private Key Format

```javascript
secretKey: Uint8Array(64)
// [0-31]:  Private key bytes (32 bytes)
// [32-63]: Public key bytes (32 bytes)
```

- Standard Ed25519 keypair format
- Compatible with all Solana wallets
- Exported as Base58 string for display

---

## 🔍 Compare with Production

**Verify these snippets are real:**

1. Visit: `https://vwg.rip/vanity-worker.js`
2. Search for: `const keypair = Keypair.generate();`
3. Should appear around line 78
4. Context matches what we show above

**Production File Hash** (for verification):
```
SHA256: [Calculate with: curl https://vwg.rip/vanity-worker.js | sha256sum]
```

---

## ❓ Common Questions

### Q: Could you intercept the private key in postMessage?

**A**: Technically, if we modified the main thread code. But:
1. You can view main thread code too (view-source)
2. No network calls exist there either (verify in DevTools)
3. This entire repo exists for you to verify that

### Q: What about after the wallet is displayed?

**A**:
- We offer OPTIONAL saving to database (for leaderboard feature)
- You see a modal: "Save this wallet?"
- If you click "No" → not saved anywhere
- If you click "Yes" → only PUBLIC key saved (not private key)

### Q: How do I know @solana/web3.js isn't compromised?

**A**:
- It's the official Solana Foundation library
- Used by Phantom, Solflare, and all major Solana wallets
- Loaded from JSDelivr CDN (not our server)
- You can verify CDN integrity

---

**Remember**: These snippets prove our security model.
**Full code remains proprietary** - use vwg.rip instead of building your own.
