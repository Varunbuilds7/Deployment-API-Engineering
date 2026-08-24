# TOOLS DATA SCIENCE: Deploy a CORS-Aware Metrics API
---

## 1. What This Question Is Really Testing

Strip away the FastAPI syntax and this question is testing **three separate engineering skills** stacked together:

| Skill | What the grader checks |
|---|---|
| API design | Correct parsing + correct JSON stats response |
| Browser security model (CORS) | Origin-specific `Access-Control-Allow-Origin`, no wildcard |
| Observability middleware | Per-request `X-Request-ID` and `X-Process-Time` headers |

The trap most students fall into: they get the stats endpoint right, but fail CORS because they either allow all origins (`*`) or hardcode a static header instead of dynamically checking the request's `Origin`.

---

## 2. Core Concept: How CORS Actually Works

CORS is a **browser-enforced** policy — your server can send whatever headers it wants, but it's the *browser* that decides whether to expose the response to JavaScript, based on the `Access-Control-Allow-Origin` (ACAO) header.

### The two-request lifecycle

1. **Preflight (`OPTIONS`)** — sent automatically by the browser before certain "non-simple" requests, to ask: *"is this origin allowed?"*
2. **Actual request (`GET`)** — sent only if the preflight succeeded; the browser again checks that the response carries the correct ACAO.

Your job is to make the server answer differently depending on the `Origin` header of the incoming request:

```
if request.Origin == ALLOWED_ORIGIN:
    respond with Access-Control-Allow-Origin: ALLOWED_ORIGIN
else:
    respond with NO Access-Control-Allow-Origin header at all
```

This is the key nuance: **rejecting** a disallowed origin doesn't mean sending an error — it means *omitting* the ACAO header. The request can still return 200, but without ACAO, the browser blocks the JS from reading it.

### Why you can't just use FastAPI's built-in `CORSMiddleware` naively

`CORSMiddleware(allow_origins=["https://dash-xp07do.example.com"])` actually does exactly what's needed here — it already reflects ACAO **only** for the allowed origin and omits it otherwise. The common mistake is instead using `allow_origins=["*"]` for "convenience," which the grader explicitly checks against.

---

## 3. Core Concept: Middleware for Request Timing & Tracing

