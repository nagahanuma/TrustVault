# TrustVault — Architecture & Solution Approach Note (Python + FastAPI + SQLite)

> **Target Project:** TrustVault (RISE Security Challenge)  
> **Adopted Stack:** **Python 3.11 + FastAPI + SQLite (All-in-One Embedded Engine)**  
> **Target Execution Time:** 2-Hour Fast-Track Build  
> **Core Architecture Principle:** *Zero-Trust Server Authorization, Private Database BLOB Storage, Cryptographic Hash-Chained Audit Ledger, and Instant Revocation.*

---

## 1. Why Python + FastAPI + SQLite is the Winning Choice

1. **Fits 100% Free on Render (512 MB RAM / 0.1 CPU):**
   - Memory footprint is under **40 MB RAM**.
   - Boots up in under 2 seconds.
   - Deployable via native Python runtime on Render with zero Docker overhead.
2. **Crystal-Clear Interview Check (The Authorization Line):**
   - In FastAPI, authorization is an explicit, verifiable dependency:
     ```python
     @router.get("/documents/{doc_id}/download")
     def download_document(
         doc_id: str,
         current_user: User = Depends(get_current_active_user),
         permission: Permission = Depends(verify_document_download_access),  # <-- THE AUTHORIZATION LINE
         db: Session = Depends(get_db)
     ):
         ...
     ```
   - If an evaluator removes `permission: Permission = Depends(...)`, the automated mutation test suite immediately fails.
3. **Zero-Configuration All-In-One Storage (SQLite BLOB):**
   - Both metadata and binary document bytes are stored directly inside `trustvault.db`.
   - No S3 buckets, no cloud storage keys, no directory traversal vulnerabilities.
   - Database is automatically seeded on startup with sample users (`admin`, `owner`, `viewer`) and documents.
4. **Single-Service Deployment:**
   - FastAPI serves the modern responsive UI directly from `/static`, avoiding CORS issues, deployment lag, and multi-service complexity.

---

## 2. Solving the 3 Non-Negotiables

### A. Non-Negotiable 1: The Server Decides Every Time
- All access decisions are calculated in `app/authorization.py` directly against the database records.
- Rules:
  1. Unauthenticated request $\rightarrow$ `401 Unauthorized`.
  2. Document Owner $\rightarrow$ `ALLOW`.
  3. User has active named record in `document_permissions` $\rightarrow$ `ALLOW`.
  4. Any other user $\rightarrow$ Log `DENIED` attempt in `audit_logs`, return `403 Forbidden`.

### B. Non-Negotiable 2: Verifiable, Tamper-Evident Audit Trail
- Every access, download, permission change, and refused attempt is stored in `audit_logs`.
- **SHA-256 Hash Chaining:**
  $$\text{CurrentHash} = \text{SHA256}(\text{PreviousHash} + \text{SequenceId} + \text{Timestamp} + \text{UserId} + \text{DocId} + \text{Action} + \text{Status})$$
- Verification endpoint: `GET /api/audit/verify` scans the chain and confirms 100% mathematical integrity.

### C. Non-Negotiable 3: Immediate Access Revocation
- When an owner clicks "Revoke", the record in `document_permissions` is deleted/revoked in SQLite immediately.
- The next millisecond's download request queries SQLite and is instantly rejected with `403 Forbidden`.

---

## 3. Human-in-the-Loop AI Guardrail

- The AI service analyzes uploaded documents and suggests potential reviewers/viewers.
- **Security Invariant:** The AI cannot modify permissions. It creates an `AiSuggestion` record with status `PENDING`.
- Only the human Document Owner can approve the suggestion, which logs both the AI suggestion and human approval into the tamper-evident audit trail.

---

## 4. Test Suite & Mutation Defense

- `tests/test_security.py`:
  - `test_unauthorized_user_cannot_download()`
  - `test_revoked_permission_blocks_instantaneously()`
  - `test_refused_attempt_logged_to_audit_trail()`
  - `test_tampered_audit_chain_is_detected()`
- `tests/test_mutation.py`:
  - Programmatically comments out the authorization check in `routes/documents.py` and asserts that tests fail.

---

## 5. Mandatory Deliverables Checklist

- [ ] Complete codebase (`app/`, `static/`, `tests/`)
- [ ] `THREATS.md` (Threat vectors, mitigations, trade-offs)
- [ ] `PROMPTS.md` (5 dated entries detailing AI corrections)
- [ ] Working automated test suite (`pytest`)
- [ ] `README.md` with demo credentials and live deployment instructions
