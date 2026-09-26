# E16 — API Authentication

API authentication is the security boundary that allows an extractor to access a protected API.

The extractor must obtain credentials securely, send them correctly, renew them when necessary, distinguish authentication from authorization failures, and prevent secrets from appearing in logs, checkpoints, or source code.

> Treat credentials as sensitive infrastructure state, not ordinary pipeline data.

## 1. Problem Recognition

Common problems:
- API keys are hard-coded in source code.
- Credentials are committed to Git.
- Authorization headers appear in logs.
- Access tokens expire during long extractions.
- Refresh tokens are revoked or rotated.
- Tokens have insufficient scopes.
- Different tenants require different credentials.
- Secret rotation breaks running jobs.
- Authentication failures are retried forever.
- Credentials leak into URLs or pipeline state.

## 2. Authentication vs Authorization

Authentication answers: who is making this request?

Authorization answers: what is this identity allowed to access?

```text
CLIENT
  ↓
AUTHENTICATE
  ↓
IDENTITY
  ↓
AUTHORIZE
  ↓
API RESOURCE
```

A valid credential can still receive HTTP 403 because the identity lacks permission.

## 3. Common Authentication Mechanisms

| Mechanism | Typical use |
|---|---|
| API key | Simple service-to-service APIs |
| Basic authentication | Legacy or controlled APIs |
| Bearer token | OAuth access tokens and similar credentials |
| OAuth 2.0 | Delegated or SaaS access |
| Service account | Machine-to-machine access |
| Signed requests | APIs requiring cryptographic request signing |
| mTLS | Strong client identity using certificates |

Use the mechanism defined by the provider contract.

## 4. Authentication Contract

Before implementation, document:
- authentication method
- credential types
- token endpoint
- required scopes
- token lifetime
- refresh behavior
- credential rotation procedure
- authentication headers
- expected 401 and 403 behavior
- environment-specific credentials
- tenant-specific credentials
- certificate requirements
- secret storage requirements

Do not assume two endpoints from the same SaaS platform use identical authentication behavior.

## 5. Never Hard-Code Secrets

Unsafe:

```python
API_KEY = "production-secret"
```

Safer:

```python
import os

API_KEY = os.environ["API_KEY"]
```

Production deployments commonly retrieve secrets from a dedicated secret manager.

## 6. Separate Configuration from Credentials

Non-sensitive configuration can be ordinary application configuration:

```python
import os

API_BASE_URL = os.environ["API_BASE_URL"]
API_TIMEOUT_SECONDS = int(os.environ.get("API_TIMEOUT_SECONDS", "30"))
API_KEY = os.environ["API_KEY"]
```

The application references the secret. It does not contain the secret.

## 7. Credential Provider

Keep credential retrieval separate from extraction logic.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class ApiCredentials:
    api_key: str


class CredentialProvider:
    def get_credentials(self) -> ApiCredentials:
        raise NotImplementedError
```

An environment implementation can read the credential from environment configuration. A production implementation can instead read from a secret manager.

## 8. Centralize Authentication

Do not scatter authentication headers throughout extractor functions.

```python
class ApiClient:
    def __init__(self, base_url, credentials):
        self.base_url = base_url
        self.credentials = credentials

    def headers(self):
        return {
            "Authorization": f"Bearer {self.credentials.api_key}",
            "Accept": "application/json",
        }
```

The exact header and authentication scheme must match the provider.

## 9. API Keys

An API may use a dedicated header:

```python
headers = {
    "X-API-Key": api_key,
}
```

Another API may use a bearer-style header. Never guess the placement.

## 10. Basic Authentication

Python can use HTTP authentication directly:

```python
import requests

response = requests.get(
    "https://api.example.com/customers",
    auth=(username, password),
    timeout=30,
)
```

Never manually put usernames or passwords into URLs.

## 11. Bearer Tokens

Bearer authentication commonly uses:

```python
headers = {
    "Authorization": f"Bearer {access_token}",
}
```

Treat the token as a secret and never log the complete header.

## 12. OAuth 2.0

OAuth commonly follows:

```text
AUTHORIZATION SERVER
        ↓
