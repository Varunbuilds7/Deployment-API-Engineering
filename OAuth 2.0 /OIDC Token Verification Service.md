# TOOLS DATA SCIENCE: OAuth 2.0 / OIDC Token Verification Service

---

## 1. What This Question Is Really Testing

This question tests whether you understand **how a resource server verifies a JWT issued by an identity provider**, without ever talking to that IdP directly. This is the core trust model behind OAuth 2.0 / OIDC in production systems — your API doesn't call the IdP on every request; it just holds the IdP's **public key** and checks the token's signature and claims locally.

Four things are graded, mapping to the four canonical JWT validation checks:

| Check | What it verifies |
|---|---|
| Signature | Token was actually issued by the real IdP (not forged) |
| `iss` (issuer) | Token came from *this* specific IdP, not another one |
| `aud` (audience) | Token was intended *for this API*, not some other service |
| `exp` (expiry) | Token hasn't expired |

---

## 2. Core Concept: How RS256 JWTs Work

A JWT has three parts: `header.payload.signature`, base64url-encoded and dot-separated.

- **RS256** = RSA signature with SHA-256. The IdP signs the `header.payload` using its **private key**; anyone holding the corresponding **public key** can verify that signature — but cannot forge one, since they don't have the private key.
- This is why you were given a **public key only** — you're the *verifier*, not the *issuer*.

### Why signature verification must happen first, and must be strict

If you decode the payload without verifying the signature (`jwt.decode(token, options={"verify_signature": False})`), you're trusting the *claims* without proving they came from the real IdP. That's the "tampered token" attack the grader tests for — someone edits the payload (e.g. changes `aud` or extends `exp`) but can't produce a valid signature over the new payload since they lack the private key. Correct verification must fail on this, because the signature no longer matches the modified payload.

### Why `iss` and `aud` checks matter separately from signature

Signature verification only proves *this exact IdP* issued *some* token. It does **not** prove:
- This IdP is the one your service trusts (`iss` check) — relevant if you might trust multiple issuers.
- This token was meant for your service specifically (`aud` check) — a token minted for a *different* API, even if validly signed by the same IdP, should not be accepted by yours. This prevents "token confusion" attacks where a token issued for Service A gets replayed against Service B.

---

## 3. Library Choice: `PyJWT`

`PyJWT` handles RS256, `iss`, `aud`, and `exp` validation natively — you don't need to hand-roll any of this.

```
pip install pyjwt cryptography
```

The `cryptography` package is required as a backend for RSA operations — `pyjwt` alone will raise on RS256 without it.

### Key method: `jwt.decode()`

```python
jwt.decode(
    token,
    key=PUBLIC_KEY_PEM,
    algorithms=["RS256"],
    audience=EXPECTED_AUD,
    issuer=EXPECTED_ISS,
)
```

