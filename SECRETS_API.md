# Secrets API Implementation Plan

*Last Updated: February 11, 2026*  
*Owner: Garrett Carrol*

---

## Goal
Provide a centralized Secrets API that supports **human** and **non-human** (service/automation) secret placement with strong auditing, versioning, and short-lived access.

## Core Requirements
- Encrypt secrets at rest (KMS envelope encryption) and in transit (TLS).
- Versioned secrets with immutable history, metadata, and rotation schedules.
- Audit log for every read/write/rotate action.
- Policy-driven access control (repo, environment, service).

## Authentication & Authorization
### Human Access
- SSO/OAuth login + MFA.
- Role-based policies (admin, operator, read-only).

### Non-Human Access
- Workload identity with **OIDC** (GitHub Actions, CI/CD).
- Service accounts with short-lived tokens (15–60 min).
- Policies tied to workflow, repo, environment, and secret prefix.

## Non-Human Secret Placement Flow (OIDC)
1. CI job requests an OIDC token from the provider.
2. `POST /auth/oidc/exchange` with token + requested scope.
3. Secrets API validates allowlist policy (repo/workflow/env).
4. API returns a short-lived access token.
5. `POST /secrets` to create/rotate secrets with metadata.
6. Audit log captures the placement event.

```http
POST /auth/oidc/exchange
Content-Type: application/json

{
  "provider": "github",
  "token": "<oidc-token>",
  "scope": "write:secrets",
  "environment": "prod"
}
```

```http
POST /secrets
Authorization: Bearer <short-lived-token>
Content-Type: application/json

{
  "name": "enterprise-platform/prod/openai_api_key",
  "value": "<encrypted-in-transit>",
  "rotation_days": 30,
  "metadata": {
    "owner": "platform-team",
    "source": "ci",
    "workflow": "deploy"
  }
}
```

## API Surface (MVP)
- `POST /auth/oidc/exchange` - Exchange OIDC token for short-lived access.
- `POST /auth/service-token` - Service account token issuance.
- `POST /secrets` - Create or rotate a secret (versioned).
- `GET /secrets/{name}?environment=prod` - Fetch latest version.
- `GET /secrets/{name}/versions` - List history.
- `DELETE /secrets/{name}/versions/{id}` - Revoke a version.
- `GET /audit?subject=secret` - Audit stream.

## Placement Policies (Non-Human)
- Naming convention: `<service>/<env>/<secret-name>`.
- Non-human principals can only write to `automation/*` or their service prefix.
- Enforce environment allowlist per repo/workflow.
- Require metadata: `owner`, `source`, `workflow`, `rotation_days`.

## Rollout Checklist
- [ ] Define KMS keys and storage backend.
- [ ] Create policy allowlist for Tier-1 repos.
- [ ] Wire GitHub Actions OIDC to Secrets API.
- [ ] Implement audit log storage and retention.
- [ ] Pilot secret rotation on one repo before rollout.
