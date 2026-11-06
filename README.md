# TrustVault — Secure Document Sharing with a Real Audit Trail (RISE)

> **Core Philosophy:** *"This is a security problem wearing the clothes of a CRUD application."*  
> **Built for:** RISE 7-Day Security Challenge  
> **Stack:** Python 3.11, FastAPI, SQLite (Encrypted Embedded BLOB Storage), Vanilla/Tailwind Reactive UI

---

## 1. Quick Demo Credentials (Pre-Seeded)

The application automatically seeds with four distinct role accounts:

| Role | Email | Password | Allowed Capabilities |
|:---|:---|:---|:---|
| **System Administrator** | `admin@trustvault.com` | `AdminPass123!` | System oversight, full audit ledger view, cryptographic chain verification, emergency doc management. |
| **Document Owner** | `owner@trustvault.com` | `OwnerPass123!` | Upload documents, grant/revoke named user permissions, approve/reject AI suggestions. |
| **Authorized Viewer** | `viewer@trustvault.com` | `ViewerPass123!` | View and download documents explicitly granted by owners. |
| **Red-Team / Attacker** | `unauthorized@trustvault.com` | `UserPass123!` | Attempt unshared downloads (demonstrates server-side 403 refusal & audit logging). |

---

## 2. Local Setup & Running on Your Laptop

### Prerequisites
- Python 3.10+ installed
- Git installed

### Step 1: Clone and Navigate to Directory
```bash
cd e:/2026/sai_kiran/new_requirement
```

### Step 2: Create and Activate Virtual Environment (Recommended)
```bash
# On Windows PowerShell:
python -m venv venv
.\venv\Scripts\activate

# On Linux/macOS:
python3 -m venv venv
source venv/bin/activate
```

### Step 3: Install Dependencies
```bash
pip install -r requirements.txt
```

### Step 4: Run the Application
```bash
python -m uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

### Step 5: Open in Your Browser
- **Main Application & Document Explorer:** Open [http://localhost:8000](http://localhost:8000)
- **Administrative Audit Ledger & Tamper Verification:** Open [http://localhost:8000/admin](http://localhost:8000/admin)
- **Interactive Swagger API Documentation:** Open [http://localhost:8000/docs](http://localhost:8000/docs)

*(Note: On the first launch, the database `trustvault.db` is automatically created, schema applied, and pre-seeded with sample users, confidential documents, and initial audit entries!)*

---

## 3. How to Deploy to Render (100% Free)

You can deploy TrustVault to Render in less than 3 minutes:

### Option A: 1-Click Blueprint (Simplest)
1. Push this folder to a GitHub repository:
   ```bash
   git init
   git add .
   git commit -m "feat: complete TrustVault platform"
   git remote add origin https://github.com/<your-username>/trustvault.git
   git push -u origin main
   ```
2. Log in to [Render.com](https://render.com).
3. Click **New +** $\rightarrow$ **Blueprint**.
4. Connect your GitHub repository. Render will automatically read [`render.yaml`](file:///render.yaml).
5. Click **Apply**. Render will automatically build and deploy the web service.

### Option B: Manual Web Service on Render
1. In Render, click **New +** $\rightarrow$ **Web Service**.
2. Select your connected GitHub repo.
3. Configure the settings:
   - **Environment:** `Python`
   - **Build Command:** `pip install -r requirements.txt`
   - **Start Command:** `uvicorn app.main:app --host 0.0.0.0 --port $PORT`
   - **Plan:** `Free`
4. In **Environment Variables**, add:
   - `SECRET_KEY` = `<any-random-string>`
   - `DATABASE_URL` = `sqlite:///./trustvault.db`
   - *(Optional)* `GROQ_API_KEY` = `<your-groq-api-key>` (if left blank, intelligent heuristic fallback operates seamlessly).
5. Click **Deploy Web Service**.
6. When deployment finishes, open your live URL: `https://<your-service>.onrender.com`.

---

## 4. Running the Automated Security Tests & Mutation Defense

The challenge explicitly requires proving that tests catch security bypasses:
> *"We will take one authorisation check out of your code and run your tests. If they still pass, they were not testing anything."*

### Run All Security Tests:
```bash
pytest tests/ -v
```

