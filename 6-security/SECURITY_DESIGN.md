# Security Design

## Overview
This document outlines the security mechanisms for protecting user data, preventing attacks, and ensuring secure operation of the piggy bank backend system.

---

## Authentication & Authorization

### Authentication Mechanism: JWT (JSON Web Tokens)

**Design**: Stateless authentication using access tokens + refresh tokens

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant Backend
    participant DB

    User->>Frontend: Enter email/password
    Frontend->>Backend: POST /auth/login<br/>{email, password}
    Backend->>DB: Verify credentials
    DB-->>Backend: User data
    Backend->>Backend: Generate JWT tokens
    Backend-->>Frontend: {access_token, refresh_token}
    Frontend->>Frontend: Store tokens securely

    Note over Frontend,Backend: Subsequent requests

    Frontend->>Backend: GET /api/balance<br/>Authorization: Bearer <access_token>
    Backend->>Backend: Verify JWT signature & expiry
    Backend->>DB: Fetch balance
    DB-->>Backend: Balance data
    Backend-->>Frontend: {balance: 1000.00}
```

**Token Types**:

| Token Type | Lifetime | Storage | Purpose |
|-----------|----------|---------|---------|
| Access Token | 15 minutes | Memory (frontend) | API authentication |
| Refresh Token | 7 days | HTTP-only cookie or secure storage | Renew access tokens |

**Token Payload** (Access Token):
```json
{
  "user_id": "uuid-123",
  "email": "user@example.com",
  "role": "user",
  "iat": 1234567890,
  "exp": 1234568790
}
```

**Why JWT?**
- ✅ Stateless (no session storage needed)
- ✅ Scales horizontally (works with load balancing)
- ✅ Fast verification (no DB lookup per request)
- ❌ Hard to revoke before expiry
- **Mitigation**: Short access token lifetime (15 min)

---

### Authorization Mechanism: Role-Based Access Control (RBAC)

**Roles**:
- `user` - Standard user (deposit, withdraw, view own transactions)
- `admin` - Administrator (view all users, reconciliation, manual adjustments)

**Permission Matrix**:

| Action | User | Admin |
|--------|------|-------|
| Create deposit | ✅ | ✅ |
| Create withdrawal | ✅ | ✅ |
| View own balance | ✅ | ✅ |
| View own transactions | ✅ | ✅ |
| View all users | ❌ | ✅ |
| Manual balance adjustment | ❌ | ✅ |
| View webhook logs | ❌ | ✅ |
| Reconciliation reports | ❌ | ✅ |

**Enforcement Pattern**:
```python
# Decorator-based authorization
@require_auth
@require_role('user')
def get_balance(request):
    # User can only access their own balance
    user_id = request.jwt_payload['user_id']
    return Balance.objects.get(user_id=user_id)

@require_auth
@require_role('admin')
def get_all_users(request):
    # Only admins can access
    return User.objects.all()
```

**Resource Ownership Validation**:
```python
# Ensure users can only access their own resources
def get_transaction(request, transaction_id):
    user_id = request.jwt_payload['user_id']
    txn = Transaction.objects.get(id=transaction_id)

    # Authorization check
    if txn.user_id != user_id and request.user.role != 'admin':
        raise PermissionDenied("Not authorized to access this transaction")

    return txn
```

---

### Credential Protection

**Password Storage**: Bcrypt hashing with salt

```python
# User registration
password_hash = bcrypt.hashpw(password.encode('utf-8'), bcrypt.gensalt(rounds=12))

# Login verification
if bcrypt.checkpw(provided_password.encode('utf-8'), stored_hash):
    # Valid password
```

**Why Bcrypt?**
- ✅ Slow by design (prevents brute force)
- ✅ Automatic salting (prevents rainbow tables)
- ✅ Configurable cost factor (future-proof)
- Alternative: Argon2 (winner of Password Hashing Competition)

**Password Requirements**:
- Minimum 8 characters
- At least 1 uppercase, 1 lowercase, 1 number
- Optional: 1 special character
- Validated on both frontend and backend

**Credential Transmission**:
- HTTPS only (TLS 1.3)
- Passwords never logged
- Passwords never stored in plaintext
- Passwords never sent in URL parameters

---

## Attack Prevention

### 1. Replay Attacks on Webhooks ✅

**Already Covered in Section 5 (Webhook Handling)**

**Mechanisms**:
- Timestamp validation (5-minute window)
- Event ID deduplication (database UNIQUE constraint)
- Signature verification

**Reference**: `5-webhook-handling/WEBHOOK_HANDLING.md`

---

### 2. Spoofed Webhooks ✅

**Already Covered in Section 5 (Webhook Handling)**

**Mechanisms**:
- RSA-SHA256 signature verification
- Verify against Revio's public key
- Reject any webhook with invalid signature

**Reference**: `5-webhook-handling/WEBHOOK_HANDLING.md`

---

### 3. SQL Injection Prevention

**Primary Defense**: Django ORM (parameterized queries)

```python
# ✅ SAFE: ORM automatically parameterizes
user = User.objects.get(email=user_input)
transactions = Transaction.objects.filter(user_id=user_id, amount__gte=100)

