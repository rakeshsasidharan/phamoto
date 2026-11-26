# Phamoto — Technical Specification

| Field | Value |
|---|---|
| Product | Phamoto (family photo, video & document storage) |
| Companion to | [PRD.md](PRD.md) v0.1 (2026-10-07) |
| Document type | Technical Specification (*how*). The PRD defines *what & why* and wins on any conflict of intent. |
| Version | 0.1 (draft) |
| Date | 2026-11-24 |
| Owner | Rakesh Pillai |

---

## 0. How to use this document

- Section numbers are stable. Requirement IDs (`FR-…`, `NFR-…`, `PL-…`) refer to the PRD. New technical decisions in this spec have IDs `TD-nnn`.
- Every **[TECH SPEC]** marker and every open question (OQ) in the PRD is resolved in §1. Build agents MUST treat §1 as binding unless the owner changes it.
- Items marked **[SPIKE]** are decisions that hold unless a time-boxed spike in M0 shows they don't work. Each spike has a fallback that is already designed, so a failed spike changes an implementation, not the architecture.
- Prices are approximate (late 2026; S3, Lambda, SQS and CloudFront list prices in us-west-2 match us-east-1, the PRD's reference region) and only used for design trade-offs. Check current AWS pricing before using them in the cost dashboard config.
- Code identifiers in this document (`item`, `album_item`, `/v1/items`, queue names) are normative. Keep them unless there is a reason to change them, and update this document if you do.

---

## 1. Decision summary

### 1.1 Resolved [TECH SPEC] items and open questions

| ID | Topic | Decision | Section |
|---|---|---|---|
| TD-001 | Region (OQ-02) | Primary region **us-west-2** (Oregon), decided by the owner. Kept as a config value, not hard-coded. CloudFront certificates, CloudFront Functions and KeyValueStore live in us-east-1 regardless (AWS requirement). | §3.1 |
| TD-002 | Metadata store (OQ-06) | **Amazon Aurora DSQL** (serverless, PostgreSQL-compatible, billed per use, no VPC needed). Queries go through a portable SQL layer so that **Aurora Serverless v2 PostgreSQL** remains a drop-in fallback. **[SPIKE S1]** | §6 |
| TD-003 | Web hosting (PL-WEB-001) | Next.js **static export** (`output: 'export'`) on S3 and CloudFront. No SSR Lambda. The web app is a pure client of the public API (enforces PL-X-002). | §20 |
| TD-004 | API compute | One TypeScript "API" Lambda (Hono router) behind API Gateway HTTP API, served at `phamoto.com/api/v1/*` through the same CloudFront distribution as the web app. A separate **Vault API** Lambda holds the only IAM role that can decrypt vault content. | §5, §13 |
| TD-005 | Identity | **In-house auth module** built on vetted libraries (`openid-client`, `@simplewebauthn/server`, `otplib`, `jose`), not Cognito. Reason: step-up vault elevation, recovery codes, owner-confirmed re-binding, session listing and invite-only binding are all first-class needs that Cognito only covers partially. | §4 |
| TD-006 | Processing compute (FR-PRC-001) | Self-run tools (libvips, libheif, LibRaw, exiftool, ffmpeg, poppler, LibreOffice) in **Lambda container images (arm64)**, with **ECS Fargate Spot** tasks for jobs that exceed Lambda limits (long videos, very large files). AWS Elemental MediaConvert is **not** used. | §10 |
| TD-007 | Pipeline orchestration | S3 → EventBridge → **SQS** queues with idempotent workers. Step Functions only for low-volume, multi-step jobs (export, member removal, re-process). | §10.1 |
| TD-008 | Video proxy (OQ-07) | **Progressive MP4** (H.264 High, 720p, ~2 Mbps, AAC 128 kbps, `+faststart`). No HLS in v1. | §10.4 |
| TD-009 | Derivative formats (OQ-10) | **WebP** for thumbnails and previews. No AVIF, and no JPEG fallback (every supported browser and OS decodes WebP). A ThumbHash placeholder per item. | §10.3 |
| TD-010 | Office previews (FR-PRC-006) | LibreOffice headless → PDF → same path as PDFs, in an isolated Lambda container. If conversion fails: icon and download only. | §10.5 |
| TD-011 | RAW previews (§10 PRD) | Use the embedded JPEG preview when it is ≥ 1600 px on the long edge. Otherwise decode with LibRaw. | §10.3 |
| TD-012 | Reverse geocoding (FR-PRC-010) | **Offline** reverse geocoder using the GeoNames `cities1000` dataset (CC-BY 4.0) bundled in the worker. No per-request cost, and no location leaves the AWS account (NFR-PRV-001). | §10.7 |
| TD-013 | Map basemap | MapLibre GL (web) / MapLibre Native (mobile) with a **self-hosted Protomaps PMTiles** basemap on S3 and CloudFront. No third-party tile server sees family locations. | §11.4 |
| TD-014 | Media delivery & signing (NFR-SEC-003) | CloudFront with **HMAC-tokenised URLs** checked by a CloudFront Function (secret in CloudFront KeyValueStore). Expiry is bucketed so URLs stay cache-stable. Vault content bypasses CloudFront (S3 presigned URLs from the Vault API role). | §8 |
| TD-015 | Vault KMS isolation (FR-VLT-006) | Separate KMS key. Only the Vault API role and the vault processing roles can decrypt. Vault metadata fields are also encrypted at the application layer. | §13 |
| TD-016 | Vault dedup leak (FR-DUP-005, PRD §12) | Blob identity is `(family, sha256, domain)`, where domain is `library` or `vault:{user}`. A library upload never sees vault blobs, so a vault-only match returns the same `upload_required` answer as a brand-new file. | §9.4 |
| TD-017 | Bootstrap (FR-FAM-001) | `phamoto-admin bootstrap` CLI calls an admin-only Lambda (IAM-authorised, not exposed through CloudFront). | §4.9 |
| TD-018 | IaC | **AWS CDK v2 (TypeScript)**. One CDK app, one stack per concern, deployed per environment account. | §3.3 |
| TD-019 | Mobile stack (OQ-03, PL-X-001) | **Flutter** for UI, with **native Swift/Kotlin modules** for the backup engine (PhotoKit and background `URLSession` on iOS; MediaStore and WorkManager on Android). Confirmed by a background-upload spike at the start of M10. | §21 |
| TD-020 | Minimum OS versions (PL-IOS-001, PL-AND-001) | iOS 17+ and Android API 29+ at the time of writing. Re-checked at M10 kickoff. | §21 |
| TD-021 | Certificate pinning (NFR-SEC-008) | **Not** used in v1 (risk of lock-out through CA rotation outweighs benefit). Rely on TLS 1.2+ and HSTS. Re-evaluate for SaaS. | §22.3 |
| TD-022 | Cross-region replication (OQ-04) | Off. The originals bucket is CRR-ready (versioning on, keys and names parameterised). | §7 |
| TD-023 | Large imported videos to Glacier IR (OQ-01 remainder) | No. Intelligent-Tiering only. | §7 |
| TD-024 | Edited iOS photos (OQ-05) | Store both (default kept). Data model has `item_variant.role ∈ {original, edited, motion, ...}`. | §6 |
| TD-025 | Legacy contact (OQ-08), mobile app lock default (OQ-09) | PRD defaults kept: no auto-share; app lock off. | — |
| TD-026 | Storage-class tracking source (FR-STO-006) | **S3 Inventory** (daily) is the baseline. **S3 Metadata** (live inventory and journal tables, which AWS stores as S3 Tables) is evaluated in M9 and replaces Inventory if it passes the criteria in §17.1.1. | §7.1, §17.1 |
| TD-027 | Vector store for future AI search (P3, PRD §13) | **S3 Vectors** is the planned vector store, one index per family, with visibility still enforced in the metadata store. Not built in v1. | §10.9 |

### 1.2 Principles that drive the design

1. **Serverless, scale-to-near-zero.** No always-on databases, containers, NAT gateways or VPCs (NFR-COST-002). Nothing in v1 runs inside a VPC.
2. **One authorisation module.** The visibility rule (PRD §11) is written once, in `packages/core/src/authz/visibility.ts`, and every read path uses it.
3. **Least privilege per stage.** Workers that parse untrusted files have no database access and no access to keys they don't need.
4. **Content-addressed, immutable storage.** Originals are keyed by hash and never rewritten. Derivatives are keyed by blob and spec version, and cached forever.
5. **The web app is just a client.** No server code is specific to the web; the Release 1 API is the Release 2 API.

---

## 2. Architecture overview

### 2.1 Component diagram

```
                           ┌──────────────────────────── phamoto.com (CloudFront: dist-web) ───────────────────────────┐
 Browser / iOS / Android ──┤  /*            → S3 web bucket (static Next.js export) + CF Function (route rewrite)      │
                           │  /api/v1/*     → API Gateway HTTP API → Lambda "api" (Hono)                                │
                           │  /api/v1/vault/* → API Gateway → Lambda "vault-api"                                       │
                           └────────────────────────────────────────────────────────────────────────────────────────────┘
                           ┌──────────────────────────── media.phamoto.com (CloudFront: dist-media) ───────────────────┐
                           │  /d/*  derivatives bucket   (cached, immutable, HMAC token via CF Function)                │
                           │  /o/*  originals bucket     (not cached, HMAC token, 15-min expiry)                       │
                           │  /x/*  exports prefix       (not cached, HMAC token)                                       │
                           │  /tiles/* basemap PMTiles   (cached, no token: tiles hold no family data, §11.4)          │
                           └────────────────────────────────────────────────────────────────────────────────────────────┘
 Vault content: S3 presigned URLs signed by vault-api role (direct S3, 5-min expiry)

 Upload:  client ──presigned UploadPart──▶ S3 staging (tmp/uploads)  ──complete──▶ api ──▶ SQS q-verify
          verify worker ──hash+copy──▶ originals/ (INTELLIGENT_TIERING) ──S3 event──▶ EventBridge ──▶ SQS q-dispatch
          dispatcher ──▶ q-image | q-video | q-video-long (Fargate) | q-pdf | q-office | q-other
          workers ──write derivatives──▶ derivatives bucket ;  ──result msg──▶ q-results ──▶ results-writer ──▶ DSQL

 Data:    Aurora DSQL (metadata)   ·  S3 (all bytes)  ·  CloudFront KeyValueStore (URL-signing keys)
 Jobs:    EventBridge Scheduler → Lambdas (trash purge, ref reconcile, inventory ingest, integrity check, notification batches)
 Email:   Amazon SES      Push (R2): APNs / FCM direct
 Ops:     CloudWatch Logs/Metrics/Alarms → SNS email to owner · AWS Budgets · X-Ray (sampled)
```

### 2.2 AWS services used

S3, CloudFront (+ Functions, KeyValueStore), API Gateway HTTP API, Lambda (arm64, Node.js LTS runtime and container images), ECS Fargate Spot, ECR, SQS, EventBridge (+ Scheduler), Step Functions (Standard), Aurora DSQL, KMS, SSM Parameter Store, SES, Route 53, ACM, CloudWatch, X-Ray, AWS Backup, AWS Budgets, IAM Identity Center (operator access).

Not used in v1: VPC, NAT, RDS instances, Cognito, MediaConvert, Rekognition, Amazon Location Service, WAF (see §22.3 for the WAF decision).

### 2.3 Domains

| Host | Purpose |
|---|---|
| `phamoto.com` | Web app and API (`/api/v1`). Same origin, so no CORS for the web client. |
| `www.phamoto.com` | 301 to apex. |
| `media.phamoto.com` | Derivatives, originals, exports and basemap tiles. Cookie-less. |
| `mail.phamoto.com` | SES custom MAIL FROM. SPF, DKIM (Easy DKIM) and DMARC `p=quarantine` on `phamoto.com`. |
| `dev.phamoto.com` | Dev environment (same layout, separate AWS account). |

---

## 3. Environments, infrastructure and delivery

### 3.1 Accounts and regions

- AWS Organizations with three accounts: `phamoto-mgmt` (billing, Identity Center, budgets), `phamoto-dev`, `phamoto-prod` (NFR-OPS-002).
- Primary region **us-west-2** (TD-001) for all regional resources: buckets, DSQL, Lambda, SQS, Fargate, SES, KMS, backups. Global resources (ACM for CloudFront, CloudFront Functions, KeyValueStore) in us-east-1.
- Operator access through IAM Identity Center with MFA. No IAM users. No long-lived access keys anywhere, including CI.

### 3.2 Repository layout (monorepo)

pnpm workspaces and Turborepo.

```
apps/
  web/                    Next.js (App Router, static export), TypeScript
  mobile/                 Flutter app (Release 2)
services/
  api/                    Hono app → Lambda "api"
  vault-api/              Hono app → Lambda "vault-api"
  jobs/                   Scheduled and queue-driven Lambdas (verify, dispatch, results-writer, purge, …)
  workers/
    image/                Dockerfile: libvips + libheif + libde265 + LibRaw + exiftool; Node handler
    video/                Dockerfile: ffmpeg (with zimg for tonemapping) + exiftool; handler for Lambda and Fargate
    pdf/                  Dockerfile: poppler-utils (pdftocairo, pdfinfo)
    office/               Dockerfile: LibreOffice headless
  admin-cli/              phamoto-admin (bootstrap, reprocess, reconcile, export-family)
packages/
  core/                   Domain logic: authz/visibility, dedup, item state machine, cost estimator
  db/                     Kysely schema types, migrations, DSQL connection (IAM token signer)
  contracts/              zod schemas → OpenAPI 3.1 → generated TS client and Dart client
  media-specs/            Derivative spec registry (versioned JSON + types)
  ui/                     Shared React components (Radix primitives + Tailwind tokens)
  i18n/                   Message catalogues (ICU), English only in v1
infra/
  cdk/                    CDK app: stacks per §3.3
docs/
  runbooks/               Operational runbooks (restore, key rotation, reprocess, incident)
PRD.md
TECH_SPEC.md
```

### 3.3 CDK stacks (TD-018)

| Stack | Contents | Notes |
|---|---|---|
| `Foundation` | KMS keys, SSM parameters, SNS alarm topic, Budgets | Termination protection on |
| `Storage` | Buckets (§7), lifecycle, Intelligent-Tiering config, inventory, EventBridge notifications | Retain on delete |
| `Data` | DSQL cluster, AWS Backup plan | Retain on delete |
| `Edge` | ACM certs, CloudFront distributions, CF Functions, KeyValueStore, Route 53 records, response-headers policies | us-east-1 parts in a cross-region stack |
| `Api` | HTTP API, `api` and `vault-api` Lambdas, admin Lambda | |
| `Pipeline` | SQS queues and DLQs, workers (container Lambdas), Fargate cluster and task defs, dispatcher, results-writer | |
| `Jobs` | Schedulers, Step Functions (export, member removal, reprocess) | |
| `Messaging` | SES identities, configuration set, templates | |
| `Observability` | Dashboards, alarms, log retention | |

Stateful resources (buckets, keys, DSQL) MUST use `RemovalPolicy.RETAIN`, and their logical IDs MUST NOT change across refactors.

### 3.4 CI/CD (NFR-OPS-004, NFR-SEC-010)

- GitHub Actions. Deploys assume an IAM role per account through GitHub OIDC (no stored AWS keys).
- On PR: typecheck, lint (zero warnings), unit tests, integration tests (Postgres container, §24), contract tests, Playwright smoke against a preview build, `cdk synth` and `cdk diff`, dependency audit (`pnpm audit` plus OSV-Scanner), container scan (Trivy) of worker images.
- On merge to `main`: deploy to dev, run the E2E suite against `dev.phamoto.com`. Promotion to prod is a manual approval on a tagged release.
- Database migrations run as a separate pipeline step before Lambda deploys (§6.6).

---

## 4. Identity, authentication and sessions (`FR-AUTH`, `FR-FAM`)

### 4.1 Model

- **User** is the Phamoto account. **Identity** is one sign-in method linked to a user: `google` (OIDC `sub`), `apple` (OIDC `sub`), `passkey` (WebAuthn credential ID). A user may link several identities (FR-AUTH-005).
- **TOTP** is a second factor stored per user. **Recovery codes** are a per-user set of 10 single-use codes.
- **Session** is one signed-in device or browser. Sessions own a rotating refresh-token family.

### 4.2 Sign-in flows

| Method | Flow |
|---|---|
| Google | OAuth 2.0 Authorization Code + PKCE via `openid-client`. Require `email_verified = true`. Callback `/api/v1/auth/oidc/google/callback`. |
| Apple | Sign in with Apple (Services ID for web; native on iOS in R2). `form_post` response mode. Handle private-relay emails (§4.3). Needs an Apple Developer Program membership (~$99/yr, also needed for the iOS app). |
| Passkey | WebAuthn via `@simplewebauthn/server` and `@simplewebauthn/browser`. RP ID `phamoto.com`. Discoverable credentials, `userVerification: required`. Synced passkeys (iCloud Keychain, Google Password Manager) work across devices (FR-AUTH-003 AC). |
| TOTP | RFC 6238, SHA-1, 6 digits, 30 s step, ±1 step window, replay protection (store last used step). Secret encrypted with the `app-secrets` KMS key (envelope encryption). |

**MFA policy (FR-AUTH-004):**
- Passkey sign-in satisfies MFA on its own (phishing-resistant, user-verified).
- Google or Apple sign-in for a user with TOTP enabled requires a TOTP or recovery code before the session is issued.
- The owner MUST have a passkey or TOTP. Until one is registered, an owner session is limited to the security-setup routes.

### 4.3 Invite-only enforcement (FR-AUTH-006, FR-FAM-002/003)

- Every invite email contains a single-use link `phamoto.com/invite/{token}`. The token is 32 random bytes; only its SHA-256 is stored. Expiry 7 days.
- An invite is accepted only by following the link and completing a sign-in (Google, Apple or new passkey). Whatever identity completes that flow is bound to the invite, even when its email differs (handles Apple Hide My Email). Google identities must still have a verified email.
- A sign-in that does not come through an invite link and does not match an existing linked identity is rejected with "Phamoto is invite-only". **No database row is written.** The failure is logged to CloudWatch only (structured log, hashed identifier), which keeps FR-AUTH-006 ("creates no records") while still making abuse visible to alarms.

### 4.4 Tokens and sessions (FR-AUTH-008/009)

| Token | Format | Lifetime | Storage |
|---|---|---|---|
| Access token | JWT (ES256), claims: `sub`, `fam`, `sid`, `role`, `amr`, `iat`, `exp` | 5 min | Web: memory only. Mobile: memory. |
| Refresh token | Opaque, 32 random bytes; DB stores the SHA-256 | Rotates on every use. Web: expires after 30 days without use. Mobile: 90 days without use. Absolute max 365 days. | Web: `HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth` cookie. Mobile: Keychain / Keystore. |
| Vault elevation | Server-side state on the session (`vault_elevated_until`) | 5 min, sliding on vault activity | DB |

- JWT signing key: asymmetric KMS key (`ECC_NIST_P256`, `SIGN_VERIFY`). The API signs with `kms:Sign`; verifiers cache the public key. Key ID in the JWT header, so keys can rotate.
- **Refresh token reuse detection:** presenting an already-rotated refresh token revokes the whole session and writes an audit event.
- **Per-request session check:** after verifying the JWT, the API reads `session` and `app_user` by primary key and rejects the request if the session is revoked or the user is not `active`. This gives immediate lock-out on member removal (FR-FAM AC) at the cost of one indexed read per request. The 5-minute access-token TTL is the upper bound for any client that only checks locally.
- **CSRF:** the refresh cookie is `SameSite=Strict` and the refresh endpoint also requires the `X-Phamoto-Client` header. All other endpoints use `Authorization: Bearer`, so they are not cookie-authenticated.
- **Sign-out (FR-AUTH-009):** revokes the session server-side. Web clears memory, IndexedDB upload queue entries belonging to that user, and the service-worker caches. Mobile wipes the Keychain/Keystore, thumbnail cache, offline content and vault cache.
- **Sessions list:** `session` rows include device label (parsed user agent or app-reported device name), platform, created, last seen, and the last IP truncated to /24 (IPv4) or /48 (IPv6) for display. Full IP goes to the audit log only.

### 4.5 Step-up (vault elevation and sensitive actions)

`POST /v1/auth/step-up` accepts a WebAuthn assertion (preferred) or a TOTP code. It requires a credential that is not merely the session itself: Google/Apple re-sign-in alone does not count. On success it sets `session.vault_elevated_until = now + 5 min` and `session.step_up_at = now`.

Actions requiring a step-up within the last 5 minutes: open vault, export including vault, ownership transfer, member removal, confirming an account re-bind, viewing recovery codes regeneration, deleting the account, removing the last MFA method.

Users without a passkey or TOTP cannot open the vault; the UI asks them to set one up first.

### 4.6 Recovery (FR-AUTH-007)

- 10 codes of 10 base32 characters, generated server-side, shown once, stored as Argon2id hashes (`@node-rs/argon2`). Each code is consumed on use. Regenerating invalidates the old set.
- **Re-bind:** the owner issues a "recovery invite" for an existing member. The member completes sign-in with a new identity through the link; the account moves to `pending_rebind`. The owner confirms in-app (with step-up). Then all old identities and sessions of that user are revoked and the new identity is linked. Audit events at each step.
- **Owner recovery:** if the owner loses all methods and recovery codes, the operator runs `phamoto-admin rebind-owner` with AWS credentials (documented runbook). This is the break-glass path.

### 4.7 Rate limiting

- API Gateway route throttling: default 50 rps burst 100 per stage; auth routes 5 rps.
- Application limits on auth endpoints using a DSQL `rate_limit` table (fixed windows keyed by hashed IP and by identity): 10 failed sign-ins per identity per 15 min, 30 per IP per 15 min, 5 TOTP failures then a 15-minute lock.

### 4.8 Family membership (FR-FAM)

- Roles live in `membership(user_id, family_id, role)`. v1 enforces exactly one membership per user with a unique index on `user_id`, which is easy to drop later (FR-FAM-007).
- **Member removal (FR-FAM-004):** a Step Functions workflow:
  1. Revoke all sessions and set `membership.status = removed` (immediate, same transaction).
  2. Remove the user from `album_member`. Their albums stop being shared.
  3. Apply the chosen option: (a) transfer ownership of every non-vault item in batches of 500 (respecting the DSQL row limits, §6.1), writing one audit event per batch; (b)/(c) set `app_user.status = read_only_export` or `pending_delete` with `delete_after = now + 30 days`. In (b) the user can sign in only to the export screen.
  4. At `delete_after`, purge the user's items and vault through the normal permanent-delete path (§14), then anonymise the user row (keep the id for audit integrity).
- **Ownership transfer (FR-FAM-005):** owner (with step-up) proposes; recipient accepts (with step-up); both role changes happen in one transaction.

### 4.9 Bootstrap (FR-FAM-001, TD-017)

`phamoto-admin bootstrap --env prod --family-name "…" --owner-email …` runs on the operator's machine with Identity Center credentials. It invokes the `admin` Lambda (IAM-authorised function URL, no CloudFront route), which creates the family and a pre-accepted owner invite, then prints the invite link. The command refuses to run if a family already exists. The owner opens the link, signs in, and is forced through passkey/TOTP setup and recovery-code download.

---

## 5. API

### 5.1 Conventions (PL-X-002)

- Base path `https://phamoto.com/api/v1`. JSON over HTTPS. Breaking changes require `/v2`; additive changes do not.
- Schemas in `packages/contracts` (zod) are the source of truth. CI generates OpenAPI 3.1 and fails if the committed spec is stale. The TypeScript client (web) and Dart client (mobile) are generated from it.
- Errors: RFC 9457 `application/problem+json` with a stable `type` URI and a `code` string.
- Pagination: opaque cursor (`?cursor=…&limit=…`), keyset-based, never offset.
- Idempotency: every non-idempotent `POST` accepts `Idempotency-Key`. Responses are cached in `idempotency_key` for 24 h per user. Mobile background retries depend on this.
- IDs: UUIDv4 strings. Timestamps: RFC 3339 UTC. Capture times also carry `tz_offset_minutes`.
- CloudFront adds a secret `X-Origin-Verify` header; API Gateway requests without it are rejected by the Lambda (prevents bypassing CloudFront). Client IP comes from `CloudFront-Viewer-Address`.
- The Lambda handler is a single Hono app (a "lambdalith") to keep cold starts and deployment simple. Provisioned concurrency is not used (cost); target cold start < 800 ms with arm64, esbuild bundle, lazy SDK clients.

### 5.2 Endpoint catalogue (v1)

Grouped by resource. Every route enforces authorisation in `packages/core/src/authz`.

| Group | Endpoints |
|---|---|
| Auth | `POST /auth/oidc/{google,apple}/start`, `GET|POST /auth/oidc/{provider}/callback`, `POST /auth/passkey/{options,verify}`, `POST /auth/passkey/register/{options,verify}`, `POST /auth/totp/{setup,verify,disable}`, `POST /auth/mfa/verify`, `POST /auth/recovery-codes` (regenerate), `POST /auth/recovery/verify`, `POST /auth/refresh`, `POST /auth/step-up`, `POST /auth/sign-out`, `GET /auth/sessions`, `DELETE /auth/sessions/{id}`, `GET|DELETE /me/identities/{id}` |
| Invites & family | `GET /family`, `PATCH /family`, `GET /family/members`, `POST /family/invites`, `POST /family/invites/{id}/{resend,revoke}`, `GET /invites/{token}` (public, shows family name only), `POST /family/members/{id}/remove`, `POST /family/ownership-transfers`, `POST /family/ownership-transfers/{id}/{accept,decline}`, `POST /family/members/{id}/recovery-invite`, `POST /family/rebinds/{id}/confirm` |
| Uploads | `POST /uploads` (declare, §9.2), `POST /uploads/{id}/parts` (presign batch of part URLs), `GET /uploads/{id}` (status + completed parts), `POST /uploads/{id}/complete`, `DELETE /uploads/{id}` |
| Items | `GET /items/{id}`, `PATCH /items/{id}` (capture time/location override, description), `POST /items/batch` (multi-select actions), `POST /items/{id}/original-url`, `POST /items/{id}/full-render`, `POST /items/{id}/save-copy` (FR-ALB-008) |
| Library | `GET /library/segments` (counts per month for the scrubber and layout), `GET /library/items?month=YYYY-MM&scope=all|mine` , `GET /search` (filters, §11.2), `GET /documents` |
| Albums | `GET /albums?scope=mine|shared`, `POST /albums`, `GET|PATCH|DELETE /albums/{id}`, `GET /albums/{id}/items`, `POST /albums/{id}/items`, `DELETE /albums/{id}/items` (body: item IDs), `PUT /albums/{id}/order`, `PUT /albums/{id}/members`, `POST /albums/{id}/download` |
| Map & memories | `GET /map/clusters?bbox=&zoom=&filters`, `GET /map/items?cell=`, `GET /places` , `GET /places/{id}/items`, `GET /memories/today`, `POST /memories/{id}/save-as-album`, `POST|DELETE /memories/hidden` |
| Trash | `GET /trash`, `POST /trash/restore`, `POST /trash/delete-permanently`, `POST /trash/empty` |
| Vault (vault-api) | `GET /vault/items`, `GET /vault/items/{id}`, `POST /vault/move-in`, `POST /vault/move-out`, `POST /vault/uploads` (same shape as `/uploads`), `POST /vault/items/{id}/urls`, `POST /vault/keepalive`, `POST /vault/lock` |
| Import | `POST /imports`, `GET /imports/{id}`, `POST /imports/{id}/files` (register batch), `POST /imports/{id}/{pause,resume,finish}`, `GET /imports/{id}/report.csv` |
| Export | `POST /exports` (scope: selection, album, me, family), `GET /exports`, `GET /exports/{id}`, `POST /exports/{id}/parts/{n}/url`, `GET /exports/estimate` |
| Dashboard | `GET /dashboard/usage` (member or owner view by role), `GET /dashboard/cost` (owner), `GET /dashboard/processing` (owner), `POST /dashboard/processing/retry` |
| Audit | `GET /audit?actor=&action=&from=&to=` |
| Notifications | `GET /notifications`, `POST /notifications/read`, `GET|PUT /me/notification-prefs` |
| Settings & profile | `GET|PATCH /me`, `POST /me/avatar` (upload declare), `POST /me/delete` |
| Devices & sync (R1 API, used by R2) | `POST /devices`, `PATCH /devices/{id}`, `GET /sync/changes?cursor=`, `POST /sync/validate` (item IDs → still visible?), `POST /devices/{id}/asset-index` (batch upsert), `GET /devices/{id}/asset-index?cursor=` |
| Meta | `GET /health`, `GET /config/client` (limits, feature flags, derivative spec versions) |

### 5.3 Change feed (`/sync/changes`)

Needed by R2 offline sync (FR-OFF-003/004) and used by the web client to refresh open views.

- Every mutating transaction appends to `change_log(family_id, change_id, ts, entity, entity_id, op, scope_album_id, scope_user_id, payload)`. `change_id` is a time-ordered UUIDv7 generated in the API (DSQL has no sequences; ordering ties resolve by `(ts, change_id)`).
- `GET /sync/changes` returns entries after the cursor, filtered to what the caller can see, as upserts, plus **revocations**: for ops that can remove visibility (`album_item.remove`, `album.unshare`, `album_member.remove`, `item.trash`, `item.vault_in`, `membership.remove`), the server re-evaluates visibility for the affected items for this caller and emits `revoke` records for those no longer visible.
- Clients that fall more than 30 days behind (log retention) get `410 Gone` and do a full resync (`/sync/validate` over their cached IDs plus re-listing).

---

## 6. Data model and metadata store (TD-002)

### 6.1 Why Aurora DSQL

| Need | DynamoDB | Aurora Serverless v2 | Aurora DSQL |
|---|---|---|---|
| Visibility rule (joins across items, albums, members) | Fan-out tables or many queries | Native SQL | Native SQL |
| Multi-attribute search filters (FR-LIB-007) | Scans or many GSIs | Native SQL | Native SQL |
| Cost floor at year-1 scale | ~$0 | ~$45/month at 0.5 ACU min; scale-to-zero resumes in ~15 s (bad UX) | Pay per use, no idle cost |
| Runs without VPC/NAT | Yes | Data API only | Yes (IAM-token auth over public endpoint) |
| PITR | 35 days | Yes | Backups via AWS Backup; PITR to be confirmed |

DSQL is the only option that gives SQL without an idle cost or a cold-start penalty.

**DSQL constraints we design around:**
- No foreign-key enforcement, no triggers, no sequences. Integrity rules live in `packages/core` and are covered by tests; reconciliation jobs (§14.3) catch drift.
- Limits on rows modified per transaction (thousands, not millions). Bulk operations (ownership transfer, purge, share-with-family) run in batches of ≤ 500 rows.
- Optimistic concurrency: transactions can fail with a serialisation conflict at commit. The `db.tx()` helper retries up to 5 times with jitter. Every transaction body must be safe to re-run.
- One DDL statement per transaction, no mixing DDL and DML (§6.6).
- Limited extension support (no PostGIS, no `pg_trgm`). Geo uses geohash columns (§11.4); text search uses a token table (§11.2).
- Store JSON as `text` with app-side validation; do not depend on `jsonb` operators.

**[SPIKE S1] (M0, 3 days):** confirm current DSQL limits (rows per transaction, index count, supported types), backup/PITR capability, p95 latency for the library and search queries on a synthetic 300k-item dataset, and per-month cost at the §23 activity model. **Exit criteria:** NFR-PERF-004 met (p95 < 300 ms for list and search), restore tested, cost < $5/month at year-1 activity. **Fallback:** Aurora Serverless v2 PostgreSQL (min 0 ACU with auto-pause off in prod, ~$45/month), accessed through the RDS Data API so Lambdas stay outside a VPC. The Kysely layer and migrations are written to run unchanged on both.

### 6.2 Data access layer

- **Kysely** (type-safe query builder) with the `pg` driver. Connection uses a DSQL IAM auth token generated by `@aws-sdk/dsql-signer` (15-min tokens, refreshed per connection). One pooled connection per Lambda execution environment.
- Every query function takes a `Ctx { familyId, userId, sessionId, vaultElevated }`. A lint rule (custom ESLint) forbids queries on tenant tables that don't filter by `family_id` (NFR-SEC-004).

### 6.3 Schema (core tables)

Types abbreviated: `uuid`, `text`, `int8`, `ts` = `timestamptz`, `bool`. Primary keys first. "Idx" lists secondary indexes. All tenant tables carry `family_id`.

```sql
-- Tenancy & identity
family            (id uuid PK, name text, owner_user_id uuid, settings text /*json*/, created_at ts)
app_user          (id uuid PK, family_id uuid, display_name text, avatar_blob_id uuid, timezone text,
                   status text /* active|read_only_export|pending_delete|pending_rebind|deleted */,
                   delete_after ts, created_at ts)
membership        (user_id uuid, family_id uuid, role text /* owner|member */, status text, joined_at ts,
                   PK(user_id, family_id))                         -- Idx: UNIQUE(user_id) [v1 only], (family_id)
identity          (id uuid PK, user_id uuid, kind text /* google|apple|passkey */, subject text,
                   email text, public_key bytea, sign_count int8, transports text, created_at ts, last_used_at ts)
                                                                   -- Idx: UNIQUE(kind, subject), (user_id)
mfa_totp          (user_id uuid PK, secret_ciphertext bytea, last_step int8, created_at ts)
recovery_code     (user_id uuid, code_hash text, used_at ts, PK(user_id, code_hash))
invite            (id uuid PK, family_id uuid, email text, token_hash text, kind text /* member|recovery|owner_bootstrap */,
                   target_user_id uuid, invited_by uuid, status text, expires_at ts, created_at ts)
                                                                   -- Idx: UNIQUE(token_hash)
session           (id uuid PK, user_id uuid, family_id uuid, refresh_hash text, refresh_family uuid,
                   device_label text, platform text, device_id uuid, ip_display text,
                   created_at ts, last_seen_at ts, idle_expires_at ts, absolute_expires_at ts,
                   step_up_at ts, vault_elevated_until ts, revoked_at ts)
                                                                   -- Idx: UNIQUE(refresh_hash), (user_id)

-- Content
blob              (id uuid PK, family_id uuid, sha256 bytea, domain text /* library | vault:{user_id} */,
                   size int8, mime text, storage_key text, storage_class text, it_access_tier text,
                   kms_key text /* library|vault */, ref_count int8, state text /* active|deleting|deleted */,
                   first_item_id uuid, created_at ts, deleted_at ts)
                                                                   -- Idx: UNIQUE(family_id, domain, sha256)
item              (id uuid PK, family_id uuid, owner_user_id uuid,
                   type text /* photo|video|live|doc|other */,
                   state text /* uploading|uploaded|processing|ready|failed */,
                   visibility text /* normal|vault */,
                   filename text, filename_ct bytea /* vault: encrypted */, mime text, size int8,
                   capture_at ts, capture_tz_offset int, capture_source text /* exif|sidecar|mtime|override */,
                   capture_at_override ts, capture_tz_override int,
                   capture_local_date date, capture_md int /* MMDD of local date, for memories */,
                   lat float8, lon float8, alt float8, geo_source text /* exif|sidecar|override */,
                   geohash text, place_id int8,
                   width int, height int, duration_ms int8, page_count int,
                   camera_make text, camera_model text, description text, thumbhash text,
                   source text /* upload|import|backup|save_copy */, source_device_id uuid, import_job_id uuid,
                   processing_attempts int, processing_error text,
                   trashed_at ts, trashed_by uuid, pre_trash_state text /*json*/,
                   created_at ts, updated_at ts)
       -- Idx: (family_id, owner_user_id, visibility, trashed_at, capture_at DESC, id)
       --      (family_id, type, capture_at DESC)       (family_id, geohash)  (family_id, capture_md)
       --      (family_id, place_id)                   (family_id, state)    (family_id, trashed_at)
item_variant      (item_id uuid, role text /* original|edited|still|motion|sidecar */, blob_id uuid,
                   family_id uuid, PK(item_id, role))              -- Idx: (blob_id)
derivative        (blob_id uuid, kind text, spec_version int, family_id uuid, storage_key text,
                   url_secret text, bytes int8, format text, width int, height int, created_at ts,
                   PK(blob_id, kind, spec_version))
search_token      (family_id uuid, token text, item_id uuid, field text /* filename|camera|place|desc */,
                   PK(family_id, token, item_id, field))
place             (id int8 PK /* GeoNames id */, name text, admin1 text, country_code text, country text,
                   lat float8, lon float8)                          -- reference data, not tenant data

-- Albums & sharing
album             (id uuid PK, family_id uuid, owner_user_id uuid, name text, description text,
                   cover_item_id uuid, sort_mode text, share_whole_family bool, whole_family_perm text,
                   item_count int8, created_at ts, updated_at ts, deleted_at ts)
                                                                   -- Idx: (family_id, owner_user_id), (family_id, share_whole_family)
album_member      (album_id uuid, user_id uuid, family_id uuid, permission text /* viewer|contributor */,
                   added_at ts, PK(album_id, user_id))             -- Idx: (family_id, user_id)
album_item        (album_id uuid, item_id uuid, family_id uuid, added_by uuid, added_at ts,
                   sort_position text /* fractional index */, capture_at ts /* denormalised */,
                   PK(album_id, item_id))
       -- Idx: (album_id, capture_at DESC, item_id), (item_id), (album_id, sort_position)
trashed_album_link (item_id uuid, album_id uuid, family_id uuid, sort_position text, added_by uuid, added_at ts,
                   PK(item_id, album_id))                          -- restore target (FR-DEL-002)

-- Upload, import, export
upload_session    (id uuid PK, family_id uuid, user_id uuid, item_id uuid, sha256 bytea, size int8,
                   target text /* library|album:{id}|vault */, s3_upload_id text, staging_key text,
                   part_size int, state text, expires_at ts, created_at ts)
import_job        (id uuid PK, family_id uuid, user_id uuid, source text /* folder|takeout|icloud */,
                   status text, options text /*json*/, counts text /*json*/, report_key text,
                   created_at ts, updated_at ts)
import_file       (import_job_id uuid, rel_path_hash bytea, rel_path text, size int8, sha256 bytea,
                   status text /* pending|uploaded|duplicate|failed|skipped */, item_id uuid, reason text,
                   PK(import_job_id, rel_path_hash))
export_job        (id uuid PK, family_id uuid, user_id uuid, scope text, include_vault bool, status text,
                   parts int, bytes int8, manifest_key text, expires_at ts, created_at ts)

-- Devices (used from R2; table exists in R1)
device            (id uuid PK, user_id uuid, family_id uuid, platform text, name text, app_version text,
                   backup_settings text /*json*/, last_backup_at ts, push_token text, created_at ts)
device_asset      (device_id uuid, local_asset_id text, family_id uuid, sha256s text, item_id uuid,
                   status text, updated_at ts, PK(device_id, local_asset_id))

-- Memories, notifications, audit, ops
memory_hide       (user_id uuid, kind text /* date|item */, value text, family_id uuid, PK(user_id, kind, value))
notification      (id uuid PK, family_id uuid, user_id uuid, category text, payload text, created_at ts, read_at ts)
                                                                   -- Idx: (user_id, created_at DESC)
notification_pref (user_id uuid, category text, in_app bool, email bool, push bool, PK(user_id, category))
notification_outbox (id uuid PK, family_id uuid, kind text, key text, payload text, not_before ts, sent_at ts)
audit_event       (id uuid PK /* v7 */, family_id uuid, ts ts, actor_user_id uuid, action text,
                   target_type text, target_id uuid, target_owner_id uuid, scope text /* family|owner_only */,
                   session_id uuid, ip text, user_agent text, detail text /*json, no vault metadata*/)
       -- Idx: (family_id, ts DESC), (family_id, actor_user_id, ts DESC), (family_id, target_owner_id, ts DESC)
change_log        (family_id uuid, change_id uuid, ts ts, entity text, entity_id uuid, op text,
                   scope_album_id uuid, scope_user_id uuid, payload text, PK(family_id, change_id))
usage_snapshot    (family_id uuid, day date, dims text /* user|type|class|kind */, key text,
                   bytes int8, objects int8, PK(family_id, day, dims, key))
idempotency_key   (user_id uuid, key text, response text, status int, created_at ts, PK(user_id, key))
rate_limit        (bucket text PK, window_start ts, count int)
```

### 6.4 Visibility rule (PRD §11), single implementation

`packages/core/src/authz/visibility.ts` exports `visibleItems(ctx)`, a Kysely sub-query used by library, search, map, memories, album and sync reads:

```sql
-- Non-vault, non-trashed items U can see in family F
SELECT i.* FROM item i
WHERE i.family_id = :F AND i.trashed_at IS NULL AND i.visibility = 'normal'
  AND ( i.owner_user_id = :U
        OR EXISTS (
          SELECT 1 FROM album_item ai JOIN album a ON a.id = ai.album_id
          WHERE ai.item_id = i.id AND a.family_id = :F AND a.deleted_at IS NULL
            AND ( a.owner_user_id = :U
                  OR a.share_whole_family
                  OR EXISTS (SELECT 1 FROM album_member am WHERE am.album_id = a.id AND am.user_id = :U))))
```

- For the timeline, the planner-friendly form is a `UNION` of (a) own items by the owner index and (b) items reached through accessible albums by `album_item (album_id, capture_at)`, then keyset pagination on `(capture_at_effective DESC, id DESC)`. Both forms live in the same module and share one test matrix.
- **Vault items** are never returned by `visibleItems`. The Vault API has its own `vaultItems(ctx)` that requires `ctx.vaultElevated` and `owner_user_id = :U`.
- `canEditItem`, `canDeleteItem`, `canAddToAlbum`, `canManageAlbum`, `canRemoveFromAlbum` live next to it. Route handlers MUST NOT re-implement these checks.
- Required test matrix (PRD §17.2): owner, other member, album viewer, album contributor, whole-family share, removed member, vault item, trashed item, deleted album, member added after a whole-family share.

`capture_at_effective` is a stored column computed in application code as `coalesce(capture_at_override, capture_at)` whenever either changes (no generated columns are assumed).

### 6.5 Item state machine (FR-PRC-008)

```
uploading ──complete+verified──▶ uploaded ──dispatch──▶ processing ──results ok──▶ ready
    │                               │                       │
    └──verify mismatch──▶ (upload_session failed; client retries; item stays uploading)
                                                            └──3 attempts failed──▶ failed ──retry (owner)──▶ processing
```

- Transitions happen only in `packages/core/src/items/state.ts`. `failed` items are still downloadable (FR-PRC-008).
- An item is "backed up" for client purposes (NFR-REL-001, FR-FRE-001) once it is `uploaded`, `processing`, `ready` or `failed`, because the verified original is then stored.

### 6.6 Migrations

- Plain SQL files `packages/db/migrations/NNNN_name.sql`, one DDL statement per file (DSQL rule). A small runner records applied migrations in `schema_migration` and executes each file in its own transaction.
- Index creation uses DSQL's asynchronous index build and the runner waits for completion before continuing.
- Migrations must be backwards compatible with the previous Lambda version (expand → deploy → contract).

### 6.7 Backup and recovery (NFR-REL-003)

- AWS Backup plan: hourly backups of the DSQL cluster retained 7 days, daily retained 35 days, copied to a vault in the `phamoto-mgmt` account with a vault lock. If S1 shows native PITR, enable it with 35-day retention as well.
- **Metadata rebuild path:** originals carry object metadata (`x-amz-meta-family`, `-first-item`, `-first-owner`, `-filename` (URL-encoded), `-capture-at`). **Vault objects carry only `family`, `user` and `item` IDs**: no filename or capture time, because object metadata can surface in inventory and S3 Metadata tables (§17.1.1). A nightly job also writes a logical export (items, variants, albums, album items, members, overrides) as gzipped JSON Lines to `backups/{family_id}/{date}/` in the derivatives bucket (Standard, 35-day expiry). Runbook `docs/runbooks/restore-metadata.md` covers both paths.

---

## 7. Storage layout and S3 configuration (FR-STO, PRD §12)

### 7.1 Buckets

Separate buckets (rather than prefixes) so that encryption keys, lifecycle rules, Object Lock and IAM policies cannot leak across concerns.

| Bucket | Key layout | Storage class | KMS key | Versioning | Lifecycle |
|---|---|---|---|---|---|
| `phamoto-{env}-originals` | `o/{family_id}/{sha[0:2]}/{sha256}` | INTELLIGENT_TIERING (set on copy-in) | `library` | On | Non-current versions expire after 30 days (FR-STO-007). Delete markers cleaned. |
| `phamoto-{env}-vault` | `v/{family_id}/{user_id}/{sha256}` and `vd/{family_id}/{user_id}/{blob_id}/{kind}-v{n}.{ext}` | Originals: INTELLIGENT_TIERING; derivatives: STANDARD | `vault` | On | Same as originals |
| `phamoto-{env}-derivatives` | `d/{family_id}/{blob_id}/{url_secret}/{kind}-v{n}.{ext}`, `full/…` (on-demand renders), `backups/…` | STANDARD | `library` | Off | `full/` expire 30 days; `backups/` expire 35 days |
| `phamoto-{env}-staging` | `u/{family_id}/{upload_id}`, `x/{family_id}/{export_id}/part-{n}.zip`, `r/{family_id}/{job_id}/report.csv` | STANDARD | `library` | Off | Abort incomplete MPU after 7 days (CP-5). `u/` expire 7 days. `x/` and `r/` expire 7 days (FR-EXP-005). |
| `phamoto-{env}-web` | static site | STANDARD | SSE-S3 | Off | — |
| `phamoto-{env}-tiles` | `planet.pmtiles` | INTELLIGENT_TIERING | SSE-S3 | Off | — |
| `phamoto-{env}-audit-archive` | `a/{family_id}/{yyyy}/{mm}/{dd}.jsonl.gz` | STANDARD_IA | `library` | On, **Object Lock** governance, 1 year | Expire after 1 year + 1 day |
| `phamoto-{env}-logs` | CloudFront and S3 access logs (optional, off by default for cost) | STANDARD | SSE-S3 | Off | 30 days |

- All buckets: Block Public Access (all four settings), `BucketOwnerEnforced`, TLS-only bucket policy (`aws:SecureTransport`), SSE-KMS with **S3 Bucket Keys enabled** (cuts KMS request costs by ~99%).
- **Intelligent-Tiering configuration** on originals and vault: no `ARCHIVE_ACCESS` or `DEEP_ARCHIVE_ACCESS` tiers configured (FR-STO-004 AC). A CDK unit test and an AWS Config custom check assert this.
- Derivatives use `blob_id` (not `item_id`) because derivatives depend only on bytes: deduped items share derivatives with no extra cost. Item-level metadata (overrides) never affects derivative bytes.
- `url_secret` is 16 random bytes (base32) per derivative set. It is defence in depth: even with a valid family token, a URL can't be guessed.
- **S3 Inventory** daily (Parquet) on originals and vault buckets, fields: size, storage class, IntelligentTieringAccessTier, ETag. Consumed by the inventory job (§17.1) for FR-STO-006. May be replaced by S3 Metadata after the M9 evaluation (TD-026, §17.1.1).
- Storage classes and lifecycle rules are CDK config values in `infra/cdk/config/{env}.ts`. Changing them is an infra deploy, not an app release (CP-3, FR-STO-005).

### 7.2 KMS keys

| Alias | Type | Used for | Who can decrypt |
|---|---|---|---|
| `phamoto/library` | Symmetric | Originals, derivatives, staging, audit archive | verify worker, processing workers (library), export task, api (only to presign originals), CloudFront distribution `dist-media` (via key policy with `AWS:SourceArn`) |
| `phamoto/vault` | Symmetric | Vault bucket objects, vault metadata data keys | `vault-api` role, vault processing roles (separate function aliases), vault verify worker |
| `phamoto/app-secrets` | Symmetric | TOTP secrets, URL-signing secrets in SSM | api, vault-api |
| `phamoto/jwt` | Asymmetric ECC P-256 sign/verify | Access token signing | api (`kms:Sign`), all API functions (`kms:GetPublicKey`) |

Automatic annual rotation is on for symmetric keys. The JWT key is rotated by creating a new key and publishing both public keys for one access-token lifetime.

---

## 8. Media delivery (CP-1, CP-2, NFR-SEC-003, TD-014)

### 8.1 Tokenised CloudFront URLs

Media URLs look like:

```
https://media.phamoto.com/d/{family_id}/{blob_id}/{secret}/thumb-v1.webp?e={exp}&k={kid}&s={sig}
sig = base64url( HMAC-SHA256( key[kid], path + "|" + e ) )[0:22]
```

- A **CloudFront Function** (viewer-request) on `dist-media` checks `e > now`, looks up `key[kid]` in **CloudFront KeyValueStore**, and compares the HMAC in constant time. If it fails, it returns 403. Then it strips the query string. The cache policy keys on the path only, so every user hits the same CDN cache entry.
- **Expiry bucketing:** `e = ceil(now / W) * W + W` with window `W = 30 min` for derivatives. A URL is identical for everyone inside a window, so browsers and the CDN cache it, and it stays valid for 30–60 min. This is the maximum time a user keeps access to derivatives after losing access through the API (unshare, removal). This trade-off is deliberate and recorded here. `W` is config.
- HMAC signing costs microseconds, so the API can sign 500 thumbnail URLs per page without affecting NFR-PERF-004 (RSA-signed CloudFront URLs would cost ~1 ms each).
- Keys rotate monthly: new `kid` written to KeyValueStore and SSM; the old key stays valid for 2 windows.
- **Derivatives** (`/d/*`): cache policy TTL 1 year (objects are immutable per spec version), `Cache-Control: public, max-age=31536000, immutable` set on upload.
- **Originals** (`/o/*`): window 15 min (NFR-SEC-003), `CachingDisabled`, `Content-Disposition` from a signed `fn` parameter. Routing original downloads and exports through CloudFront uses the CloudFront free tier for egress (1 TB/month) instead of S3 internet egress.
- **Exports** (`/x/*`): same as originals. Export emails link to `phamoto.com/exports/{id}`, which asks the API for fresh part URLs. (S3/CloudFront URLs cannot live 7 days when signed with role credentials, so durable links must go through the app.)
- Range requests are supported for video proxies (progressive MP4 seeking) and PMTiles.

### 8.2 Vault delivery

Vault objects are never served by CloudFront. `vault-api` returns S3 presigned GET URLs (5-minute expiry, `response-content-disposition` set) signed with its own role credentials. Because only that role (and vault workers) can use `phamoto/vault`, a presigned URL produced by any other principal would fail at KMS. This satisfies FR-VLT-006 and the AC "derivative request without vault-elevated session returns 403": `vault-api` refuses to sign without elevation, and there is no other path.

Web vault views set `Cache-Control: no-store` on API responses. Vault images are rendered from object URLs (`blob:`) fetched with `cache: 'no-store'`, and revoked on lock.

### 8.3 Security headers

`dist-web` uses a response-headers policy: HSTS (`max-age=63072000; includeSubDomains; preload`), `X-Content-Type-Options`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` (camera allowed for document scan, geolocation denied), `frame-ancestors 'none'` and a CSP. Static export inlines bootstrap scripts, so the build step computes SHA-256 hashes for inline scripts and writes a per-page `<meta http-equiv="Content-Security-Policy">`; the header CSP carries the directives that meta tags can't (`frame-ancestors`). `connect-src 'self' https://media.phamoto.com`, `img-src 'self' blob: data: https://media.phamoto.com https://*.amazonaws.com` (vault presigned URLs), `media-src` likewise.

---

## 9. Upload pipeline (FR-UPL, FR-DUP, PL-WEB-006)

### 9.1 Client side (web)

- **Queue persistence:** IndexedDB (`idb`) stores the queue: file identity `(name, size, lastModified, relPath)`, hash, upload session ID, completed parts, target. On Chromium, `FileSystemFileHandle`/`FileSystemDirectoryHandle` objects are stored too, so the queue resumes after reload without re-selecting files. On Safari and Firefox, after a reload, the UI asks the user to re-select the same files or folder; matched files resume from their last completed part (FR-UPL-004).
- **Hashing:** SHA-256 streamed in a Web Worker with `hash-wasm` (WebCrypto cannot hash incrementally). Chunks of 8 MiB read via `Blob.slice()`, so memory stays bounded. A hash cache keyed by `(relPath, size, lastModified)` avoids re-hashing on resume.
- **Pipelining:** hashing runs one or two files ahead of uploading. Concurrency: 6 parts in flight on desktop, 3 on phones (configurable, via `/config/client`). File-level fan-out is capped so a 10,000-file folder never has more than ~50 files "active" in memory (FR-UPL AC).
- **Parts:** 16 MiB on desktop, 8 MiB on phones. Part size is chosen so `size / part_size ≤ 10,000` (S3 limit); at 20 GB the minimum is 2 MiB, so these defaults are fine. Each part carries `x-amz-checksum-crc32c` (computed with `hash-wasm`) so S3 rejects corrupted parts in transit.
- **Retry:** exponential backoff with jitter per part (max 8 attempts), then the file shows as failed with a retry action (FR-UPL-006).
- **Phone browsers (PL-WEB-006):** `<input type="file" multiple accept="image/*,video/*,application/pdf,…">`. On iOS, `accept` must not restrict to `image/jpeg` (forces conversion); the generic types keep HEIC/HEVC bytes intact when the user picks "Original" in the iOS picker options. `visibilitychange` pauses and resumes the queue cleanly, and a banner explains that keeping the tab open is faster and that automatic backup arrives with the apps.
- **Folder upload (FR-UPL-002):** `webkitdirectory` input plus `DataTransferItem.webkitGetAsEntry()` for drag-and-drop. "Create an album per top-level folder" groups by the first path segment.
- **Executables denylist (FR-UPL-010):** extensions and magic numbers (`.exe, .msi, .dll, .bat, .cmd, .com, .scr, .ps1, .sh, .app` bundles, ELF/PE/Mach-O headers), from config. Checked in the client for UX and again in the verify worker (authoritative).

### 9.2 Server side

**`POST /uploads`** (declare): body `{ sha256, size, mime, filename, rel_path?, last_modified?, target, client_ref, hints? }` where `hints` carries sidecar metadata from the importer (§15) and `live_pair` grouping (§10.6).

Decision, in one DSQL transaction:

1. Validate size ≤ max (20 GB, FR-UPL-007) and filename against the denylist.
2. Look up `blob` by `(family_id, domain, sha256)` where domain is `library` for library/album targets and `vault:{user}` for vault targets.
3. **Blob exists and is active:**
   - Same owner already has a non-trashed item with this blob as `original` → return that item (FR-DUP-002). If `target` is an album, add it to the album (with permission check).
   - Otherwise → create a new item referencing the blob (FR-DUP-003), `ref_count += 1`, copy extracted metadata from the blob's first item (EXIF-derived fields only; never overrides, description or album info from another user's item), state `ready` if derivatives exist. Response `{ status: "linked", item }`.
