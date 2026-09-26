# E26 — API Token Refresh

## 1. Problem Recognition

Many APIs use short-lived access tokens. A pipeline may authenticate successfully at startup and then begin receiving authentication failures after the token expires.

Recognize this problem when:

- access tokens have an expiration time
- OAuth 2.0 or similar token-based authentication is used
- long-running extraction jobs outlive token lifetime
- APIs return 401 or documented token-expired errors
- multiple workers share one credential
- authentication failures appear partway through otherwise healthy extraction

The production problem is not simply “get a new token.” It is:

> Refresh credentials safely without losing work, creating refresh storms, leaking secrets, or retrying requests incorrectly.

## 2. Concept and Reasoning

A token lifecycle looks like:

```text
OBTAIN TOKEN
    ↓
USE ACCESS TOKEN
    ↓
TOKEN NEAR EXPIRY
    ↓
REFRESH
    ↓
STORE NEW TOKEN SAFELY
    ↓
CONTINUE REQUESTS
```

A robust client must distinguish:

- access token
- refresh token
- expiration time
- token acquisition
- token refresh
- request retry

Token refresh is an authentication-state transition. It is not the same thing as an ordinary API retry.

## 3. Access Token vs Refresh Token

An access token is normally sent with API requests:

```http
Authorization: Bearer ACCESS_TOKEN
```

A refresh token is used to obtain another access token from the authorization server.

Conceptually:

```text
REFRESH TOKEN
      ↓
AUTHORIZATION SERVER
      ↓
NEW ACCESS TOKEN
      ↓
API REQUEST
```

Never send a refresh token to the resource API unless the provider explicitly defines that behavior.

## 4. Token Expiration

A token response may include:

```json
{
  "access_token": "redacted",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "redacted"
}
```

Do not wait until the exact expiration instant.

Use a safety margin:

```text
effective_expiry = expiry_time - refresh_margin
```

For example:

```text
token expiry = 10:00
refresh margin = 5 minutes
refresh before = 09:55
```

The margin protects against clock differences, network latency, and requests that begin just before expiration.

## 5. Never Log Tokens

Bad:

```python
logger.info("token=%s", access_token)
```

Better:

```python
logger.info(
    "access token refreshed; expires_at=%s",
    expires_at,
)
```

Never log:

- access tokens
- refresh tokens
- client secrets
- authorization codes
- full Authorization headers

Treat tokens as secrets even if they are short-lived.

## 6. Token State

A useful internal state model:

```python
from dataclasses import dataclass
from datetime import datetime


@dataclass
class TokenState:
    access_token: str
    expires_at: datetime
    refresh_token: str | None = None
```

The pipeline can then make a decision without exposing the secret:

```python
def token_is_expiring(state, now, margin_seconds=300):
    return now.timestamp() >= (
        state.expires_at.timestamp() - margin_seconds
    )
```

## 7. Refresh Before Expiration

Prefer proactive refresh when the token is near expiry:

```python
if token_is_expiring(state, now):
    state = refresh_token(state)
```

This reduces the number of requests that fail with 401.

However, proactive refresh should still be combined with reactive handling because:

- clocks can differ
- providers can revoke tokens
- tokens can be invalidated early
- long requests can cross the expiration boundary

## 8. Reactive Refresh After 401

A request may still return 401.

Correct pattern:

```text
REQUEST
  ↓
401 TOKEN EXPIRED
  ↓
REFRESH ONCE
  ↓
RETRY ORIGINAL REQUEST ONCE
  ↓
SUCCESS / FAIL
```

Do not blindly retry a 401 indefinitely.

A simple implementation:

```python
def request_with_refresh(client, token_manager, request):
    response = request(client, token_manager.access_token)

    if response.status_code != 401:
        return response

    token_manager.refresh()
    return request(client, token_manager.access_token)
```

The real implementation should classify the 401 because not every 401 means an expired access token.

## 9. Why 401 Must Be Classified

Authentication failures can represent different problems:

| Response | Possible meaning |
|---|---|
| 401 | expired/invalid access token |
| 403 | authenticated but not authorized |
| 400 | malformed authentication request |
| 429 | rate limited |
| 5xx | provider failure |

Do not refresh a token on every 403.

Do not retry invalid credentials forever.

Use the provider's documented error fields when available.

## 10. Single-Flight Refresh

A major production problem occurs when many workers notice expiration simultaneously.

Without coordination:

```text
100 workers
    ↓
token expires
    ↓
100 refresh requests
    ↓
authorization server overloaded
```

This is a refresh storm.

Use a single-flight pattern so one worker refreshes while others reuse the result.

Conceptually:

```text
WORKER A ──┐
WORKER B ──┼──→ REFRESH LOCK ──→ one refresh
WORKER C ──┘                 ↓
                         new token
                             ↓
                     all workers continue
```