ACCESS TOKEN
        ↓
RESOURCE SERVER
        ↓
API DATA
```

Important concepts include client identity, authorization grants, access tokens, refresh tokens, scopes, and token expiration.

## 13. Client Credentials Flow

For machine-to-machine integrations, the client-credentials flow may be appropriate.

```python
import requests


def get_access_token(token_url, client_id, client_secret):
    response = requests.post(
        token_url,
        data={"grant_type": "client_credentials"},
        auth=(client_id, client_secret),
        timeout=30,
    )
    response.raise_for_status()
    return response.json()["access_token"]
```

The provider's exact OAuth contract takes precedence.

## 14. Token Expiration

Access tokens often have limited lifetimes:

```text
TOKEN
 ↓
VALID
 ↓
EXPIRES
 ↓
401
 ↓
REFRESH
 ↓
NEW TOKEN
 ↓
CONTROLLED RETRY
```

Do not retry a 401 indefinitely without changing authentication state.

## 15. Refresh Tokens

A refresh token can often obtain a new access token:

```python
def refresh_access_token(refresh_url, refresh_token):
    response = requests.post(
        refresh_url,
        data={
            "grant_type": "refresh_token",
            "refresh_token": refresh_token,
        },
        timeout=30,
    )
    response.raise_for_status()
    return response.json()["access_token"]
```

Some providers rotate refresh tokens. If so, the replacement must be stored safely according to the provider contract.

## 16. Token Refresh Concurrency

Multiple workers can otherwise refresh the same credential simultaneously:

```text
WORKER A → EXPIRED → REFRESH
WORKER B → EXPIRED → REFRESH
WORKER C → EXPIRED → REFRESH
```

Use controlled token management when workers share credentials:

```text
TOKEN CACHE
     ↓
CONTROLLED REFRESH
     ↓
NEW TOKEN
     ↓
WORKERS
```

The exact locking mechanism depends on whether workers share a process, host, or distributed environment.

## 17. Scopes and Permissions

OAuth credentials can have scopes such as:

```text
customers:read
payments:read
payments:write
```

Typical diagnostic distinction:

```text
401 → authentication problem
403 → authorization / permission problem
```

Individual APIs can differ, so always verify the provider documentation.

## 18. Multi-Tenant Credentials

A multi-tenant extractor may use separate credentials:

```text
TENANT A → CREDENTIAL A → API
TENANT B → CREDENTIAL B → API
TENANT C → CREDENTIAL C → API
```

Never reuse one tenant's credential for another tenant accidentally.

Tenant identity should be explicit in synchronization state and operational logs.

## 19. Credential Metadata

Store safe metadata, not secrets, in ordinary pipeline tables:

```sql
CREATE TABLE api_credential_metadata (
    integration_id TEXT PRIMARY KEY,
    provider TEXT NOT NULL,
    tenant_id TEXT,
    auth_type TEXT NOT NULL,
    scope_summary TEXT,
    expires_at TIMESTAMPTZ,
    rotated_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ NOT NULL
);
```

Do not put access tokens, client secrets, passwords, or private keys in ordinary run tables.

## 20. Authentication Failure Classification

| Response | Typical meaning | Action |
|---|---|---|
| 401 | Missing, invalid, or expired credentials | Refresh or obtain valid credentials |
| 403 | Authenticated but not authorized | Check scopes and permissions |
| 429 | Rate limited | Apply rate-limit handling |
| 500 | Provider/server failure | Apply transient-failure policy |
| Timeout | Transport/dependency issue | Apply network-failure policy |

Status-code meaning remains provider-specific.

## 21. Authentication Middleware

Centralize token acquisition in the client:

```python
class AuthenticatedClient:
    def __init__(self, transport, token_provider):
        self.transport = transport
        self.token_provider = token_provider

    def get(self, url, **kwargs):
        token = self.token_provider.get_access_token()
        headers = kwargs.pop("headers", {})
        headers["Authorization"] = f"Bearer {token}"
        return self.transport.get(url, headers=headers, **kwargs)
