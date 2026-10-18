# Security Review: BITCOIN4Traders

**Review date:** 2026-10-18
**Scope:** Static review of selected API, authentication, trading, secrets, container, frontend, and test paths, plus limited dynamic tests against a local staging instance.
**Decision:** Not approved for live trading or publicly reachable deployment.

## Executive Summary

The tested API protects trading, model, and system routes with JWT authentication. Local probes also confirmed that the WebSocket rejects connections without a token, login attempts are throttled, untrusted CORS origins are not allowed, and the configured security headers are returned.

The primary finding is critical to the trading risk posture: an authenticated user can call the API order handler directly. That handler does not invoke the documented RiskEngine and accepts up to 100 BTC. In the isolated test, an order for 100 BTC was accepted as `paper_filled` because no exchange credentials were configured. With working Binance credentials, the same path can send real market orders.

The test server required a process-local Pydantic compatibility adapter. No repository source files were changed. No real exchange credentials, live orders, or external targets were used.

## Findings

### SEC-01: API order path bypasses the documented RiskEngine

**Severity:** High
**Status:** Confirmed in local staging behavior; live execution path confirmed by static review.

`POST /api/trading/order` calls `place_market_order()` directly when a connector is available. The handler does not connect the order to `risk_engine.py` or the documented risk gates. Request validation permits up to 100 BTC without checking account balance, notional, per-trade risk, or a stop-loss. Authentication exists, but does not limit financial impact if a valid admin token is misused.

**Test evidence:** With a local test admin token and empty exchange credentials, `quantity=100` returned HTTP 200 with `mode=paper`. No real exchange order was sent.

