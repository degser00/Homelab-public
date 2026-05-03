# ADR-0010: Email Archiving and Ingestion via OpenArchiver + Paperless

**Date:** 2026-01-07  
**Status:** Accepted  
**Deciders:** Owner  
**Relates to:** ADR-0006 (Observability Core), ADR-0007 (Telemetry & Action Flow), ADR-0008 (Grafana → n8n Automation Bridge)

---

## Context

Emails represent a long-term personal archive and may contain legally or operationally relevant documents.  
The system must support **full historical and ongoing email archiving** while ensuring that:

- Email servers are **not modified**
- No emails are marked as read, moved, or altered
- Ingestion is reliable, auditable, and retryable
- Classification and search are handled centrally

Paperless supports direct IMAP ingestion, but this approach:
- Modifies mailbox state (read flags, folder moves)
- Couples document management to live email infrastructure
- Violates the requirement for non-invasive archiving

OpenArchiver is already in use as a passive email archiving solution and stores emails as immutable `.eml` files.  
Paperless excels at OCR, classification, tagging, and long-term document management when ingesting from files.

A clear separation of responsibilities is required.

---

## Decision

Adopt a **two-stage architecture**:

- **OpenArchiver** as the system of record for raw emails
- **Paperless** as the system of record for processed, classified documents

Email ingestion into Paperless will be **file-based only**, using scheduled batch copies from OpenArchiver storage into the Paperless consume directory.

Paperless **will not** connect directly to email servers.

---

## Ingestion Flow

```text
Email Server
  |
  |  (passive pull)
  v
OpenArchiver
  |
  |  (immutable .eml storage)
  v
Scheduled Job (cron / automation)
  |
  |  (copy, time-based batches)
  v
Paperless Consume Directory
  |
  v
Paperless
  - OCR
  - AI-based classification
  - Tagging
  - Renaming
  - Final storage
```

Key properties:
- Copy, never move
- Time-based batching (for example daily)
- No per-email ingestion state tracking
- Logs act as the audit and retry reference

---

## Scope & Placement

- OpenArchiver:
  - Archives all configured email accounts
  - Preserves historical and future emails
- Scheduled ingestion job:
  - Copies emails from OpenArchiver storage to Paperless
  - Logs success or failure per batch (date-based)
- Paperless:
  - Owns document lifecycle after ingestion
  - Performs all analysis and organization

All components run as containers and integrate with the existing observability stack.

---

## Security

- No IMAP access from Paperless
- No mailbox credentials exposed beyond OpenArchiver
- No modification of email server state
- Paperless consume directory requires write access only for ingestion
- Logs are collected via Alloy and stored in Loki

---

## Consequences

### Positive
- ✔ Email servers remain untouched
- ✔ Clear separation of concerns
- ✔ Deterministic, retryable ingestion
- ✔ Paperless remains the single source of truth for documents
- ✔ OpenArchiver remains a full immutable fallback
- ✔ Architecture aligns with observability-first design

### Negative
- ⚠ Ingestion is not real-time
- ⚠ Additional storage duplication (.eml + processed documents)
- ⚠ Requires scheduled job management and log monitoring

---

## Alternatives Considered

### 1. Paperless direct IMAP ingestion  
Rejected — modifies mailbox state and violates non-invasive requirement.

### 2. Hybrid IMAP + file ingestion  
Rejected — increases complexity and ambiguity of source of truth.

### 3. OpenArchiver-only (no Paperless)  
Rejected — lacks OCR, AI classification, and document management capabilities.

---

## Future Extensions

- Automate ingestion job via n8n with richer telemetry
- Add per-batch checksum verification
- Optional reprocessing workflows from OpenArchiver
- Enhanced AI classification models
- GitHub issue creation on repeated ingestion failures