```

This prevents every extraction function from implementing its own token logic.

## 22. Authentication and Retries

Keep authentication refresh separate from general network retries.

```text
REQUEST
  ↓
401?
  ├── NO → normal response handling
  └── YES
       ↓
   refresh / reauthenticate
       ↓
   controlled retry
       ↓
   success or clear failure
```

Do not combine every failure into one generic retry loop.

## 23. Secret Redaction

Never log complete request headers:

```python
logger.info("Request headers: %s", headers)
```

Instead, log safe request metadata:

```python
logger.info(
    "API request",
    extra={
        "method": "GET",
        "endpoint": "/customers",
    },
)
```

Teaching redaction function:

```python
SENSITIVE_HEADERS = {
    "authorization",
    "x-api-key",
    "proxy-authorization",
}


def redact_headers(headers):
    return {
        key: "[REDACTED]" if key.lower() in SENSITIVE_HEADERS else value
        for key, value in headers.items()
    }
```

## 24. URL Credential Leakage

Unsafe:

```text
https://api.example.com/data?api_key=secret
```

URLs can appear in access logs, proxy logs, tracing systems, and monitoring tools.

Prefer headers when the provider supports them.

## 25. Secret Rotation

Design credentials to be replaceable without source-code changes:

```text
NEW SECRET CREATED
       ↓
SECRET STORE UPDATED
       ↓
APPLICATION REFRESH / RESTART
       ↓
NEW REQUESTS USE NEW SECRET
       ↓
OLD SECRET REVOKED
```

Where the provider supports overlapping credentials, validate the replacement before revoking the old credential.

## 26. Certificates and mTLS

Some APIs require mutual TLS:

```text
CLIENT CERTIFICATE + PRIVATE KEY
            ↓
TLS HANDSHAKE
            ↓
SERVER AUTHENTICATES CLIENT
            ↓
HTTPS API
```

Private keys require the same protection as other high-value secrets.

## 27. Local Development

Use separate development credentials.

Tests should prefer:
- fake credentials
- mocked authentication
- provider sandbox accounts
- isolated test tenants

Do not copy production credentials into local environments unless explicitly required and controlled.

## 28. Testing

Test at least:
1. Valid API key.
2. Missing API key.
3. Invalid API key.
4. Expired access token.
5. Successful token refresh.
6. Failed token refresh.
7. Refresh-token rotation.
8. Missing OAuth scope.
9. Tenant credential isolation.
10. Secret redaction.
11. Credential rotation.
12. Provider returns 401.
13. Provider returns 403.
14. Authentication retry limit.
15. No secrets in logs.
16. No secrets in URLs.
17. No secrets in checkpoints.
18. Concurrent token refresh.
19. mTLS configuration where applicable.
20. Restart after credential rotation.

## 29. Example Unit Tests

```python
def test_authorization_header_is_redacted():
    headers = {
        "Authorization": "Bearer secret-token",
        "Accept": "application/json",
    }
    safe = redact_headers(headers)
    assert safe["Authorization"] == "[REDACTED]"
    assert safe["Accept"] == "application/json"


def test_api_key_header_is_redacted():
    headers = {"X-API-Key": "secret-key"}
    safe = redact_headers(headers)
    assert safe["X-API-Key"] == "[REDACTED]"


def test_non_sensitive_headers_are_preserved():
    headers = {"Accept": "application/json"}
    safe = redact_headers(headers)
    assert safe["Accept"] == "application/json"
