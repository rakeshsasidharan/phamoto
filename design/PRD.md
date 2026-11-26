# Phamoto — Product Requirements Document

| Field | Value |
|---|---|
| Product | Phamoto (family photo, video & document storage) |
| Domain | phamoto.com |
| Clients | Web (Next.js), iOS app, Android app |
| Document type | Product Requirements (what & why). A separate Technical Spec will define *how*. |
| Version | 0.1 (draft) |
| Date | 2026-10-07 |
| Owner | Rakesh Pillai |

---

## 0. How to use this document (for humans and build agents)

- Every requirement has a stable ID (e.g. `FR-UPL-003`). Reference IDs in issues, branches, commits and tests.
- Priority maps to release (see §5):
  - **P0** = **Release 1, the website (phamoto.com)**. This is the absolute first priority. Nothing else ships before it.
  - **P1** = **Release 2, the iOS and Android apps**.
  - **P2** = **Release 3, enhancements**.
  - **P3** = **later / SaaS phase**.
- A requirement that spans web and mobile is P0 for its web behaviour. Its mobile-specific parts (marked *"Mobile: … (P1)"*) ship with the apps.
- Build agents MUST NOT start P1 work until all P0 requirements are done, unless the owner explicitly says otherwise.
- "v1" in this document means the private, single-family product (Releases 1 and 2), as opposed to the later SaaS phase.
- Each requirement has **acceptance criteria (AC)**. A requirement is done only when all its ACs are covered by automated tests or a documented manual test.
- "MUST / SHOULD / MAY" follow RFC 2119 meaning.
- Anything marked **[TECH SPEC]** is intentionally left to the technical spec. Do not make irreversible architecture choices for those items based on this document alone.
- Items in §16 (Open Questions) are unresolved. Do not implement around them without a decision.

---

## 1. Summary

Phamoto is a private, family-owned media and document vault. Family members back up photos and videos from their phones, upload documents (PDFs, scans, office files), and organise everything into albums that can be private or shared with other family members.

All original files are stored byte-exact in Amazon S3, in the family's own AWS account. For every file, Phamoto also generates lightweight derivatives (thumbnail, screen-size preview, low-bitrate video proxy, document page images). Day-to-day browsing uses only these derivatives, so originals can live in cheaper storage classes without hurting the experience.

### 1.1 Problem

- Each family member pays separately for iCloud or Google One, or the family pays for a top storage tier, and storage is fragmented across providers and accounts.
- Photos of the same events are scattered across different phones and clouds.
- Important family documents (IDs, insurance, tax, medical) sit in email, WhatsApp and random folders.
- Consumer clouds lock data in. Exports are painful and pricing changes are outside the family's control.

### 1.2 Goals

| ID | Goal | Measure |
|---|---|---|
| G1 | One place for all family photos, videos and documents | Release 1: existing family archives (Takeout/iCloud/NAS) imported within 60 days of website launch. Release 2: 100% of active family phones auto-backing up within 30 days of app launch |
| G2 | Lower total storage cost than the commercial alternatives | Monthly AWS bill for the whole family lower than the combined iCloud and Google One spend it replaces (see §2) |
| G3 | Browsing feels as fast as Google Photos or iCloud Photos | Thumbnail grid p95 load under 300 ms, smooth scrolling (see §9) |
| G4 | Zero data loss and no lock-in | Originals byte-exact, S3 durability, full export available at any time |
| G5 | Private by default, easy to share within the family | Every item has an explicit owner and visibility. Sharing an album takes 2 taps or fewer |
| G6 | SaaS-ready foundation | Data model and storage layout are multi-tenant from day 1, even though only one family uses v1 |

### 1.3 Non-goals (v1)

- Sharing outside the family (public links, guest accounts).
- Face recognition, AI object/scene search, document OCR (deferred, see §13).
- Comments, reactions and social features.
- Photo editing beyond what is needed to view correctly (rotation from EXIF).
- Billing, payments, self-service sign-up for other families.
- End-to-end encryption (it would block server-side previews and future smart features).
- Desktop sync clients (web upload covers desktop in v1).

---

## 2. Cost rationale and storage strategy

The core value proposition is cost. The PRD therefore treats storage cost as a product requirement, not just an implementation detail.

### 2.1 Reference prices (approximate, us-east-1; verify current AWS pricing before launch)

| Storage | $/GB-month | 1 TB / month | 2 TB / month | Access |
|---|---|---|---|---|
| S3 Standard | ~0.023 | ~$23.5 | ~$47 | ms, no retrieval fee |
| S3 Standard-IA | ~0.0125 | ~$12.8 | ~$25.6 | ms, retrieval fee per GB |
| S3 Intelligent-Tiering (Archive Instant tier, after 90 days idle) | ~0.004 | ~$4.1 | ~$8.2 | ms, no retrieval fee, small per-object monitoring fee |
| S3 Glacier Instant Retrieval | ~0.004 | ~$4.1 | ~$8.2 | ms, ~$0.03/GB retrieval fee, 90-day minimum |
| S3 Glacier Deep Archive | ~0.001 | ~$1 | ~$2 | 12–48 h restore (**not used**, see FR-STO-004) |

Commercial comparison: iCloud+ 2 TB and Google One 2 TB are each about $10/month and can be shared with a family.

> **Why originals are not in S3 Standard:** if every original stayed in S3 Standard, 2 TB would cost about $47/month in storage alone. That is **more** than one shared 2 TB consumer plan. Phamoto only beats commercial pricing when originals sit in the ~$0.004/GB tiers.
>
> **Decision (v1):**
> - **Snapshots (derivatives) → S3 Standard.** These are thumbnails, previews, video proxies and document page images. They are about 3–6% of original size (~60–120 GB for 2 TB of originals, ≈ $1.5–3/month) and serve all browsing.
> - **Originals → S3 Intelligent-Tiering** (frequent → infrequent after 30 days → Archive Instant Access after 90 days). Reads stay at millisecond latency with no retrieval fees or restore workflow. Cold originals drop to ~$0.004/GB automatically, so 2 TB costs roughly $8–10/month once settled.
> - Intelligent-Tiering's opt-in Archive / Deep Archive Access tiers MUST stay **disabled** (FR-STO-004).
> - Objects under 128 KB are not auto-tiered; they stay at the frequent-tier rate and have no monitoring fee. This is negligible for originals.

### 2.2 Cost principles (requirements)

- **CP-1** Browsing, scrolling, viewing, map, timeline and memories MUST be served from derivatives only. Originals are read only for explicit download, export, "view original", or re-processing.
- **CP-2** Derivatives MUST be served through a CDN (CloudFront) so most viewing egress falls inside the CDN free tier or at CDN rates.
- **CP-3** Storage class policy MUST be configuration, not code. Changing a lifecycle rule must not require an app release.
- **CP-4** Duplicate files MUST NOT be stored twice (FR-DUP).
- **CP-5** Incomplete multipart uploads MUST be aborted automatically (for example after 7 days) so they do not accrue cost.
- **CP-6** The owner MUST be able to see estimated monthly cost (FR-DSH).

---

## 3. Users, roles and personas

### 3.1 Roles (v1)

| Role | Count per family | Capabilities |
|---|---|---|
| **Owner** | Exactly 1 (transferable) | Everything a Member can do, plus: invite/remove members, view the family usage and cost dashboard, view the full audit log, configure family settings (storage policy display, trash retention), transfer ownership, delete the family |
| **Member** | 0..n | Upload and back up own content, create albums, share albums with family members, manage own vault, trash and restore own items, export own data |

Ownership is about **items**, not roles. The user who uploads an item owns it. Only the item owner can permanently delete it or move it into the vault, including when the item appears in someone else's shared album.

### 3.2 Personas

- **U1 — The organiser (Owner):** runs the AWS account, cares about cost and data safety, uploads the old family archive from a PC or NAS.
- **U2 — Phone-first parent (Member):** takes hundreds of photos and videos a month. Wants it all backed up automatically and wants space freed on the phone.
- **U3 — Light user / grandparent (Member):** mostly views shared family albums. Needs a dead-simple UI and large tap targets.
- **U4 — Document keeper (Member):** scans and stores IDs, insurance, tax and medical records. Wants them locked away and quick to find.

### 3.3 Scale assumptions (year 1)

