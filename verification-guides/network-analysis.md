# Network Analysis Guide (Advanced)

This guide is for advanced users who want to perform deep packet inspection to verify no private keys are transmitted.

---

## ⚠️ Prerequisites

This guide assumes you have:
- Basic networking knowledge
- Familiarity with packet capture tools
- Understanding of HTTP/HTTPS protocols

**If you're a regular user**, the [DevTools Check Guide](./devtools-check.md) is sufficient. This is for security researchers.

---

## 🛠️ Method 1: Wireshark (Deep Packet Inspection)

### Why Wireshark?

Wireshark captures ALL network traffic at the packet level, even before it reaches the browser. This proves no background data exfiltration.

---

### Step 1: Install Wireshark

Download from: `https://www.wireshark.org/download.html`

**Platforms**:
- Windows, Mac, Linux all supported

---

### Step 2: Start Packet Capture

1. Open Wireshark
2. Select your network interface (e.g., Wi-Fi or Ethernet)
3. Click the blue shark fin icon to start capture

---

### Step 3: Filter for vwg.rip Traffic

In the filter bar, enter:
```
http.host == "vwg.rip" || tls.handshake.extensions_server_name == "vwg.rip"
```

This shows only traffic to/from vwg.rip.

---

### Step 4: Generate Wallet

1. Go to `https://vwg.rip`
2. Generate a vanity wallet
3. Wait for results

---

### Step 5: Analyze Captured Traffic

**Expected Traffic**:

1. **Initial Page Load**:
   - `GET /` (HTML page)
   - `GET /styles.css` (stylesheets)
   - `GET /main.js` (JavaScript bundle)
   - `GET /vanity-worker.js` (worker script)

2. **CDN Requests** (to jsdelivr.net, NOT vwg.rip):
   - `GET https://cdn.jsdelivr.net/npm/@solana/web3.js@1.95.8/...`

3. **During Wallet Generation**:
   - **ZERO additional requests to vwg.rip** ✅

4. **If You Click "Save to Leaderboard"**:
   - `POST /api/wallets` (only public key in payload)

---

### Step 6: Inspect Request Payloads

Right-click any POST request → "Follow" → "HTTP Stream"

**Look for**:
- ❌ `privateKey`
- ❌ `secretKey`
- ❌ Base58-encoded private keys (64+ character strings)
- ❌ Byte arrays with 64 elements

**Should only see**:
- ✅ `publicKey` (44 characters, Base58)
- ✅ `pattern` (e.g., "Ape...")
- ✅ `attempts` (number)

---

### Step 7: Export for Analysis

File → Export Objects → HTTP

This saves all HTTP traffic to files for manual inspection.

---

## 🛠️ Method 2: mitmproxy (HTTPS Interception)

### Why mitmproxy?

Allows inspecting HTTPS traffic (encrypted) by acting as a man-in-the-middle on YOUR machine.

**Note**: This requires installing a root certificate on your system. Only do this if you understand the security implications.

---

### Step 1: Install mitmproxy

```bash
# macOS (Homebrew)
brew install mitmproxy

# Linux (apt)
sudo apt install mitmproxy

# Or using pip
pip install mitmproxy
```

---

### Step 2: Start mitmproxy

```bash
mitmproxy
```

Or for web interface:
```bash
mitmweb
```

Runs on `http://localhost:8081` by default.

---

### Step 3: Configure Browser Proxy

**Firefox**:
1. Settings → Network Settings
2. Manual proxy: `localhost:8080`
3. Check "Use this proxy for HTTPS"

**Chrome** (use FoxyProxy extension):
1. Install FoxyProxy
2. Add proxy: `localhost:8080` (HTTP and HTTPS)

---

### Step 4: Install mitmproxy Certificate

1. Browse to `http://mitm.it`
2. Download certificate for your OS
3. Install and trust the certificate

**Windows**: Double-click → Install → Trusted Root Certification Authorities
**Mac**: Keychain Access → Import → Trust
**Linux**: Copy to `/usr/local/share/ca-certificates/` and run `update-ca-certificates`

---

### Step 5: Monitor Traffic

1. Go to `https://vwg.rip`
2. Generate wallet
3. Watch mitmproxy console/web interface

**What you'll see**:
- All HTTP/HTTPS requests in plaintext (decrypted)
- Request/response bodies
- Headers

**What to verify**:
- No private keys in request bodies
- No unexpected POST requests during generation

---

### Step 6: Filter Traffic

In mitmproxy console, press `f` and enter:
```
~d vwg.rip
```

Shows only traffic to vwg.rip domain.

---

### Step 7: Inspect Request Details

- Press Enter on any request to view details
- Tab through Request → Response → Detail
- Press `q` to go back

---

## 🛠️ Method 3: Browser Network Export (HAR)

### Step 1: Open DevTools

F12 → Network tab

---

### Step 2: Preserve Log