## 11. Thread-Safe Token Manager

A simple local implementation:

```python
import threading
from datetime import datetime, timedelta, timezone


class TokenManager:
    def __init__(self, access_token, expires_at, refresh_fn):
        self._access_token = access_token
        self._expires_at = expires_at
        self._refresh_fn = refresh_fn
        self._lock = threading.Lock()

    def get_token(self):
        if self._needs_refresh():
            self._refresh_if_still_needed()

        return self._access_token

    def _needs_refresh(self):
        margin = timedelta(minutes=5)
        return datetime.now(timezone.utc) >= (
            self._expires_at - margin
        )

    def _refresh_if_still_needed(self):
        with self._lock:
            if not self._needs_refresh():
                return

            token = self._refresh_fn()

            self._access_token = token["access_token"]
            self._expires_at = (
                datetime.now(timezone.utc)
                + timedelta(seconds=token["expires_in"])
            )
```

The second expiry check inside the lock is important.

Without it, every worker that was waiting for the lock might refresh again after acquiring it.

## 12. Refresh Token Rotation

Some providers rotate refresh tokens.

Response:

```json
{
  "access_token": "new-access",
  "refresh_token": "new-refresh",
  "expires_in": 3600
}
```

The old refresh token may become invalid.

Therefore:

```text
NEW ACCESS TOKEN
+
NEW REFRESH TOKEN
        ↓
STORE BOTH AS ONE TOKEN STATE
```

Do not update only the access token if the provider rotates refresh tokens.

## 13. Atomic Credential Updates

When refresh-token rotation is possible, update related credential state atomically.

Bad:

```text
save new access token
      ↓
CRASH
      ↓
old refresh token remains
```

This can leave inconsistent credential state.

Prefer a transactional or atomic secret-store update:

```text
NEW ACCESS TOKEN
+
NEW REFRESH TOKEN
+
NEW EXPIRY
      ↓
ATOMIC UPDATE
```

## 14. Refresh Token Persistence

For a long-running pipeline, decide where token state lives.

Options include:

- process memory
- encrypted database storage
- secret manager
- provider-specific credential store

Process memory is simplest for short-lived jobs.

For long-lived services, durable secure storage may be required.

Never store refresh tokens in source code or ordinary configuration files committed to Git.

## 15. Multi-Tenant Credentials

A SaaS extraction may have separate credentials:

```text
tenant A → token A
tenant B → token B
tenant C → token C
```

Do not accidentally refresh tenant B using tenant A's credentials.

Credential state should be keyed by an explicit identity:

```text
(tenant_id, provider, account_id)
```

Each identity needs independent token state unless the provider explicitly supports shared authorization.

## 16. Token Refresh + Request Retry

Keep these concepts separate:

- **refresh:** obtain valid authentication state
- **retry:** repeat a failed API request

Correct flow:

```text
REQUEST
  ↓
401
  ↓
CLASSIFY AUTH ERROR
  ↓
REFRESH TOKEN
  ↓
RETRY ORIGINAL REQUEST ONCE
  ↓
SUCCESS / TERMINAL FAILURE
```

Do not let a request retry loop trigger unlimited token refreshes.

## 17. Refresh + Backoff

The authorization server can also fail.

A refresh request may return:

- timeout
- 429
- 5xx
- network error

Apply the appropriate retry policy to the token endpoint itself.

```text
REFRESH
  ↓
TRANSIENT FAILURE
  ↓
BACKOFF + JITTER
  ↓
REFRESH AGAIN
```

Do not use the same policy for every authentication failure.

Permanent credential failures should stop rather than retry forever.

## 18. Refresh + Rate Limits

Token endpoints can have their own rate limits.

A pipeline should therefore control both:

```text
RESOURCE API RATE LIMIT
+
TOKEN ENDPOINT RATE LIMIT
```

Do not assume token requests are unlimited because they are authentication traffic.

Single-flight refresh also reduces unnecessary token endpoint traffic.

## 19. Token Refresh + Checkpointing

Authentication recovery must not corrupt extraction progress.

Suppose:

```text
PAGE 25
   ↓
401
   ↓
REFRESH
   ↓
RETRY PAGE 25
   ↓
PERSIST
   ↓
CHECKPOINT
```

The cursor, page, or watermark should not advance merely because token refresh succeeded.

Authentication state and extraction state remain separate.

## 20. Token Refresh + Pagination

When a token expires during pagination:

```text
PAGE 10
  ↓
SUCCESS
  ↓
PAGE 11
  ↓
401
  ↓
REFRESH
  ↓
RETRY PAGE 11
  ↓
SUCCESS
  ↓
PAGE 12
```

Retry the same page/cursor.

Never move to the next pagination state because authentication failed.

## 21. OAuth Authorization Server Errors

A refresh response can contain errors such as:

```json
{
  "error": "invalid_grant"
}
```

This can indicate that the refresh token is invalid, expired, revoked, or otherwise unacceptable according to the provider.

Treat permanent credential errors differently from transient transport failures.

```text
invalid_grant
    ↓
DO NOT RETRY FOREVER
    ↓
MARK CREDENTIAL UNUSABLE
    ↓
REAUTHORIZATION / OPERATOR ACTION
```

The exact interpretation must follow the provider's documentation.

## 22. Clock Skew

Token expiration depends on time.

Different systems may have slightly different clocks:

```text
authorization server clock
        ≠
pipeline clock
```

Use a safety margin.

Also prefer a monotonic clock for measuring elapsed durations and deadlines. Use wall-clock timestamps for persisted expiry information.

Do not rely on exact equality with expiration time.

## 23. Token Refresh State Machine

A useful model:

```text
VALID
  ↓
NEAR_EXPIRY
  ↓
REFRESHING
  ↓
VALID
```

Failure:

```text
REFRESHING
    ↓
TRANSIENT FAILURE
    ↓
BACKOFF
    ↓
REFRESHING
```

Permanent failure:

```text
REFRESHING
    ↓
INVALID CREDENTIAL
    ↓
AUTH_REQUIRED
```

Explicit state makes operational behavior easier to reason about.

## 24. Python Token Manager Interface

A practical interface:

```python
class TokenProvider:
    def get_access_token(self) -> str:
        raise NotImplementedError

    def force_refresh(self) -> None:
        raise NotImplementedError
```

Request layer:

```python
def call_api(client, token_provider, method, url, **kwargs):
    token = token_provider.get_access_token()

    response = client.request(
        method,
        url,
        headers={"Authorization": f"Bearer {token}"},
        **kwargs,
    )

    if response.status_code != 401:
        return response

    token_provider.force_refresh()

    token = token_provider.get_access_token()

    return client.request(
        method,
        url,
        headers={"Authorization": f"Bearer {token}"},
        **kwargs,
    )
```

Production code should additionally classify the 401 and prevent repeated refresh loops.

## 25. Testing

### Unit tests

Test:

- token considered valid before safety margin
- proactive refresh near expiry
- refresh response parsing
- refresh-token rotation
- 401 classification
- single-flight refresh
- refresh failure classification
- token replacement
- credential isolation
- no token logging

Example:

```python
def test_expiring_token_requires_refresh():
    assert token_is_expiring(
        state,
        now,
        margin_seconds=300,
    )
```

### Integration tests

Fake authorization server:

```text
token request → access token A
API request → 401
refresh → access token B
same API request → 200
```

Verify that the original request is retried once with token B.

### Concurrency test

Start many workers with an expired token.

Expected:

```text
many workers
    ↓
one refresh
    ↓
new shared token
    ↓
many API requests
```

Not:

```text
many workers
    ↓
many refresh requests
```

## 26. Intentional Failure

### Failure Drill A — Expired Token

Configure the fake API to return 401 after a known point.

Expected:
- token refresh occurs
- original request is retried
- extraction continues
- pagination state is unchanged

### Failure Drill B — Refresh Storm

Start many workers with the same expired token.

Expected:
- single-flight coordination prevents unnecessary refresh calls

### Failure Drill C — Invalid Refresh Token

Make the authorization server return a permanent credential error.

Expected:
- refresh stops
- pipeline enters an authentication-required state
- no infinite retry loop occurs

### Failure Drill D — Refresh Endpoint 500

Return transient 5xx responses from the token endpoint.

Expected:
- bounded backoff is applied
- refresh eventually succeeds or fails within its retry budget

### Failure Drill E — Rotated Refresh Token

Return a new refresh token during refresh.

Expected:
- new access token and refresh token are stored together
- subsequent refresh uses the new refresh token

### Failure Drill F — Crash During Credential Update

Force a process failure during credential persistence.

Expected:
- credential state remains consistent
- no partially written token state is used

## 27. Observability

Track authentication behavior without secrets.

| Metric / Signal | Purpose |
|---|---|
| token refresh count | Measures refresh activity |
| proactive refresh count | Shows planned refreshes |
| reactive refresh count | Shows 401-triggered refreshes |
| refresh failures | Detects authorization-server problems |
| invalid credential count | Detects revoked/expired refresh state |
| token age | Helps diagnose expiration behavior |
| refresh latency | Measures auth endpoint performance |
| refresh retries | Measures transient auth failures |
| 401 rate | Detects token/authentication problems |
| refresh storm prevention | Confirms single-flight behavior |

Useful logs:

```text
run_id
tenant_id
provider
credential_id
refresh_reason
token_expires_at
refresh_attempt
error_class
refresh_latency_ms
```