- 1 family, fewer than 5 users, under 2 TB of originals, roughly 100k–300k items.
- Architecture MUST NOT preclude scaling to thousands of families (G6), but v1 does not need to be load-tested beyond 10× the year-1 scale.

---

## 4. Glossary

| Term | Meaning |
|---|---|
| **Family** | The tenant. All data belongs to exactly one family. |
| **Item** | One logical media or document entry (photo, video, live photo, document). Has one owner. |
| **Original** | The byte-exact uploaded file(s). |
| **Blob** | A unique stored file, identified by content hash. Multiple items may reference the same blob (dedup). |
| **Derivative** | A generated file: thumbnail, preview, video proxy, document page images, poster frame. |
| **Album** | A collection of items of any type (photos, videos, docs). Private or shared. |
| **Shared album** | An album with one or more other family members added as viewers or contributors. |
| **Library / Timeline** | All items a user can see (own items plus items in albums shared with them), ordered by capture date. |
| **Vault** | A per-user, extra-locked area. Vault items are hidden from every other view and need a biometric or passkey to open. |
| **Trash** | Soft-deleted items, restorable for 30 days. |
| **Backup set** | The device folders/albums selected for automatic backup on a phone. |

---

## 5. Release scope

### 5.1 Release 1 — Website at phamoto.com — P0 (absolute priority)

The website is the first and only client in Release 1. It MUST be fully usable on its own, on desktop and phone browsers, with no dependency on the mobile apps. The backend and API built for Release 1 MUST already serve the apps' needs (PL-X-002), so Release 2 adds no backend redesign.

1. **Web app (Next.js) at phamoto.com** with landing page, responsive layout for desktop, tablet and phone browsers (§7.1).
2. Accounts: Sign in with Google, Sign in with Apple, passkeys (WebAuthn), MFA. Invite-only family.
3. Family: owner and member roles, invites, member removal, ownership transfer.
4. Web upload: file picker, drag-and-drop, whole-folder upload with resume. Upload from a phone browser through the OS photo picker (PL-WEB-006).
5. Bulk import / migration (Google Takeout, iCloud export, local/NAS folders).
6. Processing pipeline: metadata extraction, thumbnail, preview, video proxy, document page images.
7. Byte-exact originals, content-hash dedup, storage classes per §2.1.
8. Library timeline, item viewer, basic metadata search and filters, Documents view.
9. Albums for any file type. Sharing with family members (viewer/contributor).
10. Map view, places, "On this day" memories.
11. Private vault with passkey/MFA re-authentication lock.
12. Trash with 30-day restore.
13. Download items/albums, full account and family export.
14. Family usage and estimated-cost dashboard (owner).
15. Audit log.
16. In-app and email notifications (invites, share events, export ready, new sign-in).

### 5.2 Release 2 — iOS & Android apps — P1

Same feature set as the website, delivered natively, plus everything that only makes sense on a phone:

1. iOS and Android apps (stack per OQ-03) with library, viewer, albums, map, memories, vault, trash, settings.
2. Automatic camera-roll backup with Wi-Fi/charging controls (FR-BKP).
3. "Free up space" (FR-FRE).
4. Offline thumbnail cache and "make available offline" (FR-OFF).
5. Biometric vault unlock and mobile vault protections (FR-VLT-002 mobile, FR-VLT-007).
6. Push notifications (FR-NOT-001 push channel).
7. Share-sheet "Save to Phamoto" (PL-IOS-004, PL-AND-004).
8. App Store and Play Store submission.

### 5.3 Release 3 — Enhancements — P2

- Storage tiering rules beyond the default (age- or access-based policies, see FR-STO).
- Favourites and hidden items.
- Optional cross-region replication of originals (see OQ-04).
- Duplicate/near-duplicate review screen.
- Auto-add backups to an album, server-side imports, low-storage suggestions, extra memory types.
- Actual AWS cost from Cost Explorer and budget alerts.

### 5.4 Later — P3

- Sharing outside the family (expiring links, guests).
- Face grouping, AI search, auto-tags, document OCR and full-text search.
- Comments and reactions.
- Basic non-destructive editing.
- Multi-family SaaS: self sign-up, plans, billing, per-family quotas.
- Document categories with expiry reminders (passport, insurance).

---

## 6. Functional requirements

### 6.1 Authentication & account — `FR-AUTH`

| ID | Pri | Requirement |
|---|---|---|
| FR-AUTH-001 | P0 | Users MUST be able to sign in with **Google**. |
| FR-AUTH-002 | P0 | Users MUST be able to sign in with **Apple** (required by App Store rules when another social login is offered). |
| FR-AUTH-003 | P0 | Users MUST be able to register and use **passkeys** (WebAuthn on web, which works with synced iCloud Keychain / Google Password Manager passkeys). *Mobile: native platform passkey sign-in in the apps (P1).* |
| FR-AUTH-004 | P0 | **MFA** MUST be supported (TOTP authenticator app at minimum). The owner account MUST have MFA or a passkey enabled. |
| FR-AUTH-005 | P0 | A user MAY link several sign-in methods (for example Google and a passkey) to one Phamoto account. |
| FR-AUTH-006 | P0 | **Invite-only:** an identity that has not been invited to a family MUST NOT be able to create an account in v1. A sign-in attempt by an uninvited identity shows "Phamoto is invite-only" and creates no records. |
| FR-AUTH-007 | P0 | **Account recovery:** users MUST be able to download single-use recovery codes. A user who loses all sign-in methods can be re-invited by the owner, which re-binds the account after the owner confirms in-app. |
| FR-AUTH-008 | P0 | Sessions: web sessions expire after 30 days of inactivity. Users can see active sessions and devices and revoke them. *Mobile: long-lived rotating refresh tokens (P1).* |
| FR-AUTH-009 | P0 | Sign out on a device MUST clear tokens, offline content and the vault cache from that device. |
| FR-AUTH-010 | P3 | Open self-service sign-up (SaaS). |

**AC**
- Signing in with Google or Apple as an invited email lands in the family library. An uninvited email is rejected with no account created.
- A passkey created on iPhone works for sign-in on phamoto.com through the synced passkey.
- Revoking a session from web forces that device to the sign-in screen within 5 minutes (next token refresh).
- Recovery codes are single-use and shown only once.

### 6.2 Family & membership — `FR-FAM`

| ID | Pri | Requirement |
|---|---|---|
| FR-FAM-001 | P0 | On first setup, the bootstrap user creates the family and becomes **Owner**. Bootstrap is a one-time admin action, for example a CLI or seeded config **[TECH SPEC]**. |
| FR-FAM-002 | P0 | Owner can invite a member by email. The invite expires after 7 days and can be resent or revoked. |
| FR-FAM-003 | P0 | The invitee accepts by signing in with Google, Apple, or by creating a passkey with the invited email. |
| FR-FAM-004 | P0 | Owner can **remove** a member. On removal the owner MUST choose one: (a) transfer the member's items to the owner, (b) keep the items read-only for 30 days so the member can export them, then delete, or (c) delete after 30 days. The member's vault is never transferred, only (b) or (c). |
| FR-FAM-005 | P0 | Owner can **transfer ownership** to another member. Requires re-authentication and acceptance by the recipient. |
| FR-FAM-006 | P0 | Family settings page: family name, members list, pending invites, trash retention (fixed at 30 days in v1, shown read-only). |
| FR-FAM-007 | P0 | A user belongs to exactly **one** family in v1. The data model MUST NOT assume this (see §11). |
| FR-FAM-008 | P3 | Multiple families per user, family plans, quotas. |

**AC**
- A removed member immediately loses access on all devices (token revocation) and disappears from shared album member lists.
- Removal with "transfer" changes ownership of every non-vault item to the owner and writes audit entries.

### 6.3 Upload (manual, web, folders) — `FR-UPL`