`X-Request-ID` and `X-Process-Time` are classic **observability** headers used in real production APIs (you'll see them in Stripe, GitHub, etc. — used for debugging and tracing requests across distributed systems).

The pattern is a **custom ASGI/HTTP middleware** that wraps every request:

```
on request:
    start = now()
    request_id = uuid4()
    response = await call_next(request)
    response.headers["X-Request-ID"] = str(request_id)
    response.headers["X-Process-Time"] = str(now() - start)
    return response
```

**Important ordering nuance**: FastAPI/Starlette middleware executes in a stack — the *last* added middleware wraps *outermost*. If you add your custom timing middleware **after** `CORSMiddleware`, it will run outside CORS handling and its headers will still be attached correctly to all responses, including preflights. Test this — some students find their custom headers missing from `OPTIONS` responses because of load-order bugs.

---

## 4. Core Concept: Dynamic Computation (No Hardcoding)

Since the grader sends **fresh random integers on every check**, your `/stats` handler must:

1. Read `values` from query params as a raw string.
2. Split on commas, cast each to `int`.
3. Compute `count`, `sum`, `min`, `max`, `mean` using actual Python operations — not pre-baked constants.
4. Round/format `mean` sensibly (grader tolerance is ±0.01, so plain float division is fine — no need to worry about excessive precision).

Edge cases worth defending against even though not explicitly graded:
- Trailing/leading whitespace around numbers (`"1, 2, 3"`)
- Empty value list (decide on sane behavior, e.g. `count=0`)

---

## 5. Reference Implementation

```python
import time
import uuid
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

ALLOWED_ORIGIN = "https://dash-xp07do.example.com"
EMAIL = "<your-email>"  # replace with your exact logged-in email

# --- CORS: strict single-origin policy ---
app.add_middleware(
    CORSMiddleware,
    allow_origins=[ALLOWED_ORIGIN],
    allow_methods=["GET", "OPTIONS"],
    allow_headers=["*"],
)

# --- Custom observability middleware ---
@app.middleware("http")
async def add_tracing_headers(request: Request, call_next):
    start_time = time.perf_counter()
    request_id = str(uuid.uuid4())
    response = await call_next(request)
    process_time = time.perf_counter() - start_time
    response.headers["X-Request-ID"] = request_id
    response.headers["X-Process-Time"] = f"{process_time:.6f}"
    return response

@app.get("/stats")
def get_stats(values: str):
    nums = [int(v.strip()) for v in values.split(",") if v.strip() != ""]
    count = len(nums)
    total = sum(nums)
    mean = total / count if count else 0.0
    return {
        "email": EMAIL,
        "count": count,
        "sum": total,
        "min": min(nums) if nums else None,
        "max": max(nums) if nums else None,
        "mean": mean,
    }
```

### Why this satisfies every grader check

| Grader check | How this code handles it |
|---|---|
| Preflight from allowed origin → ACAO present | `CORSMiddleware` reflects ACAO only for `ALLOWED_ORIGIN` |
| Preflight from evil origin → no ACAO | Same middleware omits ACAO for any other origin automatically |
| `/stats` fresh random batch | Computed live from `values`, never hardcoded |
| `email` field | Static, matches logged-in address exactly |
| `X-Request-ID` | UUID4 generated per request in custom middleware |
| `X-Process-Time` | Wall-clock duration measured via `time.perf_counter()` |

---

## 6. Deployment Checklist

1. **requirements.txt**
   ```
   fastapi
   uvicorn
   ```
2. **Entry point** — most platforms (Render, Railway, Fly.io) need a start command:
   ```
   uvicorn main:app --host 0.0.0.0 --port $PORT
   ```
3. **Vercel** needs an ASGI adapter wrapper (`api/index.py` with `Mangum` or Vercel's native ASGI support) — if using Vercel, confirm CORS headers survive the serverless cold-start wrapper.
4. **Test before submitting** — a quick manual sanity check:
   ```bash
   curl -i -X OPTIONS https://your-deployed-url/stats \
     -H "Origin: https://dash-xp07do.example.com" \
     -H "Access-Control-Request-Method: GET"

   curl -i -X OPTIONS https://your-deployed-url/stats \
     -H "Origin: https://evil.example.com" \
     -H "Access-Control-Request-Method: GET"

   curl -i "https://your-deployed-url/stats?values=1,2,3,4,5"
   ```
   First call should show `access-control-allow-origin`, second should **not**, third should show all five stat fields plus both custom headers.

---

## 7. Common Pitfalls (Why Points Get Lost)

- **Using `allow_origins=["*"]`** — instantly fails the "no wildcards" requirement even if it technically returns ACAO.
- **Hardcoding stats for a sample input** during testing and forgetting to switch back to computed values.
- **Middleware order bugs** — custom header middleware added *before* CORS middleware can sometimes get bypassed on `OPTIONS` short-circuit responses. Add tracing middleware first in code, then CORS (Starlette executes bottom-up in registration order for the "outer wrapper" role) — but the safest approach is always to test the live preflight response directly rather than reason about it in the abstract.
- **Case-sensitivity / trailing slash mismatches** in the endpoint path (`/stats` vs `/stats/`) causing 404s on the grader's exact URL.
- **Cold starts on serverless platforms** inflating `X-Process-Time` unpredictably — shouldn't fail grading since it just checks "non-negative decimal," but worth knowing why your local time differs from deployed time.

---

## 8. Key Takeaway

This question is a compressed version of a real production concern: **any public API that's meant to be consumed by a specific frontend domain must enforce origin-based access control at the HTTP layer, while remaining observable via request tracing.** The FastAPI-specific mechanics (`CORSMiddleware`, `@app.middleware("http")`) are just the syntax — the transferable skill is understanding *why* CORS is enforced client-side and *why* production APIs always carry request IDs and timing headers for debugging.