### What the Test Suite Proves:
1. `test_unauthenticated_user_cannot_access_documents`: Unauthenticated requests receive `401 Unauthorized`.
2. `test_idor_unauthorized_user_cannot_download_restricted_doc`: Unauthorized users attempting IDOR direct downloads receive `403 Forbidden`, and their refusal is immutably written to `audit_logs`.
3. `test_immediate_access_revocation_lifecycle`: Revoking an access grant cuts off download access immediately on the next request.
4. `test_executable_upload_blocked`: Malicious scripts (`.sh`, `.exe`) are rejected with `400 Bad Request`.
5. `test_audit_trail_tamper_detection`: Modifying an audit row breaks the SHA-256 chain and fails `/api/audit/verify`.
6. `test_ai_suggestion_requires_human_approval`: Verifies that AI never widens permissions autonomously.
7. `test_mutation_defense_demonstration`: Verifies that if `app/authorization.py:verify_document_access` is bypassed, the test suite catches it immediately.

---

## 5. API Documentation & Access Rules Matrix

| Method | Endpoint | Allowed Roles | Description & Authorization Rule |
|:---|:---|:---|:---|
| `POST` | `/api/auth/register` | Public | Register new user account. |
| `POST` | `/api/auth/login` | Public | Authenticate and obtain JWT token. Refused logins are audited. |
| `GET` | `/api/auth/me` | Authenticated | Retrieve current user profile. |
| `GET` | `/api/auth/users` | Authenticated | List active users for sharing dropdown. |
| `POST` | `/api/documents/upload` | `OWNER`, `ADMIN` | Upload document; stored safely as BLOB; triggers AI suggestions. |
| `GET` | `/api/documents` | Authenticated | List documents. Server returns **only** documents user is permitted to see. |
| `GET` | `/api/documents/{id}` | Permitted User | View document metadata. Gated by `verify_document_read_access`. |
| `GET` | `/api/documents/{id}/download` | Permitted User | **The Authorization Checkpoint:** Streams file bytes directly from DB. Gated by `verify_document_read_access`. |
| `DELETE` | `/api/documents/{id}` | `OWNER`, `ADMIN` | Soft-deletes document. Other users receive `403`. |
| `POST` | `/api/documents/{id}/permissions` | `OWNER`, `ADMIN` | Grant `READ`/`EDIT` to named user email. Audited. |
| `DELETE` | `/api/documents/{id}/permissions/{user_id}` | `OWNER`, `ADMIN` | **Immediate Revocation:** Terminates access immediately. |
| `GET` | `/api/audit` | `ADMIN` | View audit trail with search and filter. Non-admins receive `403`. |
| `GET` | `/api/audit/verify` | `ADMIN` | Cryptographic verification of SHA-256 hash chain from Genesis. |
| `GET` | `/api/ai/suggestions` | `OWNER`, `ADMIN` | Fetch pending AI suggestions for owned documents. |
| `POST` | `/api/ai/suggestions/{id}/decision` | `OWNER`, `ADMIN` | Human approval or rejection of AI suggestion. |

---

## 6. Architecture & Non-Negotiable Standards Defense

### 1. The Server Decides Every Time
- Route guards and client-side button hiding are solely for user convenience.
- Authorization is executed via FastAPI dependencies (`verify_document_read_access`) in [`app/authorization.py`](file:///app/authorization.py).
- Requests from curl, scripts, or Postman undergo identical validation against active database records.

### 2. Verifiable, Tamper-Evident Audit Trail
- Each entry in `audit_logs` is cryptographically linked to the previous entry:
  $$\text{Hash}_n = \text{SHA256}(\text{Hash}_{n-1} + \text{SeqId} + \text{Timestamp} + \text{UserId} + \text{DocId} + \text{Action} + \text{Status})$$
- If any entry is modified, deleted, or inserted out of order, `/api/audit/verify` flags the exact sequence ID where the chain was broken.
- **Refused attempts are logged:** Unauthorized downloads trigger `action="DENIED_ACCESS", status="DENIED"` with caller IP and timestamp.

### 3. Immediate Access Revocation
- Pre-signed S3 URLs and long-lived bearer tokens are avoided.
- File streaming reads permissions from the live database transaction.
- When an owner clicks "Revoke", the permission row receives `revoked_at = utc_now()`. Subsequent requests fail with `403 Forbidden` within 0 milliseconds.

---

## 7. Submission Artifacts
- [`THREATS.md`](file:///THREATS.md) — Detailed threat model, mitigations, and accepted trade-offs (15 Marks).
- [`PROMPTS.md`](file:///PROMPTS.md) — 5 dated supervision entries catching AI security mistakes with fixes (15 Marks).