4. **No active blob** → create `item` (state `uploading`), `upload_session`, call `CreateMultipartUpload` on the staging bucket with `ChecksumAlgorithm=CRC32C`. Response `{ status: "upload_required", upload_id, part_size, part_count }`.

**`POST /uploads/{id}/parts`** returns presigned `UploadPart` URLs for a requested range of part numbers (max 100 per call, 60-minute expiry). **`GET /uploads/{id}`** calls `ListParts` so the client can resume after any failure.

**`POST /uploads/{id}/complete`** calls `CompleteMultipartUpload` and enqueues `q-verify`. The client shows "verifying". The item becomes "backed up" only when the verify worker confirms (NFR-REL-001).

### 9.3 Verify worker (FR-UPL-008)

Lambda (arm64, 3008 MB, 15-min timeout) for files ≤ 8 GB; larger files go to a Fargate task with the same code.

1. Stream the staging object with parallel ranged GETs (8 × 16 MiB in flight) into a sequential SHA-256 (Node `crypto`, which uses ARMv8 SHA instructions on Graviton). Count bytes.
2. If size or hash differ from the declaration: delete the staging object, mark the upload session `rejected`, notify the client (via `GET /uploads/{id}` status). The client re-hashes and retries once, then shows an error.
3. Sniff the real type from magic bytes; reject denylisted executables.
4. In one transaction: insert `blob` with `UNIQUE(family_id, domain, sha256)`. If another upload won the race, link this item to the existing blob instead (ref_count +1) and discard staging.
5. Copy staging → `originals/o/{family}/{sha[0:2]}/{sha}` with `StorageClass=INTELLIGENT_TIERING`, SSE-KMS, object metadata (§6.7). Use `CopyObject` for ≤ 5 GB and `UploadPartCopy` for larger objects. Copy only if the key doesn't already exist (`HeadObject`; the DB row is the real lock).
6. Set item `uploaded`, delete staging. Append `change_log`. Record `blob.storage_class`.
7. The S3 `ObjectCreated` event on the originals bucket triggers processing (§10).