| ID | Pri | Requirement |
|---|---|---|
| FR-UPL-001 | P0 | Users can manually upload one or more files on the web (file picker and drag-and-drop on desktop; the OS photo/file picker from phone browsers, per PL-WEB-006). *Mobile: in-app photo picker and file picker (P1).* |
| FR-UPL-002 | P0 | Web supports **drag-and-drop of whole folders**, including nested ones, using the browser directory APIs. Folder structure MAY optionally become albums ("Create an album per top-level folder" toggle). |
| FR-UPL-003 | P0 | Uploads go **directly to S3** via presigned, multipart, resumable uploads. Application servers MUST NOT proxy file bytes. |
| FR-UPL-004 | P0 | Uploads MUST be **resumable**. A browser refresh, network drop, or app restart resumes from the last completed part. Web keeps an upload queue that survives page reloads (persisted locally) and asks the user to re-select files only if the browser cannot reopen them. |
| FR-UPL-005 | P0 | The client computes a **SHA-256** content hash before upload and asks the server whether the blob exists (FR-DUP). Existing blobs are linked, not re-uploaded. |
| FR-UPL-006 | P0 | Upload queue UI shows per-file and total progress, speed, ETA, failed items with retry, and pause/resume all. |
| FR-UPL-007 | P0 | Max file size **20 GB** per file in v1 (configurable). Larger files are rejected with a clear message. |
| FR-UPL-008 | P0 | Server validates the uploaded object (size and hash match the declared values) before marking the item `uploaded`. Mismatches are rejected and the client retries. |
| FR-UPL-009 | P0 | Uploading to a specific album from inside that album adds items to it directly. The uploader must be the album owner or a contributor. |
| FR-UPL-010 | P0 | Unsupported or unknown file types are still stored as originals (generic "file" item with an icon). No upload is refused for its type, except executables on a configurable denylist. |

**AC**
- Killing the browser at 60% of a 5 GB video upload and reopening phamoto.com resumes from 60% or higher, not from 0%.
- Uploading the same file twice results in one blob in S3 and either no new item (same owner) or a new item referencing the same blob (different owner).
- Uploading a folder of 10,000 files works without the tab crashing. Memory stays bounded by streaming hashing and a capped concurrency (for example 4–6 parallel files).

### 6.4 Mobile auto-backup — `FR-BKP` (Release 2, apps)

| ID | Pri | Requirement |
|---|---|---|
| FR-BKP-001 | P1 | On first launch (after sign-in) the app offers to enable **auto-backup** and requests the photo library permission. On iOS, "Limited" access is supported and the app explains its impact. |
| FR-BKP-002 | P1 | Users choose the **backup set**: all photos, or selected albums (iOS) / device folders (Android, e.g. Camera, WhatsApp Images, Screenshots). |
| FR-BKP-003 | P1 | Settings: Wi-Fi only (default on), allow cellular for photos only, allow cellular for videos, only while charging (default off), include videos (default on), back up existing library vs only new items going forward. |
| FR-BKP-004 | P1 | Backup runs in the **foreground** when the app is open and in the **background** within OS limits: iOS background URLSession plus BGProcessingTask and photo library change observers; Android WorkManager with constraints. The app MUST be transparent about OS limits ("Open Phamoto occasionally to keep backups current" on iOS). |
| FR-BKP-005 | P1 | Backup status screen: X of Y backed up, current item, last successful backup time, items waiting (with reason: waiting for Wi-Fi, low battery, error). |
| FR-BKP-006 | P1 | **Originals byte-exact:** iOS uploads the original resources (HEIC/JPEG/ProRAW/DNG, HEVC/ProRes). If the user edited a photo in the Photos app, both the **original** and the **current edited render** are stored as variants of one item (see OQ-05). |
| FR-BKP-007 | P1 | **Live Photos** (iOS) and **Motion Photos** (Android) are stored as one item with their still and motion components. Viewers play the motion on press-and-hold (mobile) or hover (web). |
| FR-BKP-008 | P1 | Bursts: v1 backs up the key photo. Backing up all burst frames is a setting (default off). |
| FR-BKP-009 | P1 | Each device keeps a local index (local asset ID → Phamoto item/blob hash) so reinstalling the app, or re-enabling backup, does not re-upload already backed-up items. It re-hashes and dedups instead. |
| FR-BKP-010 | P1 | Items deleted on the phone are **not** deleted from Phamoto. Backup is additive. |
| FR-BKP-011 | P1 | Auto-backed-up items are **private** to the uploader by default (consistent with FR-ALB-001). |
| FR-BKP-012 | P1 | Persistent backup errors (permission revoked, repeated failures for more than 24 h, storage policy errors) trigger a local notification and an in-app banner. |
| FR-BKP-013 | P2 | Per-album "auto-add to album" rules, for example "auto-add everything I back up to album 'Family 2026'". |

**AC**
- 1,000 new photos taken offline are fully backed up within 1 hour of joining Wi-Fi with the app in the foreground.
- With the app backgrounded on Android, new photos back up within the WorkManager window with no user action.
- Reinstalling the app and re-enabling backup results in zero duplicate items.
- An edited iOS photo shows the edited render in the timeline. "Show original" reveals the unedited original.

### 6.5 Bulk import & migration — `FR-IMP`

| ID | Pri | Requirement |
|---|---|---|
| FR-IMP-001 | P0 | Web importer accepts a **folder or files** from a local disk or NAS (via FR-UPL-002) and preserves capture dates from EXIF or video metadata. If metadata is missing, it falls back to sidecar data, then the file modified time. |
| FR-IMP-002 | P0 | **Google Takeout** support: the user extracts the Takeout zips locally and drops the folder. The importer reads Google Photos JSON sidecars (`*.json`, `*.supplemental-metadata.json`) to restore capture time, GPS and description, and maps Takeout albums to Phamoto albums. |
| FR-IMP-003 | P0 | **iCloud Photos export** support: files downloaded from iCloud.com or Apple's data-privacy export (including the CSV metadata if present). Albums are recreated where the export includes album info. |
| FR-IMP-004 | P0 | The importer **dedups** against existing content (FR-DUP). This is critical, because Takeout and phone backups overlap heavily. |
| FR-IMP-005 | P0 | Import produces a **report**: imported count, skipped duplicates, failures with reasons, albums created. The report can be downloaded as CSV. |
| FR-IMP-006 | P0 | The import job can be paused and resumed across days (large archives). |
| FR-IMP-007 | P2 | A server-side import from a zip already uploaded to S3, or from a Takeout link, so the browser does not need to stay open. |

**AC**
- Importing a Takeout folder of photos whose EXIF is stripped still places them on the correct date via the JSON sidecars.
- Re-running the same import imports 0 new items and reports all of them as duplicates.

### 6.6 Media processing & derivatives — `FR-PRC`

| ID | Pri | Requirement |
|---|---|---|
| FR-PRC-001 | P0 | Every uploaded original triggers asynchronous processing (event-driven from S3) **[TECH SPEC: Lambda / MediaConvert / containers]**. |
| FR-PRC-002 | P0 | **Metadata extraction:** capture time (with timezone), camera make/model, dimensions, orientation, duration, GPS lat/long/alt, MIME type, page count (PDF), file size, content hash. |
| FR-PRC-003 | P0 | **Thumbnail:** about 256 px on the short edge (plus a 2× variant if needed for high-density grids), WebP or AVIF, with JPEG fallback if required by a client. Orientation applied. |
| FR-PRC-004 | P0 | **Screen preview:** about 2048 px on the long edge (configurable 1080–2048), WebP/AVIF, quality tuned for about 200–500 KB. HEIC, RAW and DNG originals MUST produce previews viewable on all clients. |
| FR-PRC-005 | P0 | **Video:** poster frame thumbnail and preview, plus a **low-bitrate proxy** (720p H.264/AAC, about 1.5–2.5 Mbps; HLS or progressive MP4 **[TECH SPEC]**) for streaming. HDR originals are tone-mapped to SDR for the proxy. |
| FR-PRC-006 | P0 | **Documents:** PDFs get a first-page thumbnail and per-page preview images (rendered on demand or up to N pages eagerly, configurable). Office docs (docx, xlsx, pptx) get a thumbnail and preview via conversion to PDF if feasible **[TECH SPEC]**. Otherwise a generic icon and download only. |
| FR-PRC-007 | P0 | **Live/Motion photos:** a short motion proxy (MP4) for playback. In Release 1, Live Photo pairs arriving through web upload or import (same base name, still + MOV, for example from an iCloud export) and Android Motion Photos (embedded MP4) MUST be detected and stored as one item. |
| FR-PRC-008 | P0 | Item states: `uploading → uploaded → processing → ready` or `failed`. Before derivatives exist, clients show a placeholder with the type icon. Failed processing is retried with backoff (3 attempts). After that the item is still accessible (download original) and flagged in the owner's dashboard. |
| FR-PRC-009 | P0 | Derivatives are regenerable from originals. A re-process job MUST exist (per item, per family, or for all) so derivative specs can change later. |
| FR-PRC-010 | P0 | Reverse geocoding of GPS to place names (city, region, country) for map clustering and search. The provider is chosen in the tech spec (for example Amazon Location Service). Results are cached per rounded coordinate to control cost. |
| FR-PRC-011 | P3 | Hooks for future AI enrichment (labels, faces, OCR) as additional pipeline stages. The pipeline MUST be extensible without rework. |

**AC**
- A typical 4 MB HEIC photo reaches `ready` with thumbnail and preview within 30 s (p95) of upload completion.
- A 1-minute 4K video has a playable proxy within 10 min (p95).
- A 50-page PDF shows its first-page thumbnail within 60 s and can be paged through in the viewer.
- Changing the preview size config and running re-process regenerates previews without touching originals.

### 6.7 Storage & tiering — `FR-STO`

| ID | Pri | Requirement |
|---|---|---|
| FR-STO-001 | P0 | Originals and derivatives are stored in S3 with **SSE-KMS** encryption. All buckets block public access. Access is only via short-lived signed URLs or signed cookies. |
| FR-STO-002 | P0 | Originals and derivatives MUST live under separate prefixes or buckets so different lifecycle and storage-class rules can apply (see §12). |
| FR-STO-003 | P0 | Storage classes (configurable, per §2.1 decision): **originals → S3 Intelligent-Tiering** (written directly in that class at upload completion, archive access tiers disabled); **snapshots/derivatives → S3 Standard**; vault originals follow the same rule as originals. Export artefacts and upload staging stay in S3 Standard (short-lived). |
| FR-STO-004 | P0 | Only **millisecond-access** storage classes may be used for originals (Standard, Standard-IA, Intelligent-Tiering instant tiers, Glacier Instant Retrieval). Classes that need a restore workflow (Glacier Flexible Retrieval, Deep Archive, Intelligent-Tiering opt-in archive tiers) MUST NOT be used. Consequence: no "restore and notify" UX is needed. |
| FR-STO-005 | P0 | Storage-class rules are defined as S3 lifecycle configuration and/or Intelligent-Tiering config, managed as infrastructure-as-code (CP-3). |
| FR-STO-006 | P0 | The app MUST track the storage class of every original (from S3 inventory or events) so the dashboard can report usage by class and estimate cost. |
| FR-STO-007 | P0 | S3 Versioning ON for originals, with a lifecycle rule expiring non-current versions after 30 days. This protects against accidental overwrite or delete by bugs. |
| FR-STO-008 | P2 | Rule-based tiering (for example originals older than N days → Glacier IR; videos over X GB → Glacier IR immediately), editable by the owner from a curated set of presets in settings. |
| FR-STO-009 | P2 | Optional cross-region replication of originals (OQ-04). |

**AC**
- No S3 object is reachable without a signed URL. An anonymous GET returns 403.
- Changing the lifecycle config in IaC needs no client or app release.
- Every newly stored original reports storage class `INTELLIGENT_TIERING`. Every derivative reports `STANDARD`.
- The Intelligent-Tiering configuration on the originals bucket/prefix has no Archive Access or Deep Archive Access tier enabled.

### 6.8 Deduplication — `FR-DUP`

| ID | Pri | Requirement |
|---|---|---|
| FR-DUP-001 | P0 | Dedup by exact **SHA-256** of the original bytes, scoped to the **family**: one blob is stored once per family. |
| FR-DUP-002 | P0 | If the same owner uploads an existing blob, no new item is created. The existing item is returned (and added to the target album if one was specified). |
| FR-DUP-003 | P0 | If a different family member uploads an existing blob, a **new item** owned by that member is created and references the same blob. Each member keeps independent ownership, albums, trash and vault state. |
| FR-DUP-004 | P0 | A blob is physically deleted only when **no** item (including trashed items within retention) references it. Reference counting MUST be crash-safe and periodically reconciled. |
| FR-DUP-005 | P0 | Dedup checks MUST NOT leak the existence of another member's vault content. When a hash matches a blob referenced only by vault items, the server MUST behave identically to a normal dedup (new item, no upload needed) and reveal nothing. |
| FR-DUP-006 | P2 | Near-duplicate detection (same photo at different resolution or re-encoded) with a review screen. |

### 6.9 Library, timeline & viewer — `FR-LIB`

| ID | Pri | Requirement |
|---|---|---|
| FR-LIB-001 | P0 | **Library** shows, in reverse chronological order by capture time: the user's own items plus items in albums shared with them. Vault and trashed items are excluded. Filter: "Mine only" / "Everything I can see". |
| FR-LIB-002 | P0 | Grid zoom levels (for example day / month / year density) with pinch-to-zoom on mobile and ctrl/cmd + scroll or slider on web. Date headers and a **timeline scrubber** for fast jumps across years. |
| FR-LIB-003 | P0 | The grid uses virtualised rendering and thumbnails only. It MUST handle 300k items without degradation (see §9). |
| FR-LIB-004 | P0 | **Viewer:** swipe/arrow-key navigation, pinch/scroll zoom on the preview, video playback from the proxy, PDF page navigation, Live Photo playback, info panel (date, location, camera, size, owner, albums, storage class). |
| FR-LIB-005 | P0 | **View original:** for photos, a "Load full resolution" action fetches the original (or a full-resolution render for HEIC/RAW on web). For video, "Play original quality" where the browser or device supports the codec. |
| FR-LIB-006 | P0 | Multi-select actions: add to album, download, move to vault, delete (to trash), share to album. Actions available depend on ownership. |
| FR-LIB-007 | P0 | **Basic search & filters** (no AI): by date range, file type (photo/video/doc/other), album, owner, place name (from reverse geocoding), filename, camera. |
| FR-LIB-008 | P0 | A **Documents** smart view lists all docs (PDF, office, other non-media) the user can see, as a list with name, type, size, date and album, sortable. |
| FR-LIB-009 | P0 | Users can edit an item's capture date/time and location (for scans or imports with wrong metadata). The original file is untouched; overrides are stored as metadata. |
| FR-LIB-010 | P2 | Favourites and "hide from timeline". |

**AC**
- Scrolling through 10 years of a 200k-item library on a mid-range Android phone sustains about 60 fps with no blank cells for more than 300 ms on 4G.
- Opening a photo shows the preview within 500 ms (p95, warm CDN).
- A user never sees an item from another member's private album or vault in the library, search, map or memories.

### 6.10 Albums & sharing — `FR-ALB`

| ID | Pri | Requirement |
|---|---|---|
| FR-ALB-001 | P0 | **Private by default:** every item is visible only to its owner unless it is in an album shared with others. |
| FR-ALB-002 | P0 | Albums can hold **any item type** (photos, videos, documents, other files). |
| FR-ALB-003 | P0 | Album CRUD: create, rename, description, cover image, delete. Deleting an album never deletes its items. |
| FR-ALB-004 | P0 | Album sort: by capture date (asc/desc), date added, or custom manual order. View modes: grid (media-centric) and list (doc-centric). Filter within an album by type. |
| FR-ALB-005 | P0 | **Share album** with selected family members as **Viewer** (view/download) or **Contributor** (view, download, add own items, remove own items). There is a "Share with whole family" shortcut, which automatically includes future members. |
| FR-ALB-006 | P0 | Items added to a shared album by their owner become visible to album members **only through that album** (and in their library, per FR-LIB-001). Removing an item from the album, or the album being unshared, removes that visibility immediately. |
| FR-ALB-007 | P0 | Only the album owner can rename, delete, change sharing, or remove other people's items from the album. Contributors can remove only their own items. |
| FR-ALB-008 | P0 | A member can **save a copy** of a shared item to their own library. This creates their own item referencing the same blob (no storage cost, per FR-DUP-003). |
| FR-ALB-009 | P0 | Albums list shows "My albums" and "Shared with me", with owner avatar, item count and last updated. |
| FR-ALB-010 | P0 | Album download (FR-EXP-002). |
| FR-ALB-011 | P0 | Vault items MUST NOT be addable to any album. |
| FR-ALB-012 | P3 | Sharing outside the family: expiring links, passwords, guests. |
| FR-ALB-013 | P3 | Comments and reactions on shared album items. |

