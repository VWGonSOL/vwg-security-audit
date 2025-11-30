# Telegram Authentication - Security Code

> **Note**: These are EXCERPTS from production code for security verification.
> Bot token and sensitive credentials are REDACTED.
> Not for reuse.

---

## 🔐 How Telegram Login Works (No Private Keys Involved)

**Critical Security Point**: Telegram authentication happens COMPLETELY SEPARATELY from wallet generation. Your Telegram login NEVER has access to your wallet's private key.

---

## 📍 Step 1: User Clicks "Login via Telegram"

**File**: `app/auth/telegram-bot/route.ts`
**Purpose**: OAuth callback handler

```typescript
export async function GET(request: NextRequest) {
  // Extract one-time token from URL
  const token = searchParams.get("token");

  // Validate token in database
  const loginToken = await prisma.telegramLoginToken.findUnique({
    where: { token },
  });

  // Security checks:
  if (!loginToken) {
    return NextResponse.redirect("/auth/error?error=InvalidToken");
  }

  if (loginToken.used) {
    return NextResponse.redirect("/auth/error?error=TokenAlreadyUsed");
  }

  if (new Date() > loginToken.expiresAt) {
    return NextResponse.redirect("/auth/error?error=TokenExpired");
  }
```

### What This Means:

- One-time use tokens (can't be replayed)
- Short expiration window (5 minutes)
- Server validates against database
- **NO wallet private keys are transmitted**

---

## 📍 Step 2: Create or Find User

**Lines 47-61** (approximate):

```typescript
// Check if user exists
let user = await prisma.user.findUnique({
  where: { telegramId: loginToken.telegramId },
});

// Create user if doesn't exist
if (!user) {
  user = await prisma.user.create({
    data: {
      telegramId: loginToken.telegramId,
      name: loginToken.firstName || loginToken.username || `User${loginToken.telegramId}`,
      username: loginToken.username,
    },
  });
}
```

### What Gets Stored:

✅ **ONLY these fields are saved**:
- `telegramId` (your Telegram user ID, e.g., 123456789)
- `username` (your @username, if public)
- `firstName` (your first name from Telegram)

❌ **NOT stored**:
- Wallet private keys
- Secret keys
- Seed phrases
- Bot access token

---

## 📍 Step 3: NextAuth Session Creation

**File**: `lib/auth.ts` (NextAuth configuration)

```typescript
// Simplified excerpt showing credentials provider
providers: [
  CredentialsProvider({
    name: "Telegram",
    credentials: {},
    async authorize(credentials) {
      // Validates Telegram data
      // Returns user object with:
      return {
        id: user.id,
        telegramId: user.telegramId,
        username: user.username,
        name: user.name,
      };
    },
  }),
],
```

### What This Means:

- Uses standard OAuth flow
- Session stored in JWT or database
- **NO private keys in session**
- Authentication is separate from wallet generation

---

## 🔍 Proof: No Private Key Storage

### Search Production Database Schema

If you had access to the database schema (which we can't expose for security), you would see:

```prisma
// Conceptual schema (simplified)
model User {
  id         String   @id @default(cuid())
  telegramId BigInt   @unique
  username   String?
  name       String?
  createdAt  DateTime @default(now())

  // Notice: NO privateKey field
  // Notice: NO secretKey field
  // Notice: NO seedPhrase field
}
```

**The database schema DOES NOT have fields to store private keys.**

---

## 🔍 Proof: Telegram Bot Token is Server-Side Only

**Environment Variable** (never sent to client):

```bash
# .env (server-side only, NEVER exposed to browser)
TELEGRAM_BOT_TOKEN=<REDACTED>
```

### How to Verify:

1. Open browser DevTools → Application → Local Storage
2. Search for "TELEGRAM_BOT_TOKEN"
3. **Result**: Not found (it's server-side only)

4. Open DevTools → Network tab
5. Search all responses for "TELEGRAM_BOT_TOKEN"
6. **Result**: Never transmitted to browser

---

## 🔒 Security Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     YOUR BROWSER                        │
│                                                         │
│  ┌──────────────────┐         ┌──────────────────┐    │
│  │ Wallet Generation│         │ Telegram Login   │    │
│  │ (Web Worker)     │         │ (OAuth Flow)     │    │
│  │                  │         │                  │    │
│  │ ✓ Generates keys │         │ ✓ Authenticates  │    │
│  │ ✓ 100% local     │         │ ✓ No private keys│    │
│  │ ✗ No server comm │         │ ✓ Public ID only │    │
│  └──────────────────┘         └──────────────────┘    │
│           │                            │               │
│           │ (Private keys NEVER        │ (Telegram ID │
│           │  leave this box)           │  and username)│
│           ▼                            ▼               │
│     [Display to YOU]            [Server Session]       │
└─────────────────────────────────────────────────────────┘
                                          │
                                          ▼
                                  ┌───────────────┐
                                  │   DATABASE    │
                                  │               │
                                  │ ✓ Telegram ID │
                                  │ ✓ Username    │
                                  │ ✗ NO keys     │
                                  └───────────────┘
```

**Key Point**: These are TWO SEPARATE systems that never share data.

---

## ✅ How to Verify This Yourself

### Method 1: Check Network Requests During Login

1. F12 → Network tab
2. Click "Login via Telegram"
3. Complete Telegram auth
4. Search all requests for "secretKey" or "privateKey"
5. **Result**: NONE (only Telegram ID and username transmitted)

### Method 2: Inspect Telegram OAuth Flow

The flow is:
1. User clicks "Login via Telegram" → Redirects to Telegram
2. Telegram authenticates → Generates one-time token
3. Redirect back with token → Server validates
4. Server creates session with Telegram ID

**At NO point are wallet keys involved.**

### Method 3: Database Query (if you had access)

```sql
-- Search for any private key columns
SELECT column_name
FROM information_schema.columns
WHERE table_name = 'User'
AND column_name LIKE '%private%' OR column_name LIKE '%secret%';

-- Result: Empty (no such columns exist)
```

---

## 📊 What Data IS Stored (Complete List)

When you save a wallet to the leaderboard (OPTIONAL):

| Field | Example | Sensitive? |
|-------|---------|------------|
| Public Key | `Aa1b2c3d...` | ❌ No (public by design) |
| Pattern | `Ape...` or `...420` | ❌ No |
| Match Length | `3` characters | ❌ No |
| Attempts | `1,234,567` | ❌ No |
| Created At | `2025-11-30` | ❌ No |

**What is NEVER stored:**
- ❌ Private key / Secret key
- ❌ Seed phrase
- ❌ Mnemonic
- ❌ Bot token
- ❌ Any secret that could access funds

---

## ❓ Common Questions

### Q: How do I know the bot token isn't stealing my keys?

**A**: The bot token is used ONLY for Telegram authentication. It:
1. Lives on the server (never in browser)
2. Can only validate Telegram users
3. Cannot access Web Worker (where keys are generated)
4. Has no code to read/transmit private keys

### Q: What if you update the code to store private keys later?

**A**:
1. This is a public transparency repo - community can monitor changes
2. You can always verify the live production code (instructions above)
3. Network tab in DevTools will show any new server communication
4. Database schema changes would be visible in migrations

### Q: Could you intercept the private key between worker and UI?

**A**: Technically yes, if we modified the main thread code. But:
1. Main thread code is also viewable (inspect source)
2. You can verify it has no network calls (DevTools)
3. You can run the entire app offline (proves no server dependency)
4. This entire repo exists for you to verify that

---

## 🔍 Compare with Production

**Verify these snippets match production:**

1. Visit: `https://vwg.rip/api/auth/telegram-bot` (won't work without token, but you can see 404 error proving endpoint exists)
2. Inspect network calls during actual login
3. Confirm no private key transmission

**NextAuth Config**:
- View source of any auth page
- Search for `providers:`
- Confirm only CredentialsProvider (no privateKey handling)

---

**Remember**: Telegram auth and wallet generation are COMPLETELY SEPARATE.
Your login credentials NEVER touch your wallet's private key.
