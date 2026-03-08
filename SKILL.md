---
name: agent-auth-byvigil-skills
description: Guide developers through integrating Vigil (Agent Auth) into their website — SDK setup, database schema, callback handling, sign-in flow, and credential verification. Use when building agent authentication, integrating Vigil/Agent Auth, or setting up DID-based identity for AI agents.
---

# Vigil (Agent Auth) Integration Guide

You are helping a developer integrate Vigil into their website so AI agents can authenticate. Vigil uses W3C DIDs with Ed25519 cryptography and issues VC-JWT credentials.

## Key URLs
- **API Base**: `https://auth.usevigil.dev`
- **Sign-in Page**: `https://usevigil.dev/signin?site_id=YOUR_SITE_ID&redirect_uri=YOUR_CALLBACK_URL`
- **Docs**: `https://usevigil.dev/docs`

## SDKs
- **Node.js**: `npm install auth-agents` — `import { AuthAgents } from "auth-agents"`
- **Python**: `pip install auth-agents` — `from auth_agents import AuthAgents`

## Integration Overview

There are two integration patterns:

### Pattern 1: Website Integration (Browser Redirect Flow)
For websites that want agents to "sign in" via redirect (like Google Sign-In).

1. Add a "Sign in with Vigil" link pointing to the Vigil sign-in page
2. Create a callback page that receives the credential
3. Create a backend API route that verifies the credential with Vigil
4. Store the verified agent identity in your database

### Pattern 2: Headless/API Integration
For APIs where agents authenticate programmatically without a browser.

1. Agent registers an identity and gets a credential via SDK
2. Agent presents credential to your API endpoint
3. Your backend verifies the credential with Vigil
4. Return a session or access token

## Step-by-Step: Browser Redirect Flow

### Step 1: Sign-In Link

```html
<a href="https://usevigil.dev/signin?site_id=YOUR_SITE_ID&redirect_uri=https://yoursite.com/auth/callback">
  Sign in with Vigil
</a>
```

Parameters:
- `site_id` — Your site identifier (get from Vigil dashboard or generate one)
- `redirect_uri` — Your callback URL. Vigil redirects here with credential in the URL hash fragment.

### Step 2: Callback Page (Client-Side)

After authentication, Vigil redirects to your callback URL with credentials in the hash:
`https://yoursite.com/auth/callback#credential=eyJ...&did=did:key:z6Mk...`

```typescript
// app/auth/callback/page.tsx (Next.js example)
"use client"
import { useEffect, useState } from "react"

export default function AuthCallbackPage() {
  const [status, setStatus] = useState<"loading" | "success" | "error">("loading")

  useEffect(() => {
    // Extract credential from URL hash fragment
    const hashParams = new URLSearchParams(window.location.hash.slice(1))
    const credential = hashParams.get("credential")
    const did = hashParams.get("did")

    // Strip credentials from URL immediately for security
    window.history.replaceState({}, document.title, window.location.pathname)

    if (!credential || !did) {
      setStatus("error")
      return
    }

    // Send to your backend for verification
    fetch("/api/auth/verify-did", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ did, credential }),
    })
      .then(res => res.json())
      .then(data => {
        if (data.valid) {
          // Store session however you prefer (cookie, sessionStorage, etc.)
          sessionStorage.setItem("agent_session", JSON.stringify(data))
          setStatus("success")
          window.location.href = "/dashboard"
        } else {
          setStatus("error")
        }
      })
      .catch(() => setStatus("error"))
  }, [])

  // Render loading/success/error UI...
}
```

### Step 3: Backend Verification Route

#### Next.js (App Router)

```typescript
// app/api/auth/verify-did/route.ts
import { AuthAgents } from "auth-agents"

const authAgents = new AuthAgents()

export async function POST(request: Request) {
  const { did, credential } = await request.json()

  if (!did || !credential) {
    return Response.json({ error: "Missing did or credential" }, { status: 400 })
  }

  // Verify credential with Vigil
  const result = await authAgents.verify(credential)

  if (!result.valid) {
    return Response.json({ valid: false, error: result.message }, { status: 401 })
  }

  // Ensure the DID in the credential matches the claimed DID
  if (result.did !== did) {
    return Response.json({ valid: false, error: "DID mismatch" }, { status: 401 })
  }

  // Store in your database (see database schema below)
  // await db.upsertAgent({ did: result.did, ... })

  return Response.json({
    valid: true,
    did: result.did,
    agent_name: result.agent_name,
    agent_model: result.agent_model,
    agent_provider: result.agent_provider,
    agent_purpose: result.agent_purpose,
    key_fingerprint: result.key_fingerprint,
    key_origin: result.key_origin,
    expires_at: result.expires_at,
  })
}
```

#### Express.js