**AC**
- Member A shares album "Goa 2025" as Viewer with B. B sees the album and its items, cannot add to it, and cannot see A's other items.
- When A unshares, B loses access on next fetch, and offline copies on B's device are purged at the next sync (FR-OFF-004).
- Contributor B adds 20 of B's photos to A's album. A sees them in the album. B still owns them; A cannot delete them permanently, only remove them from the album.

### 6.11 Map, timeline & memories — `FR-MEM`

| ID | Pri | Requirement |
|---|---|---|
| FR-MEM-001 | P0 | **Map view:** clustered pins of geotagged items the user can see (excluding vault and trash). Tapping a cluster opens a grid of those items. Filters: date range, owner, album. |
| FR-MEM-002 | P0 | Places list: browse by country → city derived from reverse geocoding. |
| FR-MEM-003 | P0 | **On this day:** a daily card showing items from the same calendar date in previous years, from items the user can see. Shown at the top of the library and optionally as an email (opt-in, default off). *Mobile: optional push notification (P1).* |
| FR-MEM-004 | P0 | Memory cards open into a swipeable viewer and can be saved as an album. |
| FR-MEM-005 | P0 | Users can hide a specific date or item from memories (for example sensitive dates). |
| FR-MEM-006 | P2 | Additional memory types: "Trip to <place>" (location and date clustering), "This month N years ago". |

**AC**
- On 7 Oct, the memory card shows items captured on 7 Oct of earlier years only, in the user's local timezone.
- Items without GPS never appear on the map. Items whose location was set manually do appear.

### 6.12 Private vault — `FR-VLT`

| ID | Pri | Requirement |
|---|---|---|
| FR-VLT-001 | P0 | Each user has a personal **vault**. Items moved into it are hidden from the library, albums, search, map, memories and dashboards visible to others. They are visible only inside the vault. |
| FR-VLT-002 | P0 | Opening the vault on the web requires a fresh **passkey / MFA re-authentication**. The vault auto-locks after 5 minutes of inactivity or when the tab is hidden for more than 1 minute. *Mobile: fresh **biometric** (Face ID / Touch ID / Android BiometricPrompt) or device-credential check, auto-locking when the app goes to the background (P1).* |
| FR-VLT-003 | P0 | Vault items are **owner-only**. The family owner cannot view another member's vault content. The owner's dashboard shows only aggregate vault size per member (bytes, count), no metadata. |
| FR-VLT-004 | P0 | Users can move items into and out of the vault. Moving an item into the vault removes it from all albums (with a confirmation that lists the affected shared albums). |
| FR-VLT-005 | P0 | Uploading directly into the vault is supported (for example scan or upload an ID document from inside the vault). Vault uploads MUST NOT pass through the normal library or backup paths. Auto-backup never targets the vault. |
| FR-VLT-006 | P0 | Vault originals and derivatives use a **separate KMS key**, and access policies restrict decryption to vault-authorised sessions **[TECH SPEC]**. |
| FR-VLT-007 | P1 | On mobile, vault items made available offline are stored in app-private encrypted storage and are cleared on sign-out or app lock-out. Vault content never appears in OS share sheets without explicit user action, nor in app-switcher snapshots (the screen is blurred when backgrounded). |
| FR-VLT-008 | P0 | Every vault open, view, download and move generates an audit event (visible to the vault owner). |

**AC**
- After moving a photo into the vault, it disappears from the library, album, map, memories and search for every user, including the owner, within one refresh.
- An API request for a vault item's derivative without a vault-elevated session returns 403.
- The family owner's audit view does not show titles, thumbnails or metadata of other members' vault items.

### 6.13 Trash & deletion — `FR-DEL`

| ID | Pri | Requirement |
|---|---|---|
| FR-DEL-001 | P0 | Deleting moves the item to **Trash**. Trashed items disappear from all views (including for album members) and are restorable for **30 days**. |
| FR-DEL-002 | P0 | Restoring an item puts it back in its previous albums (if they still exist) and its previous vault state. |
| FR-DEL-003 | P0 | Users can permanently delete individual trash items or empty the trash, with a confirmation. |
| FR-DEL-004 | P0 | After 30 days, items are permanently deleted automatically. Blob deletion follows FR-DUP-004. |
| FR-DEL-005 | P0 | Only the item's **owner** can delete. Album members can only remove items from albums (FR-ALB-007). |
| FR-DEL-006 | P1 | Deleting an item in Phamoto never deletes it from the user's phone (the reverse of FR-BKP-010), unless the user explicitly chooses "Delete from phone too" in a mobile confirmation. |

### 6.14 Free up phone space — `FR-FRE` (Release 2, apps)

| ID | Pri | Requirement |
|---|---|---|
| FR-FRE-001 | P1 | The mobile app offers **Free up space**. It lists local items whose originals are confirmed `ready` in Phamoto (hash verified, all variants, including Live Photo pairs and edited versions, uploaded), with the total reclaimable size. |
| FR-FRE-002 | P1 | Deletion uses the OS-mandated confirmation (iOS PhotoKit delete prompt; Android MediaStore delete request), with an option for "items older than N days only". |
| FR-FRE-003 | P1 | Items not fully backed up MUST never be offered for deletion. |
| FR-FRE-004 | P1 | After deletion, the items remain fully viewable in Phamoto, and the backup index MUST NOT treat them as new or deleted. |
| FR-FRE-005 | P2 | A periodic suggestion when device storage is low (opt-in). |

**AC**
- An item whose upload completed but whose processing failed is still eligible (the original is safe). An item with a pending Live Photo video component is **not** eligible.

### 6.15 Offline — `FR-OFF` (Release 2, apps)

| ID | Pri | Requirement |
|---|---|---|
| FR-OFF-001 | P1 | Mobile caches thumbnails (bounded LRU, default 1 GB, configurable) and library metadata so the library and albums are browsable offline at thumbnail level. |
| FR-OFF-002 | P1 | Users can **Make available offline** for an album or individual items. This downloads previews (default) or originals (option), and shows download progress and storage used. |
| FR-OFF-003 | P1 | Offline-pinned content syncs when online: new items in a pinned album are downloaded, removed ones are deleted locally. |
| FR-OFF-004 | P1 | Losing access (unshare, member removal, item trashed by its owner) MUST purge the corresponding offline copies at the next sync. |
| FR-OFF-005 | P1 | Settings show offline storage usage per album, with "remove all offline content". |
| FR-OFF-006 | P1 | Offline actions (create album, add to album, delete) are queued and synced. Conflicts resolve server-wins with a user notice. |
| FR-OFF-007 | P3 | Web offline support (PWA). Not in v1. |

### 6.16 Download & export — `FR-EXP`