```

Add integration tests against a mock or sandbox provider for the complete authentication flow.

## 30. Observability

Track:

```text
api_auth_requests_total
api_auth_failures_total
api_401_total
api_403_total
api_token_refresh_total
api_token_refresh_failures_total
api_credentials_expiring_total
api_credential_rotation_total
api_authentication_latency_seconds
```

Useful log fields:

```text
provider
tenant_id
integration_id
auth_type
endpoint
http_status
error_type
token_refresh_attempt
```

Never log access tokens, refresh tokens, API keys, passwords, private keys, or Authorization headers.

## 31. Intentional Failure

### Failure 1 — Remove the API key
Remove the configured credential. Verify the pipeline fails clearly instead of producing misleading extraction errors.

### Failure 2 — Expire the access token
Use a short-lived test token. Verify the extractor refreshes it and continues.

### Failure 3 — Revoke the refresh token
Verify the pipeline stops and reports an authentication failure instead of retrying forever.

### Failure 4 — Remove a required scope
Verify the API returns an authorization failure and the pipeline identifies it as a permission problem.

### Failure 5 — Leak a header
Temporarily add unsafe header logging in a test environment. Verify the secret appears, then fix the logging boundary and add a regression test preventing leakage.

### Failure 6 — Rotate credentials
Replace a credential during a running test. Verify the pipeline transitions to the replacement according to the provider's rotation model.

### Failure 7 — Tenant credential mix-up
Configure the wrong credential for one test tenant. Verify the system detects the failure and does not silently associate the response with the wrong tenant.

## 32. Recovery

1. Identify provider and tenant.
2. Determine authentication mechanism.
3. Check credential availability.
4. Check expiration.
5. Check required scopes.
6. Check provider status.
7. Inspect 401 versus 403 responses.
8. Refresh or rotate credentials when appropriate.
9. Verify replacement credentials against a safe endpoint.
10. Resume extraction from the last safe checkpoint.
11. Confirm no authentication material entered logs or pipeline state.
12. Verify subsequent requests succeed.

Do not reset extraction checkpoints merely because authentication failed. Authentication failure does not automatically mean source data was lost.

## 33. Production Tools You Should Know

| Tool | What to know |
|---|---|
| **HashiCorp Vault** | Centralized secret storage, access control, leasing, and rotation capabilities. |
| **AWS Secrets Manager** | Managed secret storage with IAM integration and rotation capabilities. |
| **OAuth 2.0** | Authorization framework used by many SaaS and API integrations. |

Understand the security boundary first. A secret manager does not replace correct authentication design.

## 34. Production Runbook

### API returns 401
Check credentials, access-token expiration, token refresh, credential revocation, header construction, and provider authentication status.

### API returns 403
Check OAuth scopes, API permissions, tenant access, resource permissions, and provider policy.

### Token refresh fails
Check refresh-token validity, client credentials, token endpoint, OAuth grant configuration, provider status, and refresh-token rotation.

### Secrets appear in logs
Immediately stop unsafe logging, identify the exposed secret, restrict affected logs, rotate the credential, add redaction, add regression tests, and review log retention.

### What not to do
Do not commit secrets to Git. Do not put credentials in URLs. Do not log Authorization headers. Do not store tokens in ordinary pipeline run records. Do not retry 401 forever. Do not confuse 403 with network failure. Do not share production credentials between tenants. Do not revoke the only working credential before validating its replacement. Do not disable TLS verification to fix authentication.

## 35. Definition of Done

You are done when you can:
- explain authentication versus authorization
- identify common API authentication mechanisms
- document an authentication contract
- load secrets safely
- separate configuration from credentials
- build a credential provider
- centralize authentication in an API client
- implement API-key authentication
- implement bearer authentication
- understand OAuth 2.0
- implement token refresh
- handle token expiration
- understand scopes
- isolate tenant credentials
- redact sensitive headers
- prevent URL credential leakage
- design credential rotation
- understand mTLS at a high level
- test authentication failures
- observe authentication without exposing secrets
- intentionally break authentication
- recover safely
- prove credentials are absent from logs and checkpoints

## 36. What You Learned

API authentication is a security boundary around extraction, not merely an HTTP header.

The core pattern is:

```text
LOAD CREDENTIAL
      ↓
AUTHENTICATE
      ↓
AUTHORIZE
      ↓
REQUEST
      ↓
HANDLE EXPIRATION
      ↓
REFRESH / ROTATE
      ↓
CONTINUE EXTRACTION
      ↓
OBSERVE WITHOUT LEAKING SECRETS
```

The most important rule is:

> Treat credentials as sensitive infrastructure state. Never allow authentication material to become ordinary pipeline data.

Once this principle is understood, API keys, OAuth tokens, refresh flows, scopes, tenant isolation, secret rotation, redaction, and authentication recovery become parts of the same security boundary.