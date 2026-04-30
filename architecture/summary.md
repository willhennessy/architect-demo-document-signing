# Architecture Summary

## System Purpose
Multi-tenant B2B document signing platform that lets tenant admins author and dispatch envelopes, lets external signers sign via single-use email magic links, fires HMAC-signed webhooks to customer systems on state changes, and keeps a tamper-evident audit trail for every meaningful transition. Targeted at a credible MVP delivered in 10 weeks by a 5-engineer team.

## Repo Archetype
`modular_monolith`. A Next.js web/API monolith for the request path, plus four concern-specific worker pools (pdf-sealing, webhook-delivery, audit, notifications) that all share a single Postgres source-of-truth. Postgres also hosts the per-concern pg-boss queues. Module boundaries inside the monolith (document, template, signing, audit, webhook, notification) are the seams for any further split.

## Primary Containers or Modules
- Web / API Monolith (`container-web`): admin SSR, signer SSR, REST API, tenancy + RBAC enforcement, transactional writes (including sync-path audit appends), and in-transaction job enqueue across the per-concern queues.
- PDF Sealing Pool (`container-pdf-pool`): CPU-bound worker pool that subscribes only to the pdf-sealing queue and writes sealed PDFs to S3.
- Webhook Delivery Pool (`container-webhook-pool`): retry-heavy I/O worker pool, isolated outbound egress, the only worker that talks to arbitrary customer URLs.
- Audit Pool (`container-audit-pool`): compliance-critical, single-writer for async-path audit appends, plus periodic chain-integrity verification; alert-on-error.
- Notifications Pool (`container-notifications-pool`): email + reminder + expiration handling.
- Postgres (`container-postgres`): canonical store for all business state plus the per-concern pg-boss queues; row-level multi-tenancy with RLS backstop; append-only audit and signature tables enforced by a restricted DB role.

## Critical Flows
- Send a document: admin clicks Send → web monolith writes document/recipients/signature_requests and enqueues `send_invitation` jobs onto the notifications queue in one transaction → notifications pool mints magic-link tokens, sends invitation emails, and enqueues `append_audit` for the audit pool.
- Sign a document: signer clicks magic link → web validates token, mints a scoped signer cookie → on submit, web writes `signature_event` + sync-path `audit_event` and enqueues `seal_pdf`, `fire_webhook`, `notify_sender` onto their respective queues, all in one transaction → pdf-sealing pool seals the PDF to S3, webhook-delivery pool delivers signed webhooks with retry/DLQ, notifications pool sends the completion email, and each pool enqueues `append_audit` jobs for the audit pool.
- Webhook delivery: webhook-delivery pool pulls due `webhook_deliveries` row → HMAC-signs payload → POSTs with timeout → 2xx delivers, non-2xx triggers exponential backoff with jitter, terminal failure lands in DLQ visible in admin UI for replay. Slow customer endpoints can only stall this pool.
- Audit chain verification: audit pool runs a nightly job that walks each document's audit chain end-to-end and flags any broken link. The audit pool also processes all async-path `append_audit` jobs so chain ordering stays under one writer per document.

## Key Decisions
- [DEC-001] Modular monolith for the web tier + four concern-specific worker pools (pdf-sealing, webhook-delivery, audit, notifications), each subscribing only to its own pg-boss queue and scaling on its own profile | covers: container-web,container-pdf-pool,container-webhook-pool,container-audit-pool,container-notifications-pool,view-container,view-component-pdf-pool,view-component-webhook-pool,view-component-audit-pool,view-component-notifications-pool
- [DEC-002] Shared-schema multi-tenancy with `tenant_id` on every row plus Postgres RLS as a defense-in-depth backstop | covers: container-postgres,comp-tenant-rbac-mw,view-component-monolith
- [DEC-003] Postgres-backed job queues — one logical queue per concern — co-located with the SoT to enable transactional enqueue with business writes | covers: container-postgres,comp-jobrunner-pdf,comp-jobrunner-webhook,comp-jobrunner-audit,comp-jobrunner-notifications,rel-web-enqueues-jobs
- [DEC-004] WorkOS for admin SSO (SAML/OIDC + SCIM); magic-link single-use opaque tokens for signers, scoped to one document | covers: ext-workos,comp-auth,comp-magic-link,comp-signer-ui
- [DEC-005] Append-only hash-chained `audit_events` table enforced by a restricted DB role; audit pool is the single writer for async-path appends, with a periodic verifier job; sync-path appends happen inline via the web monolith's audit writer | covers: comp-audit-writer,comp-audit-async-appender,comp-audit-finalizer,container-audit-pool,container-postgres
- [DEC-006] Webhooks delivered exclusively from the webhook-delivery pool via a durable `webhook_deliveries` table with HMAC, exponential backoff with jitter, capped retries, DLQ, and admin-UI replay | covers: comp-webhook-mgr,comp-webhook-deliverer,container-webhook-pool,rel-webhook-pool-to-customer,ext-customer-system
- [DEC-007] Server-side PDF sealing in a dedicated CPU-bound pool; sealed PDF stored in S3 and SHA-256 of sealed bytes persisted on the document | covers: comp-pdf-sealer,container-pdf-pool,ext-s3
- [DEC-008] Single-region (US) MVP topology on Fly/Render with managed Postgres (Neon/RDS) and one compute pool per worker concern; multi-region is post-MVP | covers: view-deployment,node-app,node-worker-pdf,node-worker-webhook,node-worker-audit,node-worker-notifications,node-db,node-edge