| ID | Pri | Requirement |
|---|---|---|
| FR-EXP-001 | P0 | Download a single item's **original** (or edited render or original for edited items) from the web. *Mobile: download and save to the device gallery or files (P1).* |
| FR-EXP-002 | P0 | Download an **album** or multi-selection: asynchronous zip job (zips split at a configurable size such as 4 GB). The user is notified by in-app message and email when ready (and push once the apps exist). Links expire after 7 days. |
| FR-EXP-003 | P0 | **Full export** of all of a user's items (optionally including the vault, which requires vault re-auth): originals in a `YYYY/MM/` folder structure plus a JSON/CSV manifest (metadata, albums, capture-time and location overrides), split into multiple zips. |
| FR-EXP-004 | P0 | The owner can run a **family export** (all members' non-vault items), for example for disaster recovery or migration. |
| FR-EXP-005 | P0 | Export artefacts are stored in a temp prefix with a 7-day lifecycle expiry. The cost estimate (egress plus retrieval fees for Glacier IR objects) is shown before starting large exports. |

### 6.17 Usage & cost dashboard — `FR-DSH`

| ID | Pri | Requirement |
|---|---|---|
| FR-DSH-001 | P0 | **Owner** dashboard: total stored bytes and item counts, broken down by member, by type (photo, video, doc, other), by storage class, and originals vs derivatives. |
| FR-DSH-002 | P0 | **Estimated monthly cost:** storage by class, plus estimated requests, CDN egress and processing, using a pricing table held in config (FR-DSH-005). Clearly labelled "estimate". |
| FR-DSH-003 | P0 | Trends: growth per month (bytes, items), 12-month projection at current growth. |
| FR-DSH-004 | P0 | Dedup savings ("X GB saved by deduplication") and comparison against reference consumer plans (configurable values, for example 2 TB at $9.99). |
| FR-DSH-005 | P0 | Pricing values are configurable (region and unit prices) and not hard-coded. |
| FR-DSH-006 | P0 | **Members** see their own usage (bytes and count by type), not costs. |
| FR-DSH-007 | P0 | Processing health: items in `failed` state, with a retry action. |
| FR-DSH-008 | P2 | Actual cost from AWS Cost Explorer (tag-based) alongside the estimate. AWS Budgets alert integration. |
| FR-DSH-009 | P3 | Per-member quotas (SaaS plans). |

### 6.18 Audit log — `FR-AUD`

| ID | Pri | Requirement |
|---|---|---|
| FR-AUD-001 | P0 | Record immutable audit events: sign-in (success/failure), new device/session, MFA/passkey changes, invite sent/accepted/revoked, member removed, ownership transfer, album created/shared/unshared/deleted, item added/removed from shared album, item download (original), export requested/downloaded, vault open/view/move/download, trash/restore/permanent delete. |
| FR-AUD-002 | P0 | Each event records actor, action, target (IDs), timestamp, device/session, IP and user agent. Vault events about other members' vaults are visible only to the vault owner (FR-VLT-003). |
| FR-AUD-003 | P0 | The **owner** can view and filter the family audit log (by member, action, date). **Members** can view events where they are the actor or the owner of the target. |
| FR-AUD-004 | P0 | Retention: 1 year (configurable). The log is append-only; no user, including the owner, can edit or delete entries via the product. |
| FR-AUD-005 | P0 | Viewing thumbnails and previews is **not** logged per view (too noisy). Opening originals and downloads **are** logged. |

### 6.19 Notifications — `FR-NOT`

| ID | Pri | Requirement |
|---|---|---|
| FR-NOT-001 | P0 | In-app notifications (web notification centre) for: added to a shared album, new items in a shared album (batched, at most one per album per hour), export ready, new sign-in on a new device, On this day (opt-in). *Mobile: the same events as push notifications, plus backup problems (FR-BKP-012) (P1).* |
| FR-NOT-002 | P0 | Email for: invites, export ready, new-device sign-in, ownership transfer, member removal. |
| FR-NOT-003 | P0 | Per-category notification preferences per user. |
| FR-NOT-004 | P0 | Notification content MUST NOT include vault item details. |

### 6.20 Settings — `FR-SET`

| ID | Pri | Requirement |
|---|---|---|
| FR-SET-001 | P0 | Profile: name, avatar, linked sign-in methods, passkeys, MFA, recovery codes, sessions and devices. |
| FR-SET-002 | P0 | Notifications (FR-NOT-003), appearance (light/dark/system). *Mobile: backup settings (FR-BKP-003) and offline storage (FR-OFF-005) (P1).* |
| FR-SET-003 | P0 | Family settings (owner) per FR-FAM-006. |
| FR-SET-004 | P0 | Data: export (FR-EXP-003), delete my account (owner must first transfer ownership; deletion follows FR-FAM-004 options). |
| FR-SET-005 | P0 | About: version, terms, privacy, open-source licences, support contact. |

---

## 7. Platform-specific requirements

### 7.1 Web (phamoto.com) — P0, Release 1

- **PL-WEB-001** Next.js app hosted serverlessly on AWS (static assets on S3 and CloudFront, SSR/route handlers on Lambda) **[TECH SPEC: OpenNext / Amplify Hosting / other]**.
- **PL-WEB-002** Supported browsers: latest 2 versions of Chrome, Edge, Firefox and Safari on desktop, plus Safari on iOS/iPadOS and Chrome on Android. Until the apps ship, the phone browser is how members use Phamoto on their phones, so the whole site MUST be responsive and fully usable on phone screens (360 px wide and up).
- **PL-WEB-003** The website is the reference client. Every P0 requirement in §6 MUST be usable on the web. Phone-only features (auto-backup, free up space, biometric unlock, offline) are the only things the apps add later.
- **PL-WEB-004** Keyboard shortcuts in the grid and viewer (arrows, space for select, `Del`, `Esc`, `?` for help).
- **PL-WEB-005** Landing page at phamoto.com for unauthenticated users: product description and sign-in. In Release 1 it says "invite-only".
- **PL-WEB-006** **Phone-browser upload:** on iOS Safari and Android Chrome, "Upload" opens the OS photo/file picker with multi-select (photos, videos, documents). HEIC/HEVC originals MUST be uploaded byte-exact (no browser-side conversion where the OS allows it). The upload queue MUST tolerate the tab being backgrounded and resume on return (FR-UPL-004). The UI MUST tell users that keeping the tab open speeds things up and that automatic backup arrives with the apps.
- **PL-WEB-007** Installable to the home screen (web app manifest and icons) so phone users get an app-like shortcut before Release 2. Full offline/PWA support stays P3 (FR-OFF-007).

### 7.2 iOS — P1, Release 2

- **PL-IOS-001** Minimum iOS: current major version minus 2 (iOS 17+ at time of writing; confirm in tech spec).
- **PL-IOS-002** PhotoKit integration for full and limited library access, original resource export, Live Photos, edited/original variants, and deletion for free-up-space.
- **PL-IOS-003** Background upload via background URLSession and BGTaskScheduler, within iOS limits.
- **PL-IOS-004** Share extension: "Save to Phamoto" from other apps (Files, Mail, Safari PDFs) into the library, an album, or the vault. *(This is part of manual upload.)*
- **PL-IOS-005** Sign in with Apple, platform passkeys, Face ID / Touch ID for the vault.

### 7.3 Android — P1, Release 2

- **PL-AND-001** Minimum Android: API 29 (Android 10)+ (confirm in tech spec), with handling for scoped storage and the Android 13+ granular media permissions and Android 14 partial photo access.
- **PL-AND-002** MediaStore integration, folder-based backup selection, Motion Photo handling, `createDeleteRequest` for free-up-space.
- **PL-AND-003** WorkManager background backup with constraints. A foreground service with a notification during large initial backups.
- **PL-AND-004** Share target "Save to Phamoto" (same as PL-IOS-004).
- **PL-AND-005** Google Sign-In, Credential Manager passkeys, BiometricPrompt for the vault.

### 7.4 Cross-platform

PL-X-002 to PL-X-005 are P0 (they apply to the website from day one). PL-X-001 is P1.

- **PL-X-001** Mobile tech stack (Flutter / React Native / native) is **[TECH SPEC]**. Background upload reliability on iOS MUST be a primary evaluation criterion.
- **PL-X-002** All clients use the same versioned public API. No client-specific backend logic. The Release 1 API MUST be designed for the apps too (token auth usable outside the browser, resumable upload, device registration and change-feed or sync endpoints), so Release 2 needs no breaking API changes.
- **PL-X-003** Accessibility: WCAG 2.1 AA on web. Dynamic Type / font scaling, VoiceOver/TalkBack labels, minimum 44 pt tap targets on mobile (persona U3).
- **PL-X-004** Localisation-ready: all strings externalised. v1 ships in English only.
- **PL-X-005** Dark mode on all clients.

---

## 8. Key user flows

1. **Bootstrap:** Owner deploys the infrastructure → runs the bootstrap → signs in with Google → creates passkey and MFA → downloads recovery codes → invites members.
2. **Join (Release 1):** Member gets the email invite → opens phamoto.com on laptop or phone → signs in with Apple/Google → sees an empty library and onboarding → uploads from the phone browser photo picker or imports an old archive → optionally adds Phamoto to the home screen.
2a. **Join (Release 2):** Member installs the app → signs in → enables auto-backup → selects the backup set → watches progress. Already-uploaded items are recognised by hash and skipped.
3. **Migrate:** Owner exports Google Takeout → extracts it locally → web "Import" → drops the folder → reviews the pre-scan summary (counts, estimated size, detected duplicates) → starts → can close and reopen the tab and resume → reads the import report.
4. **Share a trip:** Member selects 120 photos → "Add to album" → new album "Goa 2025" → Share → "Whole family", Contributor → other members get a notification and add their own photos.
5. **Store an ID:** Member opens the vault (passkey on web; Face ID in the apps) → Upload → camera or file → document stored → vault auto-locks.
6. **Free space (Release 2):** Member → Free up space → sees 18 GB reclaimable → "older than 30 days" → confirms the OS prompt.
7. **Check costs:** Owner → Dashboard → sees 1.4 TB, $X/month estimated, breakdown by class, dedup savings.
8. **Leave Phamoto:** User → Settings → Export all → gets notified → downloads the zips and manifest.

---

## 9. Non-functional requirements

### 9.1 Performance

| ID | Requirement |
|---|---|
| NFR-PERF-001 | Library first page (metadata plus thumbnails) interactive in under 1.5 s on web broadband and under 2 s on mobile 4G (p95, warm). |
| NFR-PERF-002 | Thumbnail fetch under 300 ms p95 from CDN. Preview under 500 ms p95. |
| NFR-PERF-003 | Grid scroll about 60 fps on mid-range devices (reference: Pixel 6a, iPhone 12) with a 300k-item library. |
| NFR-PERF-004 | Metadata API p95 under 300 ms for list and search endpoints at year-1 scale. |
| NFR-PERF-005 | Upload throughput saturates available bandwidth with parallel multipart (target at least 80% of link speed on Wi-Fi). |
| NFR-PERF-006 | Processing SLAs per FR-PRC ACs. |

### 9.2 Reliability & durability

| ID | Requirement |
|---|---|
| NFR-REL-001 | **No acknowledged upload may be lost.** An item is marked backed up on the client only after the server confirms the hash-verified original is stored. |
| NFR-REL-002 | Originals: S3 durability (11 nines) plus versioning (FR-STO-007). Optional CRR (OQ-04). |
| NFR-REL-003 | Metadata store has point-in-time recovery (at least 7 days, preferably 35) and daily snapshots. Metadata can be rebuilt from S3 objects plus object metadata/tags in an emergency (originals carry the item ID, owner and family ID in object metadata). |
| NFR-REL-004 | Availability target: 99.5% monthly for API and web (a family app on a serverless stack; no multi-region active-active in v1). |
| NFR-REL-005 | All async jobs (processing, export, import, purge) are idempotent and retry safely, with dead-letter queues and alerting. |
| NFR-REL-006 | A monthly automated **integrity check** samples originals and verifies their hash against the stored hash. |

### 9.3 Security

| ID | Requirement |
|---|---|
| NFR-SEC-001 | TLS 1.2+ everywhere. HSTS on phamoto.com. |
| NFR-SEC-002 | SSE-KMS at rest for S3 and the metadata store. A separate KMS key for vault content (FR-VLT-006). |
| NFR-SEC-003 | All S3 access via short-lived signed URLs (15 minutes or less for originals) or CloudFront signed cookies scoped to the family/user. Buckets block all public access. |
| NFR-SEC-004 | Authorisation is checked server-side on every request (owner, album membership, vault elevation). Clients are never trusted for access decisions. Tenant isolation: every query is scoped by `family_id`. |
| NFR-SEC-005 | Least-privilege IAM for every function and service. No long-lived AWS credentials in clients. |
| NFR-SEC-006 | OWASP ASVS L2 as the baseline for web and API. Rate limiting on auth and API endpoints. Security headers (CSP and others). |
| NFR-SEC-007 | Secrets in AWS Secrets Manager or SSM. Nothing secret in client bundles or the repo. |
| NFR-SEC-008 | Mobile: tokens in Keychain/Keystore. Optional app lock (biometric) for the whole app. Certificate pinning is **[TECH SPEC]**. |
| NFR-SEC-009 | Uploaded files are never executed or rendered unsandboxed. Document and image processing runs in isolated compute with resource limits (to guard against malicious PDFs and image-decoder exploits). |
| NFR-SEC-010 | Dependency and container vulnerability scanning in CI. |

### 9.4 Privacy

| ID | Requirement |
|---|---|
| NFR-PRV-001 | No third-party analytics or ad SDKs that receive media, metadata or location. Product analytics, if any, are self-hosted or privacy-preserving and contain no content. |
| NFR-PRV-002 | GPS data is visible only to users who can see the item. A per-user option strips location on download/export of their own items. |
| NFR-PRV-003 | App Store and Play privacy disclosures must accurately reflect data use. |
| NFR-PRV-004 | No server-side biometric processing in v1 (face recognition deferred). Any future face feature requires explicit per-user opt-in and regional legal review. |
| NFR-PRV-005 | Account and data deletion completes within 30 days, including derivatives and backups (subject to the versioning window). |

### 9.5 Observability & operations

- **NFR-OPS-001** Structured logs, metrics and traces for API, processing and jobs. Alarms on error rates, DLQ depth, processing backlog and auth anomalies.
- **NFR-OPS-002** All infrastructure as code. Separate `dev` and `prod` environments (separate AWS accounts recommended).
- **NFR-OPS-003** Resource tagging (`app=phamoto`, `env`, `family_id` where applicable) to support cost reporting (FR-DSH-008).
- **NFR-OPS-004** CI/CD with automated tests. Mobile builds go through TestFlight and the Play internal track.

### 9.6 Cost

- **NFR-COST-001** At year-1 scale (2 TB, 5 users), infrastructure cost other than storage (compute, DB, CDN, Cognito-equivalent, Location, monitoring) SHOULD be under $15/month in steady state. Initial bulk import processing is a one-time cost that MUST be estimated and shown before an import starts.
- **NFR-COST-002** No always-on servers (databases, containers, NAT gateways) unless the tech spec justifies them against this budget.

---

## 10. Supported file types & limits

| Category | Formats (store + preview) | Notes |
|---|---|---|
| Photos | JPEG, HEIC/HEIF, PNG, WebP, GIF (animated), AVIF, TIFF, BMP | Animated GIF previews animate |
| RAW | DNG, ProRAW, CR2/CR3, NEF, ARW, RAF, ORF, RW2 | Preview from embedded JPEG or a RAW decode **[TECH SPEC]** |
| Video | MP4/H.264, MOV/HEVC, ProRes, MKV, AVI, 3GP, WebM | Proxy 720p H.264. HDR tone-mapped |
| Live/Motion | iOS Live Photo pair, Android Motion Photo (embedded MP4) | Single item |
| Documents | PDF | Page previews |
| Office | DOCX, XLSX, PPTX, ODT, ODS, ODP, TXT, RTF, CSV | Preview where conversion succeeds; otherwise icon and download |
| Other | Any | Stored and downloadable, icon only |

Limits (v1, configurable): max file 20 GB; max 50,000 files per import job; preview generation for video up to 4 hours of duration.

---

## 11. Conceptual data model (multi-tenant ready)

Logical entities only. Storage technology is **[TECH SPEC]**. Every entity except `Blob`-to-`Family` scoping carries `family_id`.

- **Family**: id, name, owner_user_id, settings, created_at.
- **User**: id, family_id (v1: one; design allows a membership table), role (`owner`/`member`), display name, avatar, linked identities, status.
- **Membership** *(SaaS-ready)*: user_id, family_id, role, joined_at.
- **Invite**: id, family_id, email, invited_by, status, expires_at.
- **Blob**: hash (SHA-256), family_id, size, mime, storage key, storage class, ref_count, created_at, kms_key (normal/vault).
- **Item**: id, family_id, owner_user_id, type (photo/video/live/doc/other), primary_blob_hash, variant blobs (original/edited/motion component), capture_time (+tz), capture_time_override, location (+override), place (reverse geocoded), dimensions, duration, page_count, camera, filename, source (backup/upload/import + device id), state (uploading/processing/ready/failed), visibility_state (normal/vault), trashed_at, created_at.
- **Derivative**: item_id, kind (thumb/thumb2x/preview/video_proxy/poster/motion/page_n), storage key, size, format, spec_version.
- **Album**: id, family_id, owner_user_id, name, description, cover_item_id, sort_mode, share_with_whole_family flag, created_at, updated_at.
- **AlbumItem**: album_id, item_id, added_by, added_at, sort_position.
- **AlbumMember**: album_id, user_id, permission (viewer/contributor).
- **Device**: id, user_id, platform, backup settings, last_backup_at, push token.
- **DeviceAssetIndex**: device_id, local_asset_id, blob hash(es), item_id, status.
- **OfflinePin** (device-local primarily): device_id, album_id/item_id, quality.
- **ImportJob / ExportJob**: id, family_id, user_id, status, counts, report location, expires_at.
- **AuditEvent**: id, family_id, actor_user_id, action, target refs, timestamp, ip, user agent, session/device, visibility scope.
- **MemoryHide**: user_id, date or item_id.

Visibility rule (canonical, must be implemented in one place server-side):
`user U can see item I` ⇔ `I.trashed_at is null` AND ( (`I.owner == U` AND `I.visibility_state == normal`) OR (`I.visibility_state == normal` AND ∃ album A containing I where U ∈ A.members or A.share_with_whole_family) ) — and vault items are visible only to their owner inside a vault-elevated session.

---

## 12. Storage layout (conceptual)

Final naming is **[TECH SPEC]**, but these properties are required:

```
originals/{family_id}/{hash_prefix}/{sha256}            # byte-exact, content-addressed, S3 Intelligent-Tiering
vault/{family_id}/{user_id}/{sha256}                     # separate KMS key, Intelligent-Tiering; never deduped against non-vault on disk
derivatives/{family_id}/{item_id}/{kind}.{ext}            # snapshots: S3 Standard, CDN-served
exports/{family_id}/{job_id}/part-{n}.zip                 # 7-day expiry
tmp/uploads/...                                           # multipart staging, abort-incomplete rule
```

- Content-addressed originals make dedup and integrity checks trivial.
- The `family_id` prefix enables per-tenant lifecycle, inventory, cost attribution and future data-residency moves.
- Note on FR-DUP-005: a vault blob is stored under `vault/`. If a non-vault upload matches a vault-only blob, the server MUST accept a normal upload (or copy server-side) into `originals/`, without signalling that the hash already exists. Exact mechanism is **[TECH SPEC]**.

---

## 13. Future roadmap (informs architecture; not in v1)

| Feature | Notes / architectural hook |
|---|---|
| External sharing | Expiring links (CDN signed URLs), password, download toggle; guest role in Membership |
| Face grouping | Opt-in; Rekognition or on-device; biometric privacy law review (BIPA/GDPR) |
| AI search & tags | Labels plus embeddings (for example Bedrock multimodal embeddings plus a vector index); pipeline stage per FR-PRC-011 |
| Document OCR | Textract; full-text search index; doc categories and expiry reminders |
| Comments/reactions | On AlbumItem |
| Editing | Non-destructive edit lists rendered into derivatives |
| SaaS | Self sign-up, Stripe billing, plans and quotas, per-family cost metering (prefixes and tags already support this), regional buckets |
| Desktop sync | Watched-folder uploader (macOS/Windows) |

---

## 14. Out of scope (v1) — explicit

External sharing, guest accounts, face/AI/OCR features, comments, editing, billing, self sign-up, quotas, E2E encryption, web offline/PWA, desktop sync client, non-English locales, multi-family membership, Glacier/Deep Archive restore workflows.

---

## 15. Risks

| ID | Risk | Mitigation |
|---|---|---|
| R1 | **Storage cost exceeds consumer plans** | Originals in Intelligent-Tiering, only snapshots in Standard (§2.1); dashboard visibility; dedup. Residual risk: savings take about 90 days to appear, because items must sit unread before tiering down (bulk import month will cost more) |
| R2 | **iOS background backup is unreliable** (OS throttling) | Background URLSession, foreground catch-up, user education, backup-status nudges; drives the mobile stack choice |
| R3 | Takeout/iCloud imports have inconsistent metadata | Sidecar parsing, dedup, import report, manual date fix (FR-LIB-009) |
| R4 | Video processing cost and latency (MediaConvert vs self-run ffmpeg) | Tech spec compares; proxy is generated once; reprocessing is rare |
| R5 | Single-person operational burden (the owner runs AWS) | Serverless, IaC, alarms to email, monthly integrity check, documented runbooks |
| R6 | Data loss via bugs (accidental purge) | S3 versioning, trash, ref-count reconciliation, deletion guarded by checks and alarms |
| R7 | Vault leakage through dedup, notifications or caches | FR-DUP-005, FR-NOT-004, FR-VLT-007, dedicated tests |
| R8 | App Store review issues (Sign in with Apple, privacy labels, account deletion) | PL-IOS-005, FR-SET-004, NFR-PRV-003 |

---

## 16. Open questions

| ID | Question | Default until decided |
|---|---|---|
| OQ-01 | ~~Originals storage class at launch~~ **Resolved:** originals → Intelligent-Tiering, snapshots → Standard (§2.1). Remaining: should very large or old imported videos go straight to Glacier IR (skips the 90-day wait but adds a ~$0.03/GB retrieval fee)? | No; Intelligent-Tiering only |
| OQ-02 | AWS region(s) for prod. | Configurable; decide in tech spec |
| OQ-03 | Mobile stack: Flutter vs React Native vs native. | Tech spec |
| OQ-04 | Cross-region replication of originals (doubles storage cost) vs relying on S3 durability plus versioning. | Off in v1 |
| OQ-05 | For edited iOS photos: store both original and edited render (current spec), or original only? | Both |
| OQ-06 | Metadata store: DynamoDB vs Aurora Serverless Postgres (search/filter needs vs cost floor). | Tech spec |
| OQ-07 | Video proxy delivery: HLS (adaptive, more objects) vs single progressive MP4 (simpler, cheaper). | Tech spec |
| OQ-08 | Should a member's items be auto-shared with the owner for estate or continuity purposes (for example "legacy contact")? | No |
| OQ-09 | Whole-app biometric lock on mobile: default on or off? | Off (vault is always locked) |
| OQ-10 | Exact derivative sizes and formats (AVIF vs WebP) given client support. | WebP thumb and preview, JPEG fallback |

---

## 17. Build guidance for agents

### 17.1 Suggested milestones (each milestone is independently testable)

| M | Release | Scope | Key IDs |
|---|---|---|---|
| M0 | R1 (P0) | Infrastructure foundation: environments, buckets, KMS, CDN, auth provider, CI/CD, IaC, Next.js hosting skeleton at phamoto.com | FR-STO-001/002/005/007, PL-WEB-001, NFR-SEC, NFR-OPS |
| M1 | R1 (P0) | Auth & family on the web: sign-in methods, invites, roles, sessions, landing page | FR-AUTH, FR-FAM, PL-WEB-005 |
| M2 | R1 (P0) | Upload core and web uploader: presigned multipart, resume, hash and dedup, item states, folder upload, phone-browser upload | FR-UPL, FR-DUP, PL-WEB-006 |
| M3 | R1 (P0) | Processing pipeline: metadata, thumbs, previews, video proxy, PDF pages, Live/Motion pairing, geocoding | FR-PRC |
| M4 | R1 (P0) | Web library: timeline, viewer, search/filters, documents view, date/location fix | FR-LIB |
| M5 | R1 (P0) | Albums & sharing, visibility rule | FR-ALB, §11 rule |
| M6 | R1 (P0) | Trash, vault (web), audit log | FR-DEL, FR-VLT, FR-AUD |
| M7 | R1 (P0) | Import (Takeout/iCloud/folders) and export | FR-IMP, FR-EXP |
| M8 | R1 (P0) | Map, places, memories | FR-MEM |
| M9 | R1 (P0) | Dashboard, in-app/email notifications, settings, home-screen install, web launch | FR-DSH, FR-NOT, FR-SET, PL-WEB-007 |
| M10 | R2 (P1) | Mobile apps: stack decision, library, viewer, albums, map, memories, vault with biometrics | PL-IOS, PL-AND, PL-X-001 |
| M11 | R2 (P1) | Auto-backup, free up space, offline, push, share sheet, store submission | FR-BKP, FR-FRE, FR-OFF, FR-NOT-001 (push) |
| M12+ | R3 (P2) | Enhancements per §5.3 | P2 items |

### 17.2 Definition of done (every requirement)

- All ACs covered by automated tests (unit, plus integration/E2E where the AC spans services or clients).
- The authorisation and visibility rule is tested with at least: owner, other member, album viewer, album contributor, removed member, vault item, trashed item.
- No secrets, no TODO placeholders. Unimplemented sub-features show "X coming soon" and have a tracked GitHub issue.
- Audit events emitted where FR-AUD-001 requires them.
- Cost impact noted in the PR description when a change adds AWS resources or per-item work.
