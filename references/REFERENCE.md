# Vigil API Reference

Base URL: `https://auth.usevigil.dev`

## POST /v1/identities — Register Agent Identity

Register a new agent. Optionally provide a public key (BYOK) or let the server generate one.

**Request:**
```json
{
  "agent_name": "Claude",
  "agent_model": "claude-opus-4-6",
  "agent_provider": "Anthropic",
  "agent_purpose": "Research assistant",
  "public_key_jwk": { "kty": "OKP", "crv": "Ed25519", "x": "..." },
  "metadata": { "version": "1.0" }
}
```
- `agent_name` — required
- `public_key_jwk` — optional. Omit for server-generated keys, provide for BYOK.
- `metadata` — optional key-value pairs

**Response (server-generated):**
```json
{
  "did": "did:key:z6Mk...",
  "credential": "eyJhbGciOiJFZERTQSJ9...",
  "key_fingerprint": "SHA256:a1b2c3d4...",
  "key_origin": "server_generated",
  "private_key_jwk": { "kty": "OKP", "crv": "Ed25519", "x": "...", "d": "..." },
  "_notice": "Save your private_key_jwk securely. Agent Auth does NOT store it."
}
```

**Response (BYOK):**
```json
{
  "did": "did:key:z6Mk...",
  "credential": "eyJhbGciOiJFZERTQSJ9...",
  "key_fingerprint": "SHA256:a1b2c3d4...",
  "key_origin": "client_provided"
}
```

**Rate limit:** 10 requests/hour per IP

---

## POST /v1/auth/challenge — Request Authentication Challenge

**Request:**
```json
{
  "did": "did:key:z6Mk...",
  "site_id": "site_abc123",
  "credential_expires_in": 3600
}
```
- `did` — required
- `site_id` — optional, scopes to your site
- `credential_expires_in` — optional, seconds (min 300, max 2592000, 0 = non-expiring)

**Response:**
```json
{
  "challenge_id": "ch_...",
  "nonce": "64-byte-hex-string",
  "expires_in": 60
}
```

**Rate limit:** 30 requests/minute per IP. Nonce expires in 60 seconds.

---

## POST /v1/auth/verify — Submit Signed Challenge

**Request:**
```json
{
  "challenge_id": "ch_...",
  "did": "did:key:z6Mk...",
  "signature": "base64url-encoded-ed25519-signature"
}
```

The nonce must be signed as UTF-8 text (not hex-decoded bytes).

**Response (success):**
```json
{
  "valid": true,
  "session_token": "sess_...",
  "credential": "eyJhbGciOiJFZERTQSJ9...",
  "agent": {
    "did": "did:key:z6Mk...",
    "agent_name": "Claude",
    "agent_model": "claude-opus-4-6",
    "agent_provider": "Anthropic",
    "agent_purpose": "Research assistant",
    "key_fingerprint": "SHA256:a1b2c3d4..."
  },
  "expires_in": 3600
}
```

**Response (failure):**
```json
{
  "valid": false,
  "error": "signature_invalid",
  "message": "The signature does not match the registered public key for this DID."
}
```

**Rate limit:** 30 requests/minute per IP

---

## POST /v1/credentials/verify — Verify a Credential

This is the endpoint websites call to verify an agent's credential.

**Request:**
```json
{
  "credential": "eyJhbGciOiJFZERTQSJ9..."
}
```

**Response (valid):**
```json
{
  "valid": true,
  "did": "did:key:z6Mk...",
  "agent_name": "Claude",
  "agent_model": "claude-opus-4-6",
  "agent_provider": "Anthropic",
  "agent_purpose": "Research assistant",
  "key_fingerprint": "SHA256:a1b2c3d4...",
  "key_origin": "server_generated",
  "issued_at": "2026-02-25T10:30:00.000Z",
  "expires_at": "2026-02-26T10:30:00.000Z"
}
```

**Response (expired):**
```json
{
  "valid": false,
  "error": "credential_expired",
  "message": "The credential has expired. The agent should re-authenticate via challenge-response to get a fresh credential."
}
```

**Error codes:** `credential_expired`, `invalid_issuer`, `signature_invalid`, `credential_revoked`

**Rate limit:** 60 requests/minute per IP

---

## GET /.well-known/did.json — Server DID Document

Returns Vigil's DID document with Ed25519 public key for offline verification.

---

## GET /health — Health Check

Returns service status and component health (database, kv_cache).

---

## SDK Methods Summary

| Method | Network | Description |
|--------|---------|-------------|
| `new AuthAgents(config?)` | — | Create client. `config.baseUrl` defaults to `https://auth.usevigil.dev` |
| `AuthAgents.generateKeyPair()` | No | Generate Ed25519 key pair locally |
| `AuthAgents.signChallenge(privateKeyJwk, nonce)` | No | Sign a challenge nonce locally |
| `client.register(input)` | Yes | Register new agent identity |
| `client.challenge(did, site_id?, credential_expires_in?)` | Yes | Request auth challenge |
| `client.authenticate(input)` | Yes | Submit signed challenge |
| `client.verify(credential)` | Yes | Verify a VC-JWT credential |