# ❌ DANGEROUS: Raw SQL with string formatting
cursor.execute(f"SELECT * FROM users WHERE email = '{user_input}'")  # NEVER DO THIS!

# ✅ SAFE: Raw SQL with parameters
cursor.execute("SELECT * FROM users WHERE email = %s", [user_input])
```

**Defense Layers**:
1. **ORM Usage**: Use Django ORM for all queries (default)
2. **Input Validation**: Validate all inputs before processing
3. **Parameterized Queries**: If raw SQL needed, always use parameters
4. **Least Privilege**: Database user has minimal required permissions

**Database User Permissions**:
- Application user: SELECT, INSERT, UPDATE on application tables only
- No DROP, CREATE, ALTER permissions
- No access to system tables

---

### 4. Privilege Escalation Prevention

**Mechanisms**:

**A. Role Immutability**
- Users cannot change their own role
- Only admins can modify user roles
- Role changes logged in audit trail

**B. JWT Signature Verification**
- JWTs signed with secret key (HS256) or private key (RS256)
- Tampering with JWT payload invalidates signature
- User cannot modify their role in JWT

```python
# User tries to tamper with JWT
# Original: {"user_id": "123", "role": "user"}
# Tampered: {"user_id": "123", "role": "admin"}
# Result: Signature verification fails → 401 Unauthorized
```

**C. Server-Side Permission Checks**
- NEVER trust client-side role claims alone
- Always verify permissions on backend
- Check both role AND resource ownership

**D. Admin Action Logging**
```python
# Log all privileged actions
AuditLog.objects.create(
    action='manual_balance_adjustment',
    admin_user_id=admin.id,
    target_user_id=user.id,
    old_balance=old_balance,
    new_balance=new_balance,
    reason=reason
)
```

---

### 5. Cross-Site Scripting (XSS) Prevention

**Backend Responsibilities**:
- Return proper `Content-Type: application/json` headers
- Escape data in error messages
- Set security headers

**Security Headers**:
```python
# Django middleware configuration
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = 'DENY'
```

**Frontend Responsibilities** (handled by frontend team):
- Escape user input before rendering
- Use framework protections (React auto-escapes)
- Content Security Policy (CSP)

---

### 6. Cross-Site Request Forgery (CSRF) Prevention

**API Approach**: JWT in Authorization header (not cookies)

**Why This Prevents CSRF**:
- CSRF attacks rely on cookies being sent automatically
- JWTs in Authorization header require JavaScript to set
- Malicious site cannot read JWT from another domain (CORS)

**If Using Cookies for Refresh Tokens**:
```python
# Set SameSite attribute
response.set_cookie(
    'refresh_token',
    value=token,
    httponly=True,      # Prevent JavaScript access
    secure=True,        # HTTPS only
    samesite='Strict'   # Prevent CSRF
)
```

---

### 7. Rate Limiting

**Purpose**: Prevent brute force attacks, credential stuffing, API abuse

**Rate Limits**:

| Endpoint | Limit | Window |
|----------|-------|--------|
| `/auth/login` | 5 attempts | 15 minutes (per IP) |
| `/auth/register` | 3 accounts | 1 hour (per IP) |
| `/api/withdrawals` | 5 requests | 1 hour (per user) |
| `/api/*` (general) | 100 requests | 1 minute (per user) |
| `/webhooks/revio` | 1000 requests | 1 minute (global) |

**Implementation**: Django Ratelimit or Redis-based throttling

**Response**:
```json
HTTP 429 Too Many Requests
{
  "error": "Rate limit exceeded",
  "retry_after": 300
}
```

---

## Sensitive Data Protection

### API Keys & Secrets Storage

**Environment Variables** (development & production)

```bash
# .env file (NEVER commit to git)
DATABASE_URL=postgresql://user:pass@host:5432/db
REDIS_URL=redis://localhost:6379/0
REVIO_API_KEY=sk_live_abc123xyz
REVIO_WEBHOOK_SECRET=whsec_def456uvw
JWT_SECRET_KEY=super-secret-key-change-in-production
SENDGRID_API_KEY=SG.abc123xyz
```

**Railway Secrets** (production)
- Store secrets in Railway dashboard
- Injected as environment variables at runtime
- Encrypted at rest
- Access logged

**Access Pattern**:
```python
import os
from django.core.exceptions import ImproperlyConfigured

def get_env_variable(var_name):
    """Get environment variable or raise exception."""
    try:
        return os.environ[var_name]
    except KeyError:
        raise ImproperlyConfigured(f"Set {var_name} environment variable")

REVIO_API_KEY = get_env_variable('REVIO_API_KEY')
```

**Never**:
- ❌ Hardcode secrets in code
- ❌ Commit secrets to git
- ❌ Log secrets in application logs
- ❌ Return secrets in API responses

---

### Authentication Token Security

**Access Token** (JWT):
- Stored in frontend memory (not localStorage)
- Sent in `Authorization: Bearer <token>` header
- Short lifetime (15 minutes)
- Cleared on logout

**Refresh Token**:
- Stored in HTTP-only cookie OR secure storage
- Not accessible to JavaScript (if cookie)
- Longer lifetime (7 days)
- Rotated on use (optional, higher security)

**Token Rotation** (optional enhancement):
```python
# When refresh token used, issue new refresh token
old_refresh_token = request.cookies.get('refresh_token')
verify_refresh_token(old_refresh_token)

new_access_token = generate_access_token(user)
new_refresh_token = generate_refresh_token(user)  # New refresh token

# Invalidate old refresh token
invalidate_token(old_refresh_token)

return {
    'access_token': new_access_token,
    'refresh_token': new_refresh_token
}
```

---

### Data Encryption

#### Encryption at Rest

**Database Encryption**:
- Railway PostgreSQL: Encrypted at rest by default (AES-256)
- No additional application-level encryption needed for most fields

**Sensitive Fields** (optional, defense-in-depth):
- Bank account numbers: Application-level encryption with Fernet
- Personal identification: Tokenization or encryption

```python
from cryptography.fernet import Fernet

# Encrypt sensitive data before storing
cipher_suite = Fernet(encryption_key)
encrypted_account = cipher_suite.encrypt(bank_account.encode())

# Decrypt when needed
decrypted_account = cipher_suite.decrypt(encrypted_account).decode()
```

**Trade-off**:
- ✅ Extra protection if database compromised
- ❌ Cannot query encrypted fields directly
- ❌ Key management complexity
- **Decision**: Use for PII if compliance requires, otherwise rely on database encryption

#### Encryption in Transit

**HTTPS/TLS**:
- All API endpoints use HTTPS (TLS 1.3)
- Railway provides automatic SSL certificates
- HTTP requests redirected to HTTPS

**Django Settings**:
```python
# Force HTTPS
SECURE_SSL_REDIRECT = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')

# HSTS (HTTP Strict Transport Security)
SECURE_HSTS_SECONDS = 31536000  # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
```

**API Key Transmission**:
- Revio API calls use HTTPS
- API keys sent in `Authorization` header
- Certificate validation enabled

---

## Additional Security Measures

### 1. Input Validation

**Validation Layers**:
1. **Frontend**: Basic format checking (UX improvement)
2. **Backend**: Comprehensive validation (security boundary)
3. **Database**: Constraints (final enforcement)

**Example** (Withdrawal amount):
```python
from decimal import Decimal, InvalidOperation

def validate_withdrawal_amount(amount_str):
    """Validate withdrawal amount."""
    try:
        amount = Decimal(amount_str)
    except (InvalidOperation, ValueError):
        raise ValidationError("Invalid amount format")

    if amount <= 0:
        raise ValidationError("Amount must be positive")

    if amount > 10000:  # Daily limit
        raise ValidationError("Amount exceeds daily limit")

    if amount.as_tuple().exponent < -2:  # More than 2 decimal places
        raise ValidationError("Amount has too many decimal places")

    return amount
```

---

### 2. Audit Logging

**What to Log**:
- Authentication events (login, logout, failed attempts)
- Authorization failures (access denied)
- Sensitive operations (withdrawals, balance adjustments)
- Admin actions (user role changes, manual adjustments)
- Security events (invalid signatures, rate limit exceeded)

**Audit Log Structure**:
```python
AuditLog.objects.create(
    timestamp=timezone.now(),
    user_id=user.id,
    action='withdrawal_created',
    resource_type='transaction',
    resource_id=transaction.id,
    ip_address=request.META.get('REMOTE_ADDR'),
    user_agent=request.META.get('HTTP_USER_AGENT'),
    metadata={'amount': 100.00, 'status': 'pending'}
)
```

**Retention**: 1 year (compliance requirement)

---

### 3. Dependency Security

**Automated Scanning**:
- GitHub Dependabot (automatic PR for updates)
- `pip-audit` (check for known vulnerabilities)
- `safety check` (alternative tool)

**CI/CD Integration**:
```yaml
# .github/workflows/security.yml
- name: Security audit
  run: |
    pip install pip-audit
    pip-audit
```

**Update Policy**:
- Security patches: Apply immediately
- Minor versions: Monthly review
- Major versions: Quarterly review

---

### 4. Security Headers

**HTTP Security Headers**:
```python
# Django settings
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    # ... other middleware
]

# Security settings
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = 'DENY'
SECURE_HSTS_SECONDS = 31536000
```

**Headers Set**:
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `X-XSS-Protection: 1; mode=block`
- `Strict-Transport-Security: max-age=31536000`

---

## Security Checklist

### Authentication & Authorization
- [x] JWT-based authentication (access + refresh tokens)
- [x] Role-based authorization (user, admin)
- [x] Bcrypt password hashing (cost factor 12)
- [x] Resource ownership validation
- [x] Token expiration (15 min access, 7 day refresh)

### Attack Prevention
- [x] Replay attack prevention (timestamp + event ID)
- [x] Spoofed webhook prevention (signature verification)
- [x] SQL injection prevention (ORM + parameterized queries)
- [x] Privilege escalation prevention (immutable roles, server-side checks)
- [x] XSS prevention (security headers)
- [x] CSRF prevention (JWT in header, SameSite cookies)
- [x] Rate limiting (per endpoint, per user/IP)

### Data Protection
- [x] API keys in environment variables (not in code)
- [x] Tokens stored securely (memory/HTTP-only cookies)
- [x] Database encryption at rest (AES-256 via Railway)
- [x] HTTPS/TLS for all traffic (TLS 1.3)
- [x] HSTS enabled (force HTTPS)

### Operational Security
- [x] Audit logging (authentication, authorization, sensitive operations)
- [x] Dependency scanning (Dependabot, pip-audit)
- [x] Security headers (XSS, frame options, HSTS)
- [x] Least privilege database permissions

---

## Security Design Decisions

### Decision 1: JWT vs. Session-Based Auth

**Choice**: JWT (stateless tokens)

**Tradeoffs**:
- ✅ Scalable (no session storage)
- ✅ Works with load balancers
- ✅ Fast (no DB lookup)
- ❌ Hard to revoke
- **Mitigation**: Short access token lifetime (15 min)

---

### Decision 2: Bcrypt vs. Argon2 for Password Hashing

**Choice**: Bcrypt

**Tradeoffs**:
- ✅ Battle-tested (20+ years)
- ✅ Wide library support
- ✅ Configurable cost factor
- ❌ Argon2 is newer and more resistant to GPU attacks
- **Justification**: Bcrypt sufficient for current threat model

---

### Decision 3: Application-Level Encryption for Sensitive Data

**Choice**: Rely on database encryption (no app-level encryption initially)

**Tradeoffs**:
- ✅ Simpler implementation
- ✅ Can query fields directly
- ✅ Database encryption (AES-256) sufficient for most threats
- ❌ No protection if database credentials compromised
- **Evolution**: Add field-level encryption if compliance requires (PCI-DSS, etc.)

---

## Summary

**Authentication**: JWT tokens (15-min access, 7-day refresh), bcrypt password hashing

**Authorization**: RBAC (user/admin roles), server-side permission checks, resource ownership validation

**Attack Prevention**:
- Webhooks: Signature verification, timestamp validation, event deduplication
- SQL Injection: Django ORM, parameterized queries, least privilege DB user
- XSS/CSRF: Security headers, JWT in Authorization header
- Rate limiting: Per-endpoint limits, prevent brute force

**Data Protection**:
- Secrets: Environment variables, Railway secrets management
- Tokens: Secure storage (memory/HTTP-only cookies)
- At rest: Database encryption (AES-256)
- In transit: HTTPS/TLS 1.3, HSTS enabled

**Operational**: Audit logging, dependency scanning, security headers, regular updates