Passing `audience` and `issuer` here makes PyJWT automatically validate `aud` and `iss` claims and raise if they mismatch — you don't need separate manual checks for those. `exp` is checked automatically by default whenever it's present in the payload (no option needed, as long as you don't explicitly disable it).

### Critical gotcha: `algorithms=["RS256"]` must be explicit

Never omit the `algorithms` parameter or leave it open-ended. This closes off the classic **"alg confusion" attack**, where a malicious token declares `"alg": "none"` or `"alg": "HS256"` (using the public key as an HMAC secret) to bypass verification entirely. Explicitly restricting to `["RS256"]` is a hard security requirement, not just good practice — and it directly maps to why the grader's "tampered token" probe must fail.

---

## 4. Handling the Public Key Correctly

The key you were given is in **PEM format**. Keep it as a raw multi-line string — do not strip or reformat the `-----BEGIN/END PUBLIC KEY-----` markers, and preserve the line breaks exactly.

Two safe ways to embed it in code:

```python
PUBLIC_KEY = """-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA2okOHspNjgA+2rTLbeuY
...
-----END PUBLIC KEY-----"""
```

or load it from an environment variable / file at startup — cleaner for deployment, since some platforms mangle multi-line env vars (watch for `\n` literal vs actual newline issues if you go the env-var route).

---

## 5. Reference Implementation

```python
import jwt
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

ISSUER = "https://idp.exam.local"
AUDIENCE = "tds-fycuirar.apps.exam.local"

PUBLIC_KEY = """-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA2okOHspNjgA+2rTLbeuY
cxiP/hG8C6Sb9iwg3yiLAA4HCnpITcbWCSelbvbYGuc3EbNy4xFyf5Cbj5DHJMID
EkryOgyd2giIIIBOUBj8S63uGcnRpOBh9NFatfNwheKuzsPuVNldu6A9cNteNpXc
WyJjG2axVfmq7i6SuKr1JoWYG7xTTAvKPujSl4OtsQfO3h5NepzdfXpr28oNnzfW
ed+zclR6BcmNNo/WVfJ4xyCLSf0BCOgdTgW6PdaChd1l9VDetJZVEgC5tkyvXsfI
SI6iyrYbKR0NEBSqq4XkadEjsCs4F1RncsS4LlgniT7GlkL9Mce3b0wGLs9/7ZIX
dQIDAQAB
-----END PUBLIC KEY-----"""

class VerifyRequest(BaseModel):
    token: str

@app.post("/verify")
def verify_token(body: VerifyRequest):
    try:
        claims = jwt.decode(
            body.token,
            key=PUBLIC_KEY,
            algorithms=["RS256"],
            audience=AUDIENCE,
            issuer=ISSUER,
        )
        return {
            "valid": True,
            "email": claims.get("email"),
            "sub": claims.get("sub"),
            "aud": claims.get("aud"),
        }
    except jwt.PyJWTError:
        from fastapi import Response
        return Response(
            content='{"valid": false}',
            status_code=401,
            media_type="application/json",
        )
```

### Why this satisfies every grader check

| Grader probe | How this code handles it |
|---|---|
| Valid token | Signature + `iss` + `aud` + `exp` all pass → 200 with echoed claims |
| Expired token | `jwt.decode` raises `ExpiredSignatureError` (subclass of `PyJWTError`) → 401 |
| Wrong audience | Raises `InvalidAudienceError` → 401 |
| Tampered payload / bad signature | Raises `InvalidSignatureError` → 401 |

`jwt.PyJWTError` is the base exception class for all of PyJWT's validation failures, so one `except` clause cleanly covers every rejection reason without needing to enumerate each one.

---

## 6. A Cleaner Way to Return 401 with a Body in FastAPI

Instead of manually constructing a `Response` object (shown above for clarity), FastAPI's idiomatic approach uses `JSONResponse`:

```python
from fastapi.responses import JSONResponse

@app.post("/verify")
def verify_token(body: VerifyRequest):
    try:
        claims = jwt.decode(...)
        return {"valid": True, "email": claims.get("email"), "sub": claims.get("sub"), "aud": claims.get("aud")}
    except jwt.PyJWTError:
        return JSONResponse(status_code=401, content={"valid": False})
```

This is more readable and avoids manually setting `media_type` and serializing JSON by hand.

---

## 7. Deployment Checklist

1. **requirements.txt**
   ```
   fastapi
   uvicorn
   pyjwt
   cryptography
   ```
2. Confirm the endpoint accepts `POST` with JSON body `{"token": "..."}` — not query params or form data.
3. Test locally with a self-signed token pair before deploying, to confirm the exact PEM string works with your PyJWT version (some versions are picky about trailing newlines in the PEM block — safest to keep the exact format given).
4. Quick manual sanity test post-deploy:
   ```bash
   curl -X POST https://your-deployed-url/verify \
     -H "Content-Type: application/json" \
     -d '{"token": "eyJhbGciOi..."}'
   ```

---

## 8. Common Pitfalls (Why Points Get Lost)

- **Not restricting `algorithms=["RS256"]`** — leaves the door open to algorithm-confusion attacks, and some PyJWT versions require this parameter anyway or raise an error.
- **Manually decoding without verification** (`verify=False`) to "peek" at claims — defeats the entire point of the exercise and will fail the tampered-token check.
- **Returning 200 even on failure** — grader explicitly checks for non-200 (401) on invalid tokens; returning `{"valid": false}` with status 200 will fail this.
- **Forgetting `audience=` or `issuer=` in `jwt.decode()`** — without these, PyJWT will *not* validate `aud`/`iss` at all, silently accepting tokens meant for other audiences.
- **Malformed PEM string** (extra whitespace, missing final newline before `-----END PUBLIC KEY-----`) — causes `cryptography` to throw a key-loading error before verification even runs. Copy the key block exactly as given.
- **Not handling the `email` claim being absent** — use `.get()` rather than direct indexing so a missing optional claim doesn't 500 the whole handler.

---

## 9. Key Takeaway

This question compresses the entire **resource-server side of OAuth2/OIDC** into one endpoint: verifying a bearer token's authenticity (signature), provenance (issuer), intended recipient (audience), and freshness (expiry) — all *without* a network call back to the IdP, which is exactly how production APIs validate JWTs at scale. The transferable skill isn't PyJWT syntax — it's understanding *why* each of the four checks exists and what specific attack it defends against.