Vault uploads use the same worker code under a separate function and role that writes to the vault bucket with the vault key.

### 9.4 Vault and dedup (FR-DUP-005, TD-016)

- Library uploads only ever look up `domain = 'library'`. A file that exists only in someone's vault is not found, so the response is `upload_required`, identical to a new file. The library then gets its own physical copy. Nothing about the vault is observable, including timing (both paths are the same code path).
- Vault uploads only look up `domain = 'vault:{user}'`. Dedup inside one user's vault still works.
- **Residual leak (documented, accepted by PRD):** cross-member dedup in the library reveals to the uploader that some family member already uploaded that exact file to the library. This is inherent to FR-DUP-003.

### 9.5 Live Photo and Motion Photo pairing during upload (FR-PRC-007)

- The web client groups files with the same base name and directory where one is a still (`.heic/.jpg`) and one is `.mov` (iCloud export and folder imports). Both are declared with a shared `live_pair` key. The server creates one item of type `live` with variants `still` and `motion`. Each file is uploaded and verified as its own blob.
- The image worker confirms the pair by comparing Apple's `ContentIdentifier` (MakerNotes in the still, `com.apple.quicktime.content.identifier` in the MOV). On mismatch, results-writer splits them into two items and logs an audit-free system note on the item.
- Android Motion Photos (JPEG with embedded MP4, XMP `GCamera:MotionPhoto`/`MicroVideoOffset` or `Container:Directory`) are one blob. The image worker extracts the MP4 to build the motion proxy.