Never log token values.

## 28. Recovery

When API authentication starts failing:

1. Determine whether failures are 401, 403, 429, or another class.
2. Check token expiry.
3. Check refresh endpoint health.
4. Check refresh-token validity.
5. Check credential rotation.
6. Check for concurrent refresh storms.
7. Verify tenant/account credential isolation.
8. Refresh or reauthorize according to provider policy.
9. Resume from the last safe extraction checkpoint.
10. Reconcile if requests were interrupted during authentication recovery.

Never manually copy tokens into source code or logs as a recovery mechanism.

## 29. Production Tools You Should Know

### OAuth 2.0
The authorization framework behind many access-token and refresh-token flows.

### AWS SDK / botocore
Provides production credential refresh patterns for AWS authentication and expiring credentials.

### Authlib
Python library supporting OAuth and OpenID Connect flows.

Understand token lifecycle mechanics before relying on a library's automatic refresh behavior.

## 30. Production Runbook

### Before deployment

- Document authentication flow.
- Document access-token lifetime.
- Document refresh-token lifetime and rotation.
- Define refresh safety margin.
- Define 401 handling.
- Define permanent authentication failures.
- Define refresh retry policy.
- Define credential storage.
- Define concurrency behavior.
- Define tenant credential isolation.
- Verify secrets are excluded from logs.

### If 401 errors increase

1. Check token expiration.
2. Check proactive refresh behavior.
3. Check authorization-server health.
4. Check refresh failures.
5. Check whether credentials were revoked.
6. Check whether multiple workers are refreshing simultaneously.
7. Verify the original request is retried only after successful refresh.

### If refresh fails permanently

1. Stop repeated refresh attempts.
2. Mark the credential unusable.
3. Preserve the last safe extraction checkpoint.
4. Trigger the documented reauthorization/operator workflow.
5. Resume extraction only after valid credentials are restored.

### What not to do

- Do not log access or refresh tokens.
- Do not refresh on every 403.
- Do not retry invalid credentials forever.
- Do not refresh separately in every worker without coordination.
- Do not advance pagination state because a token refresh succeeded.
- Do not store secrets in Git.
- Do not assume access-token expiration is the only authentication failure.

## 31. Common Mistakes

1. Waiting until the exact expiration time to refresh.
2. Treating every 401 as identical.
3. Refreshing on 403 responses.
4. Creating refresh storms across workers.
5. Ignoring refresh-token rotation.
6. Logging tokens.
7. Retrying permanent credential failures forever.
8. Updating access and refresh tokens non-atomically.
9. Mixing credentials between tenants.
10. Advancing extraction state during authentication recovery.
11. Ignoring token endpoint rate limits.
12. Using wall-clock time alone for elapsed retry/deadline calculations.

## 32. Definition of Done

- [ ] Access-token lifecycle is documented.
- [ ] Expiration and safety margin are defined.
- [ ] Proactive refresh exists where appropriate.
- [ ] Reactive 401 handling is classified.
- [ ] Original requests are retried safely.
- [ ] Refresh loops are bounded.
- [ ] Single-flight refresh is implemented for concurrent workers where needed.
- [ ] Refresh-token rotation is handled.
- [ ] Credential updates are atomic where required.
- [ ] Credentials are isolated by tenant/account.
- [ ] Refresh endpoint failures use appropriate retry behavior.
- [ ] Authentication state is separate from extraction state.
- [ ] Pagination resumes from the same safe position after refresh.
- [ ] Tokens never appear in logs.
- [ ] Authentication failure metrics exist.
- [ ] Expiration and refresh failure drills pass.
- [ ] Recovery and reauthorization procedures are documented.

## 33. What You Learned

After this recipe, you should be able to independently:

- explain access-token and refresh-token lifecycles
- implement proactive token refresh
- handle reactive 401 refresh safely
- classify authentication failures
- prevent refresh storms
- handle refresh-token rotation
- isolate credentials across tenants
- combine token refresh with retries and backoff
- preserve pagination and checkpoint state during refresh
- reason about clock skew and safety margins
- test concurrent token refresh
- recover from permanent credential failures
- operate token refresh safely in production

### Core Mental Model

```text
LOAD CREDENTIAL
      ↓
CHECK TOKEN EXPIRY
      ↓
VALID?
  /       \
YES       NO
 ↓         ↓
REQUEST   REFRESH
 ↓         ↓
401?      NEW TOKEN
 ↓         ↓
CLASSIFY  REQUEST
 ↓
REFRESH ONCE
 ↓
RETRY SAME REQUEST
 ↓
SUCCESS / AUTH FAILURE
```

> Token refresh is authentication-state recovery. Refresh the credential safely, retry the same logical request, and never allow authentication recovery to advance or corrupt extraction progress.