**Evidence:** [Trading route](../backend/api/trading.py#L214), [RiskEngine order calculation](../risk_engine.py#L549), [Router authentication](../backend/main.py#L223)

**Recommended action:** Route all order execution through a central risk/execution service; validate notional, allowed symbols, quantity, and position limits against account state; keep live mode explicitly disabled until these checks and negative tests pass; separate testnet and live credentials.

### SEC-02: Compose exposes unauthenticated Redis

**Severity:** High when deployed on a reachable host
**Status:** Confirmed by static configuration review.

Compose publishes Redis on host port 6379 without a password or a loopback-only binding. The API and frontend are also published on host ports, and the Compose configuration has no TLS reverse proxy. Redis may therefore be accessible to unauthorized clients when the host is reachable from an untrusted network.

**Evidence:** [Compose ports](../backend/docker-compose.yml#L8), [Redis service](../backend/docker-compose.yml#L35)

**Recommended action:** Do not publish Redis to the host; use the internal Compose network, bind exposed services to loopback, configure Redis ACL/authentication, and use protected TLS termination for external access.

### SEC-03: Fallback admin password is logged in plaintext

**Severity:** High if the fallback is used in a reachable environment
**Status:** Confirmed by static code-path review.

When `ADMIN_PASSWORD` is missing, the login module generates a random password and writes the password itself to a warning log. Compose does not set `ADMIN_PASSWORD` or `ADMIN_USERNAME`, making container logs a potential credential store.

**Evidence:** [Login fallback](../backend/api/login.py#L41), [Compose environment](../backend/docker-compose.yml#L10)

**Recommended action:** Fail startup if the required admin password is missing; never log passwords or tokens; inject credentials through a secret store or secure runtime configuration and restrict log access.

### SEC-04: JWT secret fallback and long token lifetime

**Severity:** Medium
**Status:** Confirmed by static code-path review.

If `SECRET_KEY` is not configured, a random key is generated at process start, invalidating existing tokens after a restart. JWTs are valid for 24 hours by default; no revocation or refresh mechanism was found in the reviewed login module.

**Evidence:** [JWT configuration](../backend/api/login.py#L19), [Token lifetime](../backend/api/login.py#L32)

**Recommended action:** Require a stable, strong secret at startup; reduce token lifetime and define revocation/rotation behavior for operational incidents.

### SEC-05: Bearer token is stored in `localStorage`

**Severity:** Medium
**Status:** Confirmed by frontend code review.

The API client reads the JWT from `localStorage`. Any successful frontend XSS can exfiltrate the token and use it until expiry.

**Evidence:** [API client](../frontend/src/api/client.ts#L96)

**Recommended action:** Consider an `HttpOnly`, `Secure`, `SameSite` cookie-based session. If bearer tokens remain, add CSP, maintain strict output encoding, review frontend dependencies, and shorten token lifetime.

### SEC-06: Login rate limiting is process-local

**Severity:** Medium for multi-worker or multi-instance deployment
**Status:** Confirmed statically and exercised locally.

Login attempts are tracked in an in-memory dictionary. The limit applies only to one process, resets on restart, and can be bypassed across workers or instances. Repeated local attempts were throttled with HTTP 429.

**Evidence:** [Login rate limiter](../backend/api/login.py#L74)

**Recommended action:** Use a shared limiter at the proxy or Redis layer, combine IP and account-based controls, trust forwarded IP headers only from known proxies, and bound limiter storage growth.

### SEC-07: Admin check on system environment endpoint is ineffective

**Severity:** Low with the current single-admin model; Medium if non-admin users are added
**Status:** Confirmed by static code review.

`get_env_variables()` sets `current_user` through `Depends(lambda: None)`, so it is always `None`. The handler imports `get_current_user` but never invokes it. This skips the extra admin-role check. The router still requires a valid JWT, and the current login module creates only an admin user, so this is a latent privilege boundary defect rather than a currently demonstrated unauthenticated access path.

**Evidence:** [System environment route](../backend/api/system.py#L196)

**Recommended action:** Inject `get_current_user` directly and add a test proving a non-admin cannot access the endpoint.

### SEC-08: Risk profile stores consent IP in plaintext

**Severity:** Medium for privacy and data minimization
**Status:** Confirmed by data model and code-path review.

The profiling persistence layer stores `consent_ip` as plaintext. Its own module documentation notes the privacy implication. IP addresses are personal data; purpose, retention, and access controls should be defined.

**Evidence:** [Profiling database](../src/user_profiling/database.py#L20), [Profiling submission](../backend/api/user_profiling.py#L300)

**Recommended action:** Confirm necessity and legal basis; remove or minimize/hash the value where appropriate, define retention/deletion, and restrict database access.

### SEC-09: Fixed example token in README

**Severity:** Low / Informational
**Status:** Confirmed in documentation; not evidence of a real credential.

The Colab quickstart assigns the fixed example value `bt4t-secret-token` to `BT4T_API_TOKEN`. It is visibly a placeholder, but users may copy it as a real token.

**Evidence:** [README example](../README.md#L33)

**Recommended action:** Mark the value as invalid and generate a random token per environment; do not use example values as production defaults.

## Additional Operational and Integration Blockers

These are not confirmed remote exploits, but they prevent reliable security operation and can invalidate testing.

### OPS-01: Backend container omits required project source

Compose uses `backend/` as the build context, and the backend Dockerfile copies only that context. `backend/main.py` imports `src.config` from the repository root. The `src/` tree is outside the build context, so the container cannot start normally as configured.

**Evidence:** [Build context](../backend/docker-compose.yml#L6), [Dockerfile copy](../backend/Dockerfile#L8), [Import](../backend/main.py#L27)

### OPS-02: Compose secret name does not match the code

Compose sets `BINANCE_SECRET_KEY`, while the secret mapping reads `BINANCE_API_SECRET`. In addition, `python-binance` is commented out in root requirements. The declared live configuration is inconsistent.

**Evidence:** [Compose variable](../backend/docker-compose.yml#L12), [Secret mapping](../src/config/secrets_manager.py#L216), [Root requirements](../requirements.txt#L51)

### OPS-03: API import fails on Pydantic and profiling imports

`Field(..., regex=...)` was removed in Pydantic 2, while backend requirements pin Pydantic 2.5.3. The local import failed at this field. The profiling module also imports `get_question_text`, which is absent from the reviewed `questionnaire.py`, leaving that module unavailable. The local test server could only be imported with a non-repository runtime adapter for `regex`; that adapter is not a product fix.

**Evidence:** [Profiling import](../backend/api/user_profiling.py#L30), [Pydantic field](../backend/api/user_profiling.py#L164), [Backend requirements](../backend/requirements.txt#L1)

### OPS-04: Paper portfolio produces unreliable position and P&L values

The paper engine treats `risk_per_trade` as a fraction of cash deployed, not as a maximum loss budget to a stop. Repeated buys overwrite the entry price; repeated shorts replace the stored position quantity while crediting short proceeds again. Portfolio and performance figures are unreliable for these paths.

**Evidence:** [Long purchase](../src/execution/multi_exchange_paper_trader.py#L277), [Entry price](../src/execution/multi_exchange_paper_trader.py#L287), [Short booking](../src/execution/multi_exchange_paper_trader.py#L313)

### OPS-05: CI targets a missing backend test directory

CI runs `pytest backend/tests -v`, but `backend/tests` is absent from the reviewed workspace. The local root test run was also blocked because NumPy was missing from the active Python environment.

**Evidence:** [CI test step](../.github/workflows/ci.yml#L48)

### OPS-06: Python compatibility declarations and setup are inconsistent

`pyproject.toml` and the README require Python 3.11+, while `setup.py` declares Python 3.8+. Under the available Python 3.14, the pinned `pydantic-core` could not build without the MSVC linker. Python 3.11 was not installed on the test host. A reproducible supported range and lockfile are missing.

**Evidence:** [pyproject.toml](../pyproject.toml), [setup.py](../setup.py)

### OPS-07: Frontend calls API routes that were not found in the backend

The frontend client contains order-cancel and orderbook calls for which no matching handler was found in the reviewed backend routes. These UI actions are expected to return 404.

**Evidence:** [Frontend client](../frontend/src/api/client.ts#L190), [Backend trading routes](../backend/api/trading.py#L251)

### OPS-08: Production-readiness claims are not substantiated

The README labels the project “Production Ready” and lists expected return/Sharpe values while also describing the software as research/educational. The review found no reproducible validation for those performance values, and the live API bypasses the documented RiskEngine.

**Evidence:** [Performance claims](../README.md#L359), [Status claim](../README.md#L480), [Risk documentation](../docs/RISK_MANAGEMENT.md#L64)

## Verified Controls

| Check | Result |
|---|---|
| Trading status without JWT | `401` |
| Trading order without JWT | `401` |
| Model training without JWT | `401` |
| System environment without JWT | `401` |
| WebSocket without token | Handshake rejected (`403`) |
| Login rate limit | `429` after the local limit was exceeded |
| CORS with `https://attacker.invalid` | No `Access-Control-Allow-Origin` |
| Security headers | `nosniff`, `DENY`, and `strict-origin-when-cross-origin` present |
| Authenticated paper order for 100 BTC | `200`, `paper_filled`; no exchange credentials configured |

## PoC and Release Decision

- **Local paper/data-flow PoC:** Conditionally suitable without exchange keys and without public network exposure. If no champion exists, the starter uses a simplified RSI fallback; this is not a performance validation.
- **Dashboard/container PoC:** Only after the build and import blockers are fixed.
- **Live/testnet trading:** Not approved until SEC-01 is fixed and order, position, stop, restart, and circuit-breaker behavior is covered by tests.

## Recommended Order of Work

1. Disable the live order path by default and enforce the central risk check.
2. Treat missing mandatory secrets as startup errors; never log passwords.
3. Fix the container build, secret variable names, and Redis/network exposure.
4. Resolve Pydantic/profiling imports and define a Python support matrix.
5. Fix paper accounting and add regression tests for long/short/add/close cases.
6. Repair CI test paths and add security tests for auth, roles, rate limits, and order limits.
7. Only then run an authorized testnet test with restricted API permissions in separate staging.

## Limitations

This is a static code/documentation review plus local API smoke tests, not a full penetration test. No internet targets, load/DoS tests, production data, exchange credentials, or real orders were used. Dynamic API probes ran with current packages and a process-local compatibility adapter because of the local Python 3.14/Pydantic mismatch; they are not identical to a standard production build.