Check "Preserve log" option (so requests aren't cleared on navigation)

---

### Step 3: Generate Wallet

Generate a wallet while Network tab is open.

---

### Step 4: Export HAR File

Right-click in Network tab → "Save all as HAR with content"

This saves a JSON file with ALL network activity.

---

### Step 5: Analyze HAR File

Open the `.har` file in a text editor.

**Search for** (Ctrl+F):
- `secretKey`
- `privateKey`
- `keypair`

**Expected result**: NOT FOUND in any request payloads ✅

**Example of what you SHOULD see**:
```json
{
  "request": {
    "url": "https://vwg.rip/api/wallets",
    "method": "POST",
    "postData": {
      "text": "{\"publicKey\":\"Ape1a2b3...\",\"pattern\":\"Ape...\"}"
    }
  }
}
```

Notice: Only `publicKey`, not `privateKey`.

---

## 🛠️ Method 4: Command-Line with curl

### Test the API Endpoint Directly

See what data the API accepts:

```bash
curl -X POST https://vwg.rip/api/wallets \
  -H "Content-Type: application/json" \
  -d '{
    "publicKey": "TestPublicKey123",
    "pattern": "Test...",
    "attempts": 1000,
    "matchLength": 4
  }'
```

**What to verify**:
- API accepts only these fields
- No `privateKey` field accepted
- Server returns error if you try to send unexpected fields

---

### Try Sending a Private Key (Should Fail)

```bash
curl -X POST https://vwg.rip/api/wallets \
  -H "Content-Type: application/json" \
  -d '{
    "publicKey": "TestKey",
    "privateKey": "ThisShouldNotBeAccepted"
  }'
```

**Expected Result**:
- Server ignores `privateKey` field (doesn't save it)
- Or returns validation error
- **Proves API doesn't even handle private keys**

---

## 🔍 What to Look For

### ✅ Normal Traffic Pattern:

```
1. GET https://vwg.rip/
   → Status: 200 (page loads)

2. GET https://vwg.rip/vanity-worker.js
   → Status: 200 (worker loads)

3. GET https://cdn.jsdelivr.net/npm/@solana/web3.js@...
   → Status: 200 (library loads)

4. [10-60 seconds of SILENCE - wallet generation]

5. (Optional) POST https://vwg.rip/api/wallets
   → Body: { "publicKey": "...", ... }
   → Status: 200/201
```

**Key observation**: Gap between step 3 and 5 with NO network activity.

---

### 🚩 Suspicious Traffic (should NOT exist):

```
❌ POST https://vwg.rip/api/collect-keys
   Body: { "privateKey": "..." }

❌ POST https://unknown-domain.com/steal
   Body: { "secretKey": [...] }

❌ WebSocket connection during generation
   wss://vwg.rip/keys

❌ Requests to IP addresses instead of domains
   POST http://192.168.1.100/keys

❌ Base64-encoded data in unusual requests
   POST /api/log
   Body: "cHJpdmF0ZUtleT1bMTIzLDQ1Niw..." (looks obfuscated)
```

---

## 📊 Packet Analysis Checklist

After running packet capture:

- [ ] Zero requests during wallet generation (between worker load and display)
- [ ] No POST requests to unknown domains
- [ ] Save request (if used) contains only public key
- [ ] No WebSocket connections established
- [ ] No requests to IP addresses (only domain names)
- [ ] No obfuscated/base64 data in unexpected places
- [ ] HAR file search for "secret" and "private" returns no results

---

## 🛡️ Advanced: TLS Certificate Pinning Check

### Why Check This?

If the site uses certificate pinning, it prevents man-in-the-middle attacks (even with valid certificates).

### How to Check:

1. View page source (Ctrl+U)
2. Search for `Content-Security-Policy` meta tag
3. Look for pinning directives

**Example** (good):
```html
<meta http-equiv="Content-Security-Policy"
      content="upgrade-insecure-requests; block-all-mixed-content">
```

---

## ❓ FAQ

### Q: I see requests to Google/Vercel/etc - is that suspicious?

**A**: Depends on the content. Common legitimate third-party requests:
- **Google Analytics**: `google-analytics.com` (usage tracking)
- **Vercel Analytics**: `vercel-insights.com` (hosting analytics)
- **CDNs**: `jsdelivr.net`, `unpkg.com` (library hosting)

**Check**: These should NOT contain wallet data. Inspect payloads.

### Q: What if I see encrypted WebSocket traffic?

**A**: WebSockets (`wss://`) are suspicious during wallet generation. Investigate:
1. What domain is it connecting to?
2. When does it connect (before/during/after generation)?
3. What data is being sent? (Use mitmproxy to decrypt)

**Red flag** if:
- Connects during generation
- Sends data matching keypair timing
- Domain is not vwg.rip

### Q: How do I know the traffic isn't encrypted with a custom scheme?

**A**: You can't hide network requests themselves (Wireshark sees them). Even with custom encryption:
- Wireshark shows the request happens
- Packet size reveals data volume
- Timing analysis shows correlation

**If NO requests appear during generation**, custom encryption is impossible (can't encrypt what doesn't exist).

---

## 🎯 Summary for Security Researchers

**Goal**: Prove that private keys never leave the browser.

**Evidence to collect**:
1. Packet capture showing zero requests during generation
2. HAR file with no sensitive data in payloads
3. API endpoint testing showing private keys are rejected
4. Worker source code with no network functions

**If all evidence is clean**, the security claim is validated. ✅

---

**Related Guides**:
- [DevTools Check Guide](./devtools-check.md) - Simpler method for regular users
- [Source Inspection Guide](./source-inspection.md) - Verify code itself