## Data Ownership Notes
- Tenant directory, memberships, roles: Postgres (system of record).
- Documents, recipients, fields, templates: Postgres (system of record).
- Signature events: Postgres, append-only at the app layer and DB role layer.
- Audit events: Postgres, append-only and hash-chained per document. Sync-path writer is `comp-audit-writer` (in container-web); async-path writer is `comp-audit-async-appender` (in container-audit-pool). One async writer per document keeps chain ordering under control.
- Webhook subscriptions and deliveries: Postgres (delivery history is the source of truth for audits and replays).
- Magic-link tokens: Postgres, hashed at rest, single-use, short TTL.
- Job queue rows: Postgres via pg-boss — one queue per concern (pdf-sealing, webhook-delivery, audit, notifications), all in the same DB as business state for transactional enqueue.
- Document binary content (raw + sealed PDFs, signature images): Object Storage (S3/R2) with tenant-scoped key prefixes; SHA-256 of sealed bytes is mirrored back into Postgres.
- Admin identity (federated): WorkOS is the system of record for the federated identity; Postgres mirrors only the membership and tenant-scoped role.

## Major Risks or Unknowns
- Compliance bar beyond ESIGN/UETA (eIDAS QES, HIPAA, 21 CFR Part 11) is not yet decided and could materially change audit and signer-identity requirements.
- Final cloud target (Fly/Render vs AWS) is undecided; enterprise procurement may force a re-platform.
- Data residency (US-only vs EU) is undecided; affects DB and blob placement.
- Public API surface scope at MVP is not yet locked.
- Template engine scope (static vs conditional) is not yet locked.
- Hash-chain integrity story has not yet been reviewed by counsel; legal sign-off is required before GA.
- Per-concern pg-boss queue ceilings (~100 jobs/sec range each) are fine for MVP but should be revisited if a tenant drives high webhook fanout.
- Splitting async runtimes adds four more deploy units to operate; per-pool scaling policies and dashboards need to exist on day one.

## Recommended Next Reads
- `architecture/views/system-context.yaml`: who uses the system and what it depends on.
- `architecture/views/container.yaml`: where canonical state lives and how sync/async paths split across the four worker pools.
- `architecture/views/component-monolith.yaml`: module boundaries inside the web/API monolith and the security path.
- `architecture/views/component-pdf-pool.yaml`: what runs inside the pdf-sealing pool.
- `architecture/views/component-webhook-pool.yaml`: what runs inside the webhook-delivery pool.
- `architecture/views/component-audit-pool.yaml`: what runs inside the audit pool.
- `architecture/views/component-notifications-pool.yaml`: what runs inside the notifications pool.
- `architecture/views/deployment.yaml`: MVP topology and per-pool compute choices.

## Artifact Index
- `architecture/model.yaml`: canonical architecture model
- `architecture/views/system-context.yaml`: system context view
- `architecture/views/container.yaml`: container view
- `architecture/views/component-monolith.yaml`: component view of the web/API monolith
- `architecture/views/component-pdf-pool.yaml`: component view of the pdf-sealing pool
- `architecture/views/component-webhook-pool.yaml`: component view of the webhook-delivery pool
- `architecture/views/component-audit-pool.yaml`: component view of the audit pool
- `architecture/views/component-notifications-pool.yaml`: component view of the notifications pool
- `architecture/views/deployment.yaml`: deployment view
- `architecture/summary.md`: this summary
- `architecture/manifest.yaml`: artifact manifest and scope