---

## 10. Processing pipeline (FR-PRC)

### 10.1 Flow (TD-006, TD-007)

```
originals ObjectCreated ─▶ EventBridge rule ─▶ q-dispatch ─▶ dispatcher Lambda
   dispatcher: load blob + item kind (DB), choose route, send job {blob_id, family_id, key, kind, spec_versions}
       ├─▶ q-image        → image worker (Lambda container)
       ├─▶ q-video        → video worker (Lambda container), duration ≤ 3 min (config)
       ├─▶ q-video-long   → launcher Lambda → ECS RunTask (Fargate Spot, 4 vCPU/8 GB) → same image
       ├─▶ q-pdf          → pdf worker
       ├─▶ q-office       → office worker (LibreOffice) → hands PDF to q-pdf
       └─▶ q-other        → metadata-only (size, mime, icon)
   workers ─▶ write derivatives to derivatives bucket ─▶ send result message to q-results
   q-results ─▶ results-writer Lambda (has DB access) ─▶ derivative rows, item metadata, place, tokens, state
```

- Workers have **no DB access**, read access only to the source key prefix, and write access only to `d/{family}/{blob_id}/*`. A compromised decoder can't touch the database or other families' derivatives (NFR-SEC-009).
- Every job is idempotent: derivative keys are deterministic per `(blob, kind, spec_version)`. Re-running overwrites with identical output; results-writer upserts.
- SQS settings: visibility timeout = 6 × worker timeout, `maxReceiveCount = 3`, then DLQ (FR-PRC-008: 3 attempts with backoff). A DLQ consumer marks the item `failed` with the error and raises the dashboard flag. DLQ depth > 0 alarms (NFR-REL-005).
- **Concurrency caps** (reserved concurrency): image 40, video 8, pdf 10, office 2, Fargate tasks 4. These bound cost and DSQL write rate during big imports and are config.
- Processing applies to blobs. When an item links to an existing blob, the dispatcher is not invoked; results-writer copies extracted metadata onto the new item immediately.