```typescript
import express from "express"
import { AuthAgents } from "auth-agents"

const app = express()
const authAgents = new AuthAgents()

app.post("/api/auth/verify-did", express.json(), async (req, res) => {
  const { did, credential } = req.body

  const result = await authAgents.verify(credential)

  if (!result.valid) {
    return res.status(401).json({ valid: false, error: result.message })
  }

  // Create session
  req.session.agent = {
    did: result.did,
    name: result.agent_name,
    model: result.agent_model,
    key_origin: result.key_origin,
  }

  res.json({ valid: true, agent_name: result.agent_name })
})
```

#### Python (FastAPI)

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from auth_agents import AuthAgents

app = FastAPI()
client = AuthAgents()

class VerifyRequest(BaseModel):
    did: str
    credential: str

@app.post("/api/auth/verify-did")
async def verify_did(body: VerifyRequest):
    result = client.verify(body.credential)

    if not result.get("valid"):
        raise HTTPException(status_code=401, detail=result.get("message", "Invalid"))

    return {
        "valid": True,
        "did": result["did"],
        "agent_name": result.get("agent_name"),
        "agent_model": result.get("agent_model"),
        "agent_provider": result.get("agent_provider"),
        "key_origin": result.get("key_origin"),
        "expires_at": result.get("expires_at"),
    }
```

### Step 4: Database Schema

Store verified agent identities. Adapt to your database:

#### SQL (PostgreSQL / SQLite)

```sql
CREATE TABLE verified_agents (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  did           TEXT NOT NULL UNIQUE,
  site_id       TEXT,
  agent_name    TEXT NOT NULL,
  agent_model   TEXT,
  agent_provider TEXT,
  agent_purpose TEXT,
  key_fingerprint TEXT,
  key_origin    TEXT,          -- "server_generated" or "client_provided"
  credential_issuer TEXT,
  verified_at   DATETIME DEFAULT CURRENT_TIMESTAMP,
  expires_at    DATETIME,
  metadata      TEXT           -- JSON for extra data
);

CREATE INDEX idx_verified_agents_site ON verified_agents(site_id, expires_at);
```

#### Prisma

```prisma
model VerifiedAgent {
  id              Int      @id @default(autoincrement())
  did             String   @unique
  siteId          String?  @map("site_id")
  agentName       String   @map("agent_name")
  agentModel      String?  @map("agent_model")
  agentProvider   String?  @map("agent_provider")
  agentPurpose    String?  @map("agent_purpose")
  keyFingerprint  String?  @map("key_fingerprint")
  keyOrigin       String?  @map("key_origin")
  verifiedAt      DateTime @default(now()) @map("verified_at")
  expiresAt       DateTime? @map("expires_at")

  @@map("verified_agents")
}
```

#### Drizzle (for D1/SQLite)

```typescript
import { sqliteTable, text, integer } from "drizzle-orm/sqlite-core"

export const verifiedAgents = sqliteTable("verified_agents", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  did: text("did").notNull().unique(),
  siteId: text("site_id"),
  agentName: text("agent_name").notNull(),
  agentModel: text("agent_model"),
  agentProvider: text("agent_provider"),
  agentPurpose: text("agent_purpose"),
  keyFingerprint: text("key_fingerprint"),
  keyOrigin: text("key_origin"),
  verifiedAt: text("verified_at").default("CURRENT_TIMESTAMP"),
  expiresAt: text("expires_at"),
})
```

## Verify Response Shape

Successful `client.verify(credential)` returns:
```typescript
{
  valid: true,
  did: "did:key:z6Mk...",
  agent_name: "Claude",
  agent_model: "claude-opus-4-6",
  agent_provider: "Anthropic",
  agent_purpose: "Research assistant",
  key_fingerprint: "SHA256:a1b2c3d4...",
  key_origin: "server_generated" | "client_provided",
  issued_at: "2026-02-25T10:30:00.000Z",
  expires_at: "2026-02-26T10:30:00.000Z",
}
```

Failed verification returns (does NOT throw):
```typescript
{
  valid: false,
  error: "credential_expired" | "invalid_issuer" | "signature_invalid" | "credential_revoked",
  message: "Human-readable error description"
}
```

## Credential Expiration

Credentials have configurable lifetimes. The agent (or site) chooses at challenge time:
- `credential_expires_in` parameter in seconds
- Min: 300 (5 min), Max: 2592000 (30 days), `0` = non-expiring
- Default: 86400 (24 hours)
- When expired, agents re-authenticate via challenge-response to get a fresh credential

## Security Checklist

- Always verify credentials server-side, never trust client-only validation
- Strip credential from URL hash immediately after reading (prevent leakage via referrer/history)
- Validate that `result.did` matches the claimed DID
- Store credentials in HttpOnly cookies (not localStorage) for browser sessions
- Check `expires_at` — expired credentials should trigger re-authentication
- Rate limit your verification endpoint

## API Endpoints Reference

See [REFERENCE.md](references/REFERENCE.md) for the complete API specification.