### 10.2 Metadata extraction (FR-PRC-002)

- `exiftool -json -n -G1 -api LargeFileSupport=1` for every type (photos, RAW, video, PDF). `ffprobe` for video stream details.
- **Capture time precedence:** `SubSecDateTimeOriginal` / `DateTimeOriginal` + `OffsetTimeOriginal` → QuickTime `CreationDate` (has offset) → QuickTime `CreateDate` (UTC) → GPS date/time → importer sidecar (§15) → file `lastModified` declared by client → upload time. `capture_source` records which one won.
- **Time zone:** use `OffsetTimeOriginal` if present; otherwise derive from GPS coordinates with a bundled timezone boundary dataset (`geo-tz`); otherwise use the uploader's profile time zone and mark it inferred.
- Orientation is applied to derivatives, and `width`/`height` are stored post-rotation.

### 10.3 Image derivatives (FR-PRC-003/004, TD-009, TD-011)

Spec registry `packages/media-specs/specs.json` (versioned, FR-PRC-009):

| Kind | Size | Format | Target |
|---|---|---|---|
| `thumb` | 256 px short edge | WebP q70 | ~10–20 KB |
| `thumb2x` | 512 px short edge | WebP q65 | ~30–60 KB |
| `preview` | 2048 px long edge (config 1080–2048) | WebP q80, adaptive q to stay ≤ 500 KB | 200–500 KB |
| `full` | Original resolution, on demand only (FR-LIB-005 for HEIC/RAW on web) | WebP q90 (or JPEG q92 if > 16383 px, WebP's limit) | — |
| `thumbhash` | ~25 bytes, stored in `item.thumbhash` | — | — |

- libvips (via `sharp` built against a custom libvips with libheif + libde265 for HEIC, and LibRaw for RAW) in a Lambda container. HEIC licensing: libde265 (LGPL) dynamically linked; documented in the open-source licences page (FR-SET-005).
- Animated GIF/WebP: `preview` stays animated (WebP animation) up to 300 frames; `thumb` is the first frame.
- Wide-gamut and HDR stills: convert to sRGB for derivatives (embed ICC). Gain-map HDR is not preserved in v1 derivatives.
- Memory: 4096 MB Lambda handles up to ~200 MP images; larger images go to the Fargate path.

### 10.4 Video (FR-PRC-005, TD-008)

- `poster`: frame at 1 s (or 10% of duration if shorter), then `thumb`, `thumb2x`, `preview` from it.
- `proxy`: `ffmpeg -c:v libx264 -profile:v high -preset veryfast -crf 23 -maxrate 2.5M -bufsize 5M -vf scale=-2:'min(720,ih)' -c:a aac -b:a 128k -movflags +faststart`. 60 fps sources are dropped to 30 fps.
- **HDR** (HLG/PQ, Dolby Vision profile 8.4 from iPhones): tone-map to SDR BT.709 with `zscale` + `tonemap=hable`, detected from `color_transfer` in ffprobe.
- Limits: proxies are generated for videos up to 4 hours (PRD §10). Longer videos get poster + download only.
- Lambda route uses 10,240 MB (6 vCPU). Expected: a 1-min 4K HEVC clip transcodes in ~60–120 s, well within the 10-min p95 AC. Fargate Spot route for anything over 3 minutes; on Spot interruption the task re-queues (job is idempotent).
- **Why not MediaConvert:** comparable or higher per-minute cost during the bulk import, HDR tone-mapping needs the professional tier, and the ffmpeg path keeps one toolchain for Lambda and Fargate.

### 10.5 Documents (FR-PRC-006, TD-010)

- PDF: `pdfinfo` for page count and metadata. `pdftocairo` renders page 1 → `thumb`, `thumb2x`. Pages 2..N rendered eagerly up to `eager_pages` (config, default 10) as `page_{n}` WebP at 1600 px long edge; further pages rendered on demand through `POST /items/{id}/pages/{n}` (enqueues, client polls). This meets the 50-page AC (first-page thumbnail within 60 s, paging through on demand).
- Encrypted PDFs: no previews, icon only; the item is still `ready`.
- Office (`docx, xlsx, pptx, odt, ods, odp, rtf`): `soffice --headless --convert-to pdf` in a separate container with 2 GB memory, 120 s timeout, no network (function has no VPC and no egress need, but we also set `HOME` to `/tmp` and disable macros via profile). The resulting PDF goes to `vd`/`d` as `doc_pdf` and is then processed like a PDF. TXT and CSV previews: first 200 lines rendered to an image by the pdf worker.
- **Sandboxing (NFR-SEC-009):** each tool runs as a child process with `prlimit` (CPU seconds, address space, file size), a scratch dir under `/tmp` wiped after each job, and a per-job timeout. Lambda's Firecracker isolation separates functions; a dedicated function per parser family means one compromised parser can't reach other roles.

### 10.6 Live and Motion photos (FR-PRC-007)

- `motion` derivative: MP4 H.264, ≤ 1080 px, from the `.mov` (Live) or embedded MP4 (Motion Photo). Viewer plays it on hover (web) or press-and-hold (mobile).
- The `live` item's thumbnails come from the `still` variant.

### 10.7 Reverse geocoding (FR-PRC-010, TD-012)

- Bundled GeoNames `cities1000` (~150k places with admin1 and country names), loaded into a k-d tree at cold start (~20 MB). Nearest city within 50 km; beyond that, country only (by nearest point in the dataset, marked low confidence).
- Results cache: `(round(lat, 2), round(lon, 2))` → `place_id` in an in-memory LRU per worker. No external call, so no paid cache is needed.
- `item.geohash` (precision 9) is stored for map clustering. `place` rows are upserted from the dataset on first use. Search tokens get the place name, admin1 and country.
- Attribution "Contains data from GeoNames (CC-BY 4.0)" on the About page.

### 10.8 Re-processing (FR-PRC-009)

- `phamoto-admin reprocess --family F | --item I | --all [--kinds thumb,preview] [--spec-version N]` starts a Step Functions workflow that pages through blobs whose derivatives are below the target spec version and enqueues dispatch jobs at a controlled rate.
- New derivatives get new keys (`-v{n}`). results-writer switches `derivative` rows atomically, then a cleanup job deletes old-version objects after 7 days (so open clients keep working).
- Originals are only read, never written (FR-PRC AC).

### 10.9 Extensibility (FR-PRC-011)

The dispatcher reads a stage list per item kind from `media-specs`. Adding a stage (labels, OCR, embeddings) means adding a queue, a worker and a result handler; no change to existing stages.

#### 10.9.1 Planned vector store: S3 Vectors (TD-027, P3 — not built in v1)

When AI search (PRD §13) is built, embeddings go to **Amazon S3 Vectors**: vector buckets holding vector indexes, queried by similarity through the S3 Vectors API, billed for storage and queries, with no cluster to run. At one family's scale (≤ 300k vectors) this costs far less than OpenSearch Serverless or a hosted vector database, which matters against NFR-COST-001.

Design rules fixed now, so v1 doesn't block them:

- **Layout:** one vector bucket per environment (`phamoto-{env}-vectors`), encrypted with `phamoto/library`; one vector index per family (`fam-{family_id}`), so tenants are isolated by index and a family can be deleted by dropping its index (G6).
- **Vector key** = `blob_id`, not `item_id`. Embeddings depend only on bytes, so deduped items share one vector, like derivatives (§7.1). Vector metadata holds only `blob_id`, `type` and `capture_year`. No owner, filename, album or location, because those depend on the item and on sharing, which change.
- **Embedding stage:** a new pipeline stage `q-embed`, fed from the `preview` derivative (images, video posters) or extracted text (documents, once OCR exists). The model is a multimodal embedding model on Amazon Bedrock, chosen when the feature is built. Spec version recorded per vector (like `spec_version` for derivatives), so a model change is a re-process run (§10.8).
- **Visibility stays in the metadata store.** A search runs the vector query for top-K candidates (K ≈ 5–10× the page size), then filters the candidate `blob_id`s through `visibleItems` (§6.4) in DSQL. Vectors never decide who can see what. If too few visible results survive, the query widens K once.
- **Vault:** vault items are **never embedded** in the first version of the feature. Moving an item into the vault does not delete the shared blob's vector if other library items still reference it; if the library blob's `ref_count` reaches 0 (§14.2), the blob delete worker deletes its vector too.
- **Deletion:** blob deletion (§14.2) deletes the vector. The weekly reconciliation job (§14.3) also lists vector keys and alarms on any vector without an active library blob.
- **Before building:** verify S3 Vectors' current availability in us-west-2, the per-index vector limit, query latency (target p95 < 300 ms including the DSQL filter) and pricing.

---

## 11. Library, search, map and memories

### 11.1 Timeline (FR-LIB-001/002/003, NFR-PERF-001/003)

- **`GET /library/segments?scope=all|mine`** returns `[{ month: "2025-10", count: 412 }, …]` for every month with items (≈ 300 rows for 25 years). The client computes the total scroll height and the scrubber from this, without loading items. Computed with a `GROUP BY date_trunc('month', capture_at_effective)` over `visibleItems`, cached per user for 60 s.
- **`GET /library/items?month=…`** returns all items of that month (or pages of 500) as compact rows: `id, type, w, h, capture_at, tz, thumbhash, duration_ms, owner (short id), thumb_url, thumb2x_url`. Response size ~250 bytes/item, gzip/brotli.
- The client renders a justified-rows virtualised grid (`@tanstack/react-virtual`). Months near the viewport are fetched ahead. ThumbHash fills cells instantly, so there are no blank cells while thumbnails load (FR-LIB AC).
- Zoom levels: day (large tiles, `thumb2x`), month (default, `thumb`), year (small tiles, `thumb`). Ctrl/Cmd + scroll and a slider on web.

### 11.2 Search and filters (FR-LIB-007)

- `GET /search` filters: `from`, `to`, `type[]`, `album`, `owner`, `place`, `camera`, `q` (free text over filename, camera, place, description).
- Structured filters translate to SQL predicates on `visibleItems` using the indexes in §6.3.
- `q` is tokenised (lowercase, Unicode NFKD, strip diacritics, split on non-alphanumerics; filenames also split on `_`, `-` and camel-case; tokens ≥ 2 chars). Prefix match on `search_token.token` (`token >= :q AND token < :q || '￿'`), intersected per token. Results ordered by capture time.
- No relevance ranking in v1; AI search is P3.

### 11.3 Documents view (FR-LIB-008)

`GET /documents?sort=name|type|size|date&dir=…` over `visibleItems` where `type = 'doc'` or (`type = 'other'` and not media), with album names joined (first 3 plus count).

### 11.4 Map and places (FR-MEM-001/002, TD-013)

- `GET /map/clusters?bbox=…&zoom=z` maps zoom to a geohash precision (z ≤ 3 → 2 chars … z ≥ 14 → 7 chars) and returns `GROUP BY substr(geohash, 1, p)` with count, centroid and a representative item thumbnail URL, filtered by bbox (`lat/lon BETWEEN`) and `visibleItems`. Items without coordinates are excluded; manual locations count (FR-MEM AC).
- Clicking a cluster calls `GET /map/items?cell=…` (paged grid).
- **Basemap:** Protomaps planet PMTiles (vector, maximum zoom 14 to keep the file around 60–100 GB; verify size in M8) in the tiles bucket, served via `dist-media` at `/tiles/*` with range requests and long caching. MapLibre GL JS reads it with the `pmtiles` protocol. Tiles hold no family data, so `/tiles/*` needs no token.
- Places list: `GET /places` returns country → admin1 → city with counts, from `item.place_id` joined to `place`.

### 11.5 On this day (FR-MEM-003/004/005)

- Uses `item.capture_md` (MMDD of the capture's local date) where `capture_local_date < today` in the **viewer's** time zone (`app_user.timezone`, updated from the client). AC: 7 Oct shows only items captured on 7 Oct of earlier years.
- Excludes `memory_hide` entries. Grouped by year. "Save as album" creates an album with those items (permission rules apply: the viewer can only add items they own or reach through sharing, and the album stays private by default).
- Optional email (default off): a scheduled job at 08:00 local time per user with at least 1 memory sends a digest with up to 6 thumbnails (tokenised URLs with a 24-hour window).

---

## 12. Albums and sharing (FR-ALB)

- Album membership is `album_member` rows plus `share_whole_family` (with `whole_family_perm`). Whole-family sharing is evaluated at read time, so future members see it automatically (FR-ALB-005).
- **Add items:** owner of the album can add any item they can see; a contributor can add only items they own (FR-ALB AC). `album_item.capture_at` is denormalised and kept in sync when an item's capture time is overridden (same transaction, batches of 500 albums max per item, which is far above real counts).
- **Remove items:** album owner can remove any item; contributors only their own (FR-ALB-007). Removing never deletes the item.
- **Sort:** capture date asc/desc and date added use indexes; manual order uses fractional indexing (`sort_position` strings), so reordering touches one row.
- **Cover image:** explicit `cover_item_id`, or the newest item if unset or no longer visible.
- **Unshare** deletes `album_member` rows (or clears the whole-family flag) in one transaction and writes `change_log` entries; the next API fetch reflects it immediately (FR-ALB-006). Media URL residual window per §8.1.
- **Save a copy (FR-ALB-008):** creates an item owned by the caller pointing at the same blob (`source = 'save_copy'`, `ref_count + 1`). Free in storage.
- Vault items can't be added (FR-ALB-011): the album handler rejects `visibility = 'vault'`, and vault items are invisible to `visibleItems` anyway.
- `item_count` and `updated_at` on `album` are maintained in the same transaction as membership changes.
- **Notifications** for "new items in shared album" go through `notification_outbox` with key `album:{id}` and `not_before = now + 1h`, so at most one per album per hour (FR-NOT-001).

---

## 13. Private vault (FR-VLT, TD-015)

- **Opening:** web calls `POST /auth/step-up` (passkey or TOTP) → `vault_elevated_until = now + 5 min`. `vault-api` requires it on every request and extends it on activity (`/vault/keepalive` every 60 s while the vault UI is visible).
- **Auto-lock (FR-VLT-002):** client locks after 5 min inactivity or when `document.visibilityState` has been `hidden` for > 60 s; it calls `/vault/lock`, which clears the elevation server-side, and drops all vault data from memory. The server-side sliding window enforces the 5 min even if the client misbehaves.
- **Move in (FR-VLT-004):** confirmation lists affected shared albums (`album_item` joined to shared albums). Then a job:
  1. Copies the original to `vault/v/{family}/{user}/{sha}` (re-encrypted with `phamoto/vault`) if this user's vault doesn't already hold that hash, and creates or links the `vault:{user}` blob.
  2. Generates vault derivatives (vault worker functions) or copies existing derivatives with re-encryption.
  3. Switches `item_variant` to the vault blob, sets `visibility = 'vault'`, removes all `album_item` rows (not kept anywhere: moving the item out of the vault later does not restore its album membership), decrements the library blob's `ref_count`.
  4. Encrypts `filename`, `description` and location into `filename_ct` etc. with a per-user data key (wrapped by `phamoto/vault`), and nulls the plaintext columns.
  The item disappears from every library view as soon as step 3 commits; the UI shows "moving to vault" until then.
- **Move out:** reverse path. Dedup against the library domain happens server-side and is not signalled.
- **Direct vault upload (FR-VLT-005):** `POST /vault/uploads`, separate staging prefix `u/{family}/vault/…`, separate verify function, vault domain. Never touches library code paths.
- **Owner's view (FR-VLT-003):** dashboard shows per-member vault `bytes` and `count` only, from `usage_snapshot` (dims `vault_user`). No vault item metadata is ever exposed to anyone else.
- **Audit (FR-VLT-008):** `vault.open`, `vault.view` (per item opened in the viewer, not per thumbnail), `vault.download`, `vault.move_in`, `vault.move_out`, `vault.upload`. Scope `owner_only`, and `detail` contains item ID only (no filename) so the family audit view never shows titles (FR-VLT AC).
- **Notifications** never reference vault items (FR-NOT-004): the notification builder rejects any payload referencing a vault item ID (unit-tested).
- **Exports including vault** require step-up and run under a separate export task role that can use the vault key.

---

## 14. Trash, deletion and reference counting (FR-DEL, FR-DUP-004)

### 14.1 Trash

- **Delete (FR-DEL-001):** owner only. Sets `trashed_at`, `trashed_by`, moves the item's `album_item` rows into `trashed_album_link`, and saves `pre_trash_state` (visibility). Trashed items vanish from `visibleItems` immediately.
- **Restore (FR-DEL-002):** reverses that, re-adding album links only for albums that still exist and where the owner still has permission to add. Vault items restore into the vault.
- **Permanent delete / empty trash (FR-DEL-003)** and **automatic purge after 30 days** (daily scheduled job, FR-DEL-004) both call `purgeItem()`.

### 14.2 `purgeItem()` (crash-safe)

In one transaction per item (batches of 100 items per job invocation):
1. Delete `item_variant`, `trashed_album_link`, `search_token`, `memory_hide` rows for the item; mark `item` row deleted (hard delete of the row after the audit event is written).
2. For each variant blob: `UPDATE blob SET ref_count = ref_count - 1`. If it reaches 0, set `state = 'deleting'` and enqueue `q-blob-delete`.

The `blob` row is written by both linkers (`ref_count + 1`) and purgers (`- 1`). With DSQL optimistic concurrency, two concurrent writers on the same blob conflict and one retries, so a dedup link cannot race with a deletion. A linker that sees `state = 'deleting'` treats the blob as missing (new upload).

**Blob delete worker:** waits until the blob has been `deleting` for 24 h, re-checks `ref_count = 0` and no `item_variant` rows reference it, then deletes the original object and all derivative objects, and sets `state = 'deleted'`. Originals bucket versioning keeps the non-current version for 30 days (protection against bugs, R6).

### 14.3 Reconciliation (FR-DUP-004)

Weekly job: recompute `ref_count` from `item_variant` for every blob, compare, alarm on any mismatch and fix (only upwards automatically; downward corrections to zero go to a review list in the owner's processing health panel, never auto-delete). Also checks for orphans: objects in S3 inventory without a `blob` row (alarm only) and `blob` rows without an object (alarm, mark item `failed`).

---

## 15. Bulk import (FR-IMP)

The importer is a web feature built on the upload pipeline. All parsing happens in the browser (files are local).

### 15.1 Flow

1. User picks a source type (Folder, Google Takeout, iCloud export) and drops the folder.
2. **Pre-scan** (in a worker): walk the tree, classify media, sidecars and album structure; hash files (cached); call `POST /imports/{id}/files` in batches of 500 with `(rel_path, size, sha256)`. The server answers which hashes are already present for this user (FR-IMP-004) without creating items. The UI shows counts, total size, detected duplicates, and the estimated one-time processing cost (NFR-COST-001, §23.3).
3. **Run:** uploads with hints (capture time, GPS, description, album names, live pairs). Progress is stored server-side in `import_file`, so pause/resume works across days and devices (FR-IMP-006). Limit 50,000 files per job (config).
4. **Report (FR-IMP-005):** a scheduled finaliser writes `report.csv` (path, status, item ID, reason) to staging (`r/`), downloadable for 7 days and re-generable on demand.

### 15.2 Google Takeout (FR-IMP-002)

Sidecar matching for each media file `X` in a folder, first match wins:
1. `X.supplemental-metadata.json`, and its truncated variants (Takeout truncates the full sidecar name to 51 characters, e.g. `X.supplemental-metad.json`, `X.suppl.json`).
2. `X.json` and the legacy truncation to 47 characters before `.json`.
3. Duplicate-suffix files `X(1).jpg` → `X.jpg(1).json` / `X.jpg.supplemental-metadata(1).json`.
4. Edited copies `X-edited.jpg` (and localised suffixes such as `-bearbeitet`, `-modifié`) use the original's sidecar and become the `edited` variant of the same item.

Fields: `photoTakenTime.timestamp` (UTC epoch) → capture time; `geoData` (fall back to `geoDataExif`; ignore 0,0) → location; `description`. Albums: every folder that isn't `Photos from YYYY` and has a `metadata.json` with an album title becomes an album; items in both a year folder and an album folder are the same hash, so they dedup and simply get album membership.

Per FR-IMP-001, embedded EXIF capture time wins over the sidecar when present. The sidecar fills gaps (Takeout often strips EXIF).

### 15.3 iCloud export (FR-IMP-003)

- Files from iCloud.com downloads: EXIF/QuickTime metadata only; Live Photo pairs by base name (§9.5).
- Apple data-privacy export (`iCloud Photos Part N of M`): parse `Photos/Photo Details*.csv` (`imgName`, `originalCreationDate`, `importDate`, `favorite`, `hidden`, `deleted`); skip `deleted = yes`. Parse `Albums/*.csv` (one CSV per album listing image names) to recreate albums. Memories and Shared Albums folders map to albums when present.

### 15.4 Folders and NAS (FR-IMP-001)

Capture time precedence: embedded metadata → XMP sidecar (`X.xmp`, `X.XMP`) → `lastModified`. Optional "Create an album per top-level folder".

---

## 16. Download and export (FR-EXP)

- **Single original (FR-EXP-001):** `POST /items/{id}/original-url` → tokenised `/o/*` URL (15 min) with `Content-Disposition: attachment; filename*=UTF-8''…`. For edited items, a `variant` parameter chooses `edited` or `original`. Audited (FR-AUD-005). If the user has "strip location on download" set (NFR-PRV-002) and the file is JPEG/HEIC/PNG/MP4/MOV, the API instead creates a small export job that rewrites the copy with `exiftool -gps:all=` (originals are never modified).
- **Album / selection / full / family exports (FR-EXP-002/003/004):** Step Functions workflow → Fargate Spot task (`export` image, Node + `archiver` with zip64) that streams originals from S3 into multipart uploads of `x/{family}/{job}/part-{n}.zip`, splitting at 4 GB (config). Layout `YYYY/MM/{filename}` with collision suffixes, plus `manifest.json` and `manifest.csv` (item ID, owner, original filename, hash, capture time, overrides, location, albums, type). Store-only (no compression) for media, deflate for documents and manifests.
- Notification (in-app and email) with a link to `phamoto.com/exports/{id}` (§8.1). Artefacts expire after 7 days (lifecycle, FR-EXP-005).
- **Estimate (FR-EXP-005):** `GET /exports/estimate?scope=…` returns bytes, part count and estimated egress cost (bytes above the remaining CloudFront free tier × configured price). Retrieval fees are 0 in v1 because Glacier IR is not used; the formula includes the term so P2 tiering works.
- Family export (owner) excludes all vaults. Exports including the requester's own vault require step-up and use the vault-capable task role.

---

## 17. Usage and cost dashboard (FR-DSH)

### 17.1 Data collection

- **Daily inventory job** reads the S3 Inventory reports (originals and vault), joins on `blob.storage_key`, updates `blob.storage_class` / `it_access_tier` only where changed (FR-STO-006), and writes `usage_snapshot` rows by member, type, class/tier, originals vs derivatives, and vault-per-member.

#### 17.1.1 S3 Metadata evaluation (TD-026, M9)

**S3 Metadata** keeps tables of a bucket's object metadata up to date automatically. AWS stores them as S3 Tables (managed Apache Iceberg) in an AWS-managed table bucket:
- a **journal table**: every object create, delete and metadata change, within minutes;
- a **live inventory table**: the current set of objects with size, storage class and other system metadata.

You query them with Athena. Compared with daily Inventory, this gives fresher data and SQL instead of parsing report files.

**Proposed use:** enable on the originals and vault buckets. The daily inventory job becomes one Athena query per bucket (per-class and per-tier totals, plus rows whose class or tier changed since the last run) instead of downloading and parsing Parquet reports. The journal table also gives the weekly reconciliation (§14.3) an exact list of objects created and deleted, which is a better orphan check than diffing daily inventories.

**Adopt only if all of these hold** (checked in M9 against the dev account):

| # | Criterion | Why |
|---|---|---|
| 1 | The live inventory table reports the **Intelligent-Tiering access tier** (frequent / infrequent / archive instant), not only `INTELLIGENT_TIERING` | The cost estimate (FR-DSH-002) depends on the tier; without it, Inventory must stay |
| 2 | Monthly cost (metadata table fees + table storage + Athena scans) at year-1 scale is within ~$1 of Inventory's | NFR-COST-001 margin is thin (§23.1) |
| 3 | Can be enabled on **existing** buckets with a backfill of existing objects | Originals already exist by M9 |
| 4 | Tables can be encrypted with a customer-managed KMS key, and access can be limited to the inventory and reconciliation job roles | Vault object keys reveal user and hash (`v/{family}/{user}/{sha}`); the tables must be as locked down as the vault bucket |
| 5 | Available in us-west-2 | TD-001 |

If any criterion fails, keep S3 Inventory (the baseline design above). Either way, `blob.storage_class` and `usage_snapshot` stay the dashboard's source; only the feeding job changes.

**Athena setup (only if adopted):** a dedicated workgroup `phamoto-metadata` with results in the staging bucket (`q/` prefix, 7-day expiry), a per-query scan limit of 1 GB, and queries issued only by the inventory and reconciliation job roles.
- Derivative bytes come from `derivative.bytes`. Dedup savings = Σ (size × (number of items referencing blob − 1)).
- Growth and the 12-month projection (FR-DSH-003) use the last 90 days of snapshots with a linear fit, recomputed daily.

### 17.2 Cost estimate (FR-DSH-002/004/005)

`packages/core/src/cost/estimate.ts`, fed by `config/pricing.{region}.json` in SSM (editable without release):

```
storage = Σ_class(bytes_class × price_class) + IT_objects_over_128KB × monitoring_fee
requests = (puts × put_price + gets × get_price)          -- from CloudWatch S3 request metrics if enabled, else modelled per item
cdn = max(0, egress_GB − free_tier_GB) × price_GB + max(0, requests − free_requests) × price_req
compute = lambda_GB_s × price + fargate_vCPU_h × price + dsql_units × price   -- from CloudWatch metrics
fixed = route53 + kms_keys + misc (config)
```

Labelled "Estimate" everywhere. Comparison against configured consumer plans (e.g. "2 TB at $9.99") per FR-DSH-004.

### 17.3 Processing health (FR-DSH-007)

Lists `item.state = 'failed'` with error and attempts; "Retry" re-enqueues dispatch. Also shows ref-count review items (§14.3) and DLQ depths.

Members see only their own usage (bytes and count by type) (FR-DSH-006).

---

## 18. Audit log (FR-AUD)

- `audit_event` rows are written in the same transaction as the action they describe (no lost events). Helper `audit(ctx, action, target, detail)`.
- **Append-only (FR-AUD-004):** the API's DSQL database role has `INSERT, SELECT` on `audit_event` and no `UPDATE`/`DELETE`. Only the `audit-retention` job role can `DELETE` rows older than the retention (1 year, config).
- **Tamper evidence:** a nightly job exports the previous day's events to `audit-archive` (Object Lock, 1-year governance retention), with a SHA-256 chain hash per file stored in the next file's header.
- **Visibility (FR-AUD-003):** owner sees all events with `scope = 'family'` plus their own `owner_only` events; members see events where `actor_user_id = me` or `target_owner_id = me`. Vault events are `owner_only` and carry no item metadata.
- **Not logged:** thumbnail and preview views (FR-AUD-005). Logged: original downloads, exports, vault actions and everything in FR-AUD-001.
- Failed sign-ins for **known** identities are logged as `auth.sign_in_failed`. Unknown identities are logged to CloudWatch only (§4.3).

---

## 19. Notifications (FR-NOT)

- **In-app:** `notification` table; `GET /notifications` polled every 60 s while the tab is visible, and refreshed through the change feed.
- **Email:** SES with templates in `services/jobs/email-templates` (MJML compiled at build). Configuration set with bounce/complaint handling → suppression list. Categories: invite, export ready, new-device sign-in, ownership transfer, member removal, optional On this day.
- **Push (R2):** APNs (token-based auth, `.p8` key in SSM) and FCM HTTP v1 (service account in SSM) from a `push-sender` Lambda; invalid tokens are pruned.
- **Batching:** `notification_outbox` processed every 5 minutes by a scheduled Lambda; album activity collapses to one notification per album per hour.
- **Preferences (FR-NOT-003):** `notification_pref` per category and channel; security emails (new-device sign-in) can't be turned off.
- **Vault safety (FR-NOT-004):** see §13.

---

## 20. Web client (PL-WEB, TD-003)

### 20.1 Stack

- Next.js (App Router) with `output: 'export'`, React, TypeScript strict.
- Data: TanStack Query on top of the generated API client. Auth token held in a memory store; silent refresh on 401 and on a timer.
- UI: Radix UI primitives and Tailwind CSS with design tokens (light/dark via CSS variables and `prefers-color-scheme`, overridable in settings, PL-X-005). Shared components in `packages/ui`.
- i18n: FormatJS (`react-intl`) with ICU messages in `packages/i18n/en.json`. A lint rule forbids hard-coded user-facing strings in JSX (PL-X-004).
- Media: virtualised grid (`@tanstack/react-virtual`), viewer with `react-zoom-pan-pinch` on previews, `<video>` for proxies, PDF pages as images, MapLibre GL JS for the map.
- Uploads: dedicated module `apps/web/src/upload/` (Web Workers for hashing, IndexedDB queue, S3 part uploader).

### 20.2 Routing on a static export

Dynamic routes (`/albums/[id]`, `/items/[id]`, `/exports/[id]`, `/invite/[token]`) are exported as a single shell each (`/albums/_.html` etc.). A **CloudFront Function** on `dist-web` rewrites `/albums/{anything}` → `/albums/_.html`; the client reads the ID from `location.pathname`. Unknown paths go to `404.html`. `/api/*` goes to API Gateway.

### 20.3 Responsive and accessible (PL-WEB-002, PL-X-003)

- Breakpoints: 360, 768, 1024, 1440. Bottom navigation on phones, side navigation from 1024 px.
- WCAG 2.1 AA: axe-core checks in Playwright on every page; keyboard paths for grid and viewer (PL-WEB-004: arrows, space, `Del`, `Esc`, `?`), focus management in dialogs, minimum 44 px targets on touch.
- **Installable (PL-WEB-007):** `manifest.webmanifest`, icons (including maskable and Apple touch icons), `display: standalone`. A minimal service worker caches only the app shell and static assets (no media, no API), which keeps P3 offline out of scope while making the installed app start fast.

### 20.4 Performance budget

- Initial JS for the library route < 200 KB gzipped. Map, viewer, importer and vault are code-split.
- Thumbnails: `<img decoding="async" fetchpriority>` with explicit `width`/`height`; `thumb2x` only on DPR ≥ 2 or day zoom.
- HTTP/3 enabled on both distributions.

---

## 21. Mobile apps — Release 2 (summary, TD-019)

This section fixes what the Release 1 backend must already support. Full mobile design is a separate addendum at M10.

- **Stack:** Flutter (Dart) UI, Riverpod state, drift (SQLite) local DB, generated Dart API client. Native modules:
  - iOS (Swift): PhotoKit change observer, original resource export (`PHAssetResourceManager`, including `.fullSizePhoto` for edited renders, `.pairedVideo` for Live Photos), background `URLSession` upload of presigned parts, `BGProcessingTask` for hashing, Keychain, LocalAuthentication.
  - Android (Kotlin): MediaStore scanning per bucket (folder), WorkManager with network/charging constraints, foreground service for large initial backups, `createDeleteRequest`, Keystore, BiometricPrompt, Credential Manager (passkeys).
- **[SPIKE S5] at M10 start:** background upload throughput on iOS 17+ with 1,000 photos while the app is backgrounded. If Flutter plugin overhead blocks the backup engine, the engine stays native (it already is) and only the UI choice is revisited.
- **Backend features R2 relies on (all built in R1):** token auth with refresh rotation (§4.4), idempotent upload declare (§9.2), `device` and `device_asset` endpoints (FR-BKP-009), change feed with revocations (§5.3), `/sync/validate`, push token registration, presigned part URLs usable from background sessions (60-minute expiry; the engine requests new URLs on resume).
- **Backed-up rule (FR-FRE-001/003):** an asset is eligible for "Free up space" only when every variant for that asset (original, edited render, paired video) has a server item variant whose blob is verified (item state ≥ `uploaded`).
- **Offline (FR-OFF):** thumbnail LRU on disk (1 GB default), offline pins stored in drift, purged by revocation records from the change feed. Vault offline copies use an app-private encrypted directory (iOS Data Protection `complete`; Android EncryptedFile with a Keystore key requiring user authentication).

---

## 22. Non-functional implementation

### 22.1 Performance (NFR-PERF)

| Target | How |
|---|---|
| Library first page < 1.5 s (NFR-PERF-001) | Static shell from CloudFront, segments + first month in two parallel API calls, ThumbHash placeholders |
| Thumb < 300 ms, preview < 500 ms (NFR-PERF-002) | Immutable derivatives with 1-year CDN TTL, path-only cache key, HTTP/3, sizes in §10.3 |
| 60 fps grid (NFR-PERF-003) | Virtualised justified rows, fixed-size cells computed from `w/h`, no layout thrash, image decode off main thread |
| API p95 < 300 ms (NFR-PERF-004) | Indexed keyset queries, no N+1 (batch-load album names, URLs), per-user segment cache, arm64 Lambda with 1024 MB |
| Upload ≥ 80% of link (NFR-PERF-005) | Parallel parts, 16 MiB parts, pipelined hashing |

Load test (k6) at 10× year-1 scale (PRD §3.3): 3M items synthetic family, 50 concurrent users scrolling and searching. Run in M4 and before launch.

### 22.2 Reliability (NFR-REL)

- Uploads are acknowledged only after verify (§9.3). Clients mark "backed up" from server state, never from local completion.
- All queue consumers are idempotent and have DLQs with alarms. Scheduled jobs use a `job_lock` row with a lease to avoid overlap.
- **Monthly integrity check (NFR-REL-006):** samples 0.5% of originals (min 200, max 2,000) and all originals uploaded in the last 30 days that have never been checked, re-hashes them in the verify worker code, and alarms on any mismatch. Reading an IT object moves it to the frequent tier for 30 days; the sample size keeps this cost negligible.
- Availability 99.5% (NFR-REL-004) is met by managed regional services; no multi-region.

### 22.3 Security (NFR-SEC)

- TLS 1.2+ (`TLSv1.2_2021` security policy) on CloudFront; HSTS preload.
- IAM: one role per function; bucket policies additionally restrict by role ARN; KMS key policies list principals explicitly (§7.2). CDK Nag in CI.
- Secrets: SSM Parameter Store SecureString (encrypted with `phamoto/app-secrets`): OIDC client secrets, Apple private key, origin-verify header, URL-signing keys (also mirrored to CloudFront KeyValueStore), APNs/FCM credentials. Nothing secret in the repo or bundles.
- OWASP ASVS L2 checklist tracked in `docs/security/asvs.md`, with tests for authn, session, access control and input validation.
- **WAF:** not enabled in v1 (a web ACL costs ~$5–10/month, about half of the non-storage budget). Compensating controls: API Gateway throttling, app-level auth rate limits, invite-only surface. Re-evaluate if alarms show abuse, or adopt it if a CloudFront flat-rate plan that bundles WAF at no extra cost fits (check current terms).
- Uploaded content: never served from `phamoto.com` (media lives on `media.phamoto.com`, cookie-less); originals served with `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff`; SVG and HTML documents are never rendered inline.
- Dependencies: Renovate with a weekly batch; OSV-Scanner and Trivy gates in CI (NFR-SEC-010).

### 22.4 Privacy (NFR-PRV)

- No third-party analytics. Product analytics, if added, are counters in CloudWatch EMF without content or identifiers.
- Location: geocoding and maps are self-hosted (§10.7, §11.4). GPS is returned only for items the caller can see; strip-on-download per §16.
- **Account deletion (NFR-PRV-005):** follows FR-FAM-004 (b)/(c) paths, completes within 30 days, then DSQL backups age out within 35 days and S3 non-current versions within 30 days. The privacy policy states these windows.

### 22.5 Observability (NFR-OPS)

- Powertools for AWS Lambda (TypeScript): structured JSON logs with `request_id`, `family_id`, `user_id` (no PII beyond IDs, never filenames of vault items), EMF metrics, X-Ray tracing at 5% sampling.
- Log retention: 30 days (90 days for auth logs).
- **Alarms → SNS → owner email:** API 5xx rate > 1% (5 min), any DLQ depth > 0, processing queue oldest message age > 30 min, verify mismatches > 0, integrity-check mismatch, ref-count reconciliation mismatch, auth failures > 50/15 min, unknown-identity sign-in attempts > 20/hour, AWS Budgets at 80% and 100% of the monthly budget.
- Tags on every resource: `app=phamoto`, `env`, `component` (NFR-OPS-003). Cost allocation tags activated in the management account.

---

## 23. Cost model (G2, NFR-COST)

### 23.1 Steady state, year 1 (2 TB originals, ~250k items, 5 users)

| Line | Basis | Est. $/month |
|---|---|---|
| Originals storage | 2 TB in IT, mostly Archive Instant after 90 days | 8–10 |
| IT monitoring | ~250k objects > 128 KB × $0.0025/1,000 | ~0.6 |
| Derivatives | ~100 GB Standard | ~2.3 |
| **Storage subtotal** | | **~11–13** |
| DSQL | Light family usage (verify in S1) | 1–5 |
| Lambda + API Gateway | Mostly within free tier | < 1 |
| CloudFront | Within 1 TB / 10M requests free tier; Functions ~$0.10/M | < 0.5 |
| KMS | 4 keys × $1 + requests (bucket keys on) | ~4.5 |
| Route 53 | 1 hosted zone | 0.5 |
| CloudWatch | Logs ~2 GB + ~15 alarms | 2–3 |
| SES, SQS, EventBridge, Step Functions, ECR, Backup | Low volume | ~1–2 |
| Tiles bucket | ~80 GB in IT, mostly cold | ~0.5–1 |
| **Non-storage subtotal** | Target < $15 (NFR-COST-001) | **~11–17** |

The non-storage line sits at the edge of the budget. Levers if needed: merge `app-secrets` into `library` key (−$1), shorter log retention and fewer debug logs, CloudWatch alarm consolidation (composite alarms), and dropping the tiles bucket in favour of a smaller zoom range.

### 23.2 Apple Developer Program

~$99/year (needed for Sign in with Apple on the web in R1, and for the iOS app).

### 23.3 One-time bulk import (shown before import, NFR-COST-001)

`estimate = photos × c_photo + video_minutes × c_video + pages × c_page + PUT/COPY requests`, with coefficients measured in M3 and stored in pricing config. Rough planning figures: ~$0.00005 per photo (Lambda image processing), ~$0.002–0.01 per video minute (Fargate Spot / Lambda), so 250k photos + 10k video minutes ≈ $15–100. The first months also cost more for storage because originals sit in IT's frequent tier for 30 days (PRD R1).

---

## 24. Testing strategy (PRD §17.2)

| Layer | Tooling | Scope |
|---|---|---|
| Unit | Vitest | `packages/core` (visibility, dedup decisions, state machine, cost estimator, Takeout sidecar matcher, capture-time precedence), CF Function signing/verifying, zod contracts |
| Integration | Vitest + Postgres 16 container with a DSQL-compat lint (rejects FKs, triggers, sequences, multi-DDL transactions) + LocalStack S3/SQS for pipeline glue | API routes end-to-end against a real database, authorisation matrix (§6.4), purge/reconcile races |
| DSQL conformance | Nightly CI job against the dev DSQL cluster | Migrations, OCC retry behaviour, performance smoke |
| Media corpus | Worker tests with a fixture corpus in a test S3 bucket | HEIC (incl. iPhone 15+ HEIF), ProRAW, DNG, CR3, NEF, Motion Photo JPEG, Live Photo pair, HDR HLG/Dolby Vision clips, 4K HEVC, animated GIF/WebP, multi-page and encrypted PDFs, DOCX/XLSX/PPTX, Takeout and iCloud sample exports, truncated sidecar names |
| Security | Fuzzing malformed images/PDFs against workers (time/memory limits hold); authz negative tests; vault leakage tests (dedup, notifications, audit view, map, search, memories) | R7 |
| E2E | Playwright (Chromium, WebKit, Firefox; mobile viewports) against `dev.phamoto.com` with seeded test identities (passkeys via virtual authenticator) | All P0 flows in PRD §8 |
| Accessibility | axe-core in Playwright | WCAG 2.1 AA |
| Load | k6 | §22.1 |
| Infra | CDK assertions + CDK Nag | Public access blocked, IT archive tiers disabled, versioning on, lifecycle rules present (FR-STO ACs) |

Each FR's ACs map to named tests; the mapping lives in `docs/traceability.md` and is checked in review.

---

## 25. Delivery plan

PRD §17.1 milestones apply. Technical additions:

| M | Additions |
|---|---|
| M0 | **Spikes (≤ 2 weeks total):** S1 DSQL (§6.1); S2 HEIC/RAW decode in Lambda container (sharp + libheif + LibRaw build, cold start, memory); S3 CloudFront Function HMAC + KeyValueStore latency and correctness; S4 browser upload throughput and resume on Safari iOS and Chrome Android. Then: accounts, CDK stacks, buckets, keys, distributions, CI/CD, static web skeleton, API skeleton with health check. |
| M1 | Auth module, invites, sessions, step-up, recovery codes, bootstrap CLI, landing page. |
| M2 | Upload declare/parts/complete, verify worker, dedup, web uploader, folder upload, phone-browser upload. |
| M3 | Dispatcher, workers, results-writer, derivative specs, geocoder, Live/Motion pairing, re-process. Measure import cost coefficients. |
| M4 | Segments/items APIs, grid, viewer, search, documents view, overrides. Load test #1. |
| M5 | Albums, sharing, change feed. |
| M6 | Trash, purge, reconciliation, vault (vault-api, vault workers, encryption), audit log + archive. |
| M7 | Importers (folder, Takeout, iCloud), exports (Fargate), download originals. |
| M8 | Map (PMTiles basemap, clusters), places, memories. |
| M9 | S3 Metadata evaluation against §17.1.1 criteria (adopt or keep Inventory), dashboard (inventory job, cost estimator), notifications (in-app, email), settings, manifest/installability, launch checklist (security review, restore drill, load test #2). |
| M10–11 | Mobile per §21, starting with spike S5. |

---

## 26. Risks specific to this design

| ID | Risk | Mitigation |
|---|---|---|
| TR1 | DSQL limits, PITR or cost differ from assumptions | Spike S1 with explicit exit criteria; portable Kysely layer; designed fallback (Aurora Serverless v2 + Data API) |
| TR2 | In-house auth has a security bug | Vetted libraries only, no custom crypto, ASVS L2 tests, external review before launch, short access tokens, per-request session check |
| TR3 | HMAC media URLs leak (copied link) | 30–60 min validity, path-bound signature, unguessable `url_secret`, monthly key rotation |
| TR4 | HEIC/RAW decoder builds are brittle | Pinned versions in the worker Dockerfile, fixture corpus in CI, embedded-JPEG fallback for RAW |
| TR5 | Non-storage costs creep above $15 | Budgets alarms, cost levers in §23.1, cost note required in PRs (PRD §17.2) |
| TR6 | Static export limits future SSR needs | None needed for v1. Switching to OpenNext later is additive (the API doesn't change). |
| TR7 | Lambda limits for huge files | Fargate paths for verify (> 8 GB), video (> 3 min) and exports |

---

## Appendix A. PRD [TECH SPEC] marker index

| PRD location | Resolved in |
|---|---|
| FR-FAM-001 bootstrap | §4.9 |
| FR-PRC-001 compute | §10.1 (TD-006) |
| FR-PRC-005 HLS vs MP4 | §10.4 (TD-008) |
| FR-PRC-006 office conversion | §10.5 (TD-010) |
| FR-VLT-006 vault key and access | §7.2, §8.2, §13 |
| PL-WEB-001 hosting | §20 (TD-003) |
| PL-X-001 mobile stack | §21 (TD-019) |
| NFR-SEC-008 certificate pinning | TD-021 |
| §10 RAW preview | §10.3 (TD-011) |
| §11 storage technology | §6 (TD-002) |
| §12 final naming, vault dedup mechanism | §7.1, §9.4 (TD-016) |
