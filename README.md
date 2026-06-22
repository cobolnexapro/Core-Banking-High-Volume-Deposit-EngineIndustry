# Core-Banking-High-Volume-Deposit-EngineIndustry
IBM i Core Banking System in ILE COBOL + DB2 Embedded SQL. Covers teller 2FA sign-on, Luhn-validated transaction posting, KYC account maintenance, real-time AML scanning (cash aggregation &amp; velocity rules), and daily interest calculation. Built by Nexapro Technologies.
# 🏦 IBM i Core Banking System — COBOL/ILE Source Modules

> **Platform:** IBM i (AS/400) | **Language:** ILE COBOL + Embedded SQL (DB2 for i) | **Author:** Nexapro Technologies

A collection of production-grade IBM i COBOL modules forming the backbone of a Core Banking System. The suite covers teller authentication, real-time financial transaction posting, account maintenance with KYC compliance, Anti-Money Laundering (AML) scanning, and interest calculation — all designed for the IBM i ILE COBOL runtime with DB2 for i as the embedded SQL engine.

---

## 📁 Module Overview

| File | Program ID | Purpose |
|------|------------|---------|
| `BNKSIGN.CBL` | `BNKSIGN` | Teller Sign-On with 2FA & Audit Trail |
| `BNKSIGND.DSPF` | *(Display File)* | 5250 Screen Definition for Sign-On UI |
| `TXNENG01.CBL` | `TXNENG01` | Real-Time Transaction Posting Engine |
| `ACCMNT01.CBL` | `ACCMNT01` | Account Maintenance & KYC Compliance |
| `AML_SCAN.CBL` | `AML_SCAN` | Anti-Money Laundering Policy Scanner |
| `INTCALC-BLOAT.CBL` | `INTCALC-BLOAT` | Daily Interest Calculation *(legacy/refactor candidate)* |

---

## 🔐 BNKSIGN — Teller Sign-On Module

**File:** `BNKSIGN.CBL` + `BNKSIGND.DSPF`

Handles the complete teller authentication lifecycle for the Core Banking terminal, driven by a 5250 workstation display file (`BNKSIGND`).

### Key Features
- **Two-Stage Authentication:** Phase 1 validates Teller ID + Password (via DB2 query against `TELLERMST`). Phase 2 validates a 6-digit 2FA token passed via the Linkage Section.
- **Password Hashing Hook:** A stub call to `BNKCRYPTO` is structured for integration with a dedicated cryptographic module.
- **Account Lockout:** After 3 failed 2FA attempts, the account is automatically locked (`TELLER_STATUS = 'LOCKED'`) and committed to the database.
- **Row-Level Locking:** The `SELECT ... FOR UPDATE` pattern prevents concurrent login races.
- **Full Audit Trail:** Every login attempt (success, denial, lockout) is written to `SECAUDPF` with timestamp, terminal ID, and SQL state.
- **Graceful DB Error Handling:** Any unexpected SQL state triggers a `ROLLBACK` and safe program exit.

### Database Tables Used
| Table | Operation |
|-------|-----------|
| `TELLERMST` | SELECT (with lock), UPDATE login flag & timestamps |
| `SECAUDPF` | INSERT audit log entries |

### Display File (`BNKSIGND`)
- 24×80 3270/5250 screen format (`*DS3`)
- Fields: `SCRID` (Teller ID), `SCRPWD` (Password, non-display), `SCRTKN` (6-digit token)
- F3=Exit function key wired to indicator 03
- Error message rendered in red high-intensity at line 22

---

## 💳 TXNENG01 — Real-Time Transaction Posting Engine

**File:** `TXNENG01.CBL`

The financial posting core. Accepts an account number, amount, and transaction type (Debit/Credit), validates the account, and atomically posts the ledger entry.

### Key Features
- **Luhn Mod-10 Validation:** Full 12-digit account number check-digit verification before any DB access is attempted — prevents garbage data from reaching the ledger.
- **Explicit Transaction Isolation:** Sets `ISOLATION LEVEL RECORD LOCK` before any balance read, preventing dirty reads.
- **Minimum Balance Enforcement:** Debit transactions are rejected if the resulting balance would fall below `ACC_MIN_BAL`.
- **Atomic Dual-Table Write:** Both `ACCMASTER` (balance update) and `TXNJRNPF` (journal insert) are written within a single transaction. Any failure triggers a full `ROLLBACK`.
- **Fixed-Point Arithmetic:** All financial calculations use `COMP-3` packed decimal to avoid floating-point rounding errors.

### Processing Flow
```
LUHN Validate → Set Isolation → Fetch Balance (lock) → Calc New Balance → Update ACCMASTER → Insert TXNJRNPF → COMMIT
                                                                                          ↓ (any failure)
                                                                                       ROLLBACK
```

### Database Tables Used
| Table | Operation |
|-------|-----------|
| `ACCMASTER` | SELECT (for update), UPDATE balance |
| `TXNJRNPF` | INSERT transaction journal entry |

---

## 🏷️ ACCMNT01 — Account Maintenance & KYC Compliance

**File:** `ACCMNT01.CBL`

An interactive teller-facing module for account record maintenance, with automated KYC (Know Your Customer) document expiry checking.

### Key Features
- **Native RLA with Record Lock:** Uses `READ ... WITH LOCK` on `ACCMASTER` (indexed physical file) for concurrency-safe updates, avoiding SQL overhead for the primary read.
- **Hybrid I/O Pattern:** Combines native RLA (for the account master record) with Embedded SQL (for the KYC document check), showcasing the standard IBM i mixed-access architecture.
- **KYC Expiry Detection:** Queries `KYCDOCPF` for any documents whose `KYC_EXP_DATE` is older than today's `CURRENT TIMESTAMP`. If found, sets `AC-ACCT-STATUS = 'SND'` (suspended) and flags `AC-COMPLIANCE-FLAG = 'Y'`.
- **Lock Release on Failure:** An explicit `UNLOCK ACCMASTER` is issued even on failed rewrites to prevent lock leaks across teller sessions.
- **File Status 88-levels:** Clean, readable conditional logic using `88 ACC-OK`, `88 ACC-LOCKED`, `88 ACC-NOT-FOUND` rather than raw status code comparisons.

### Database Tables / Files Used
| Object | Type | Operation |
|--------|------|-----------|
| `ACCMASTER` | Physical File (RLA) | OPEN I-O, READ WITH LOCK, REWRITE, UNLOCK |
| `KYCDOCPF` | DB2 Table (SQL) | SELECT COUNT(*) for expiry check |

---

## 🚨 AML_SCAN — Anti-Money Laundering Policy Scanner

**File:** `AML_SCAN.CBL`

A real-time AML compliance module called immediately after transaction posting. Evaluates two independent regulatory rules and generates an alert record when either is breached.

### Compliance Rules Enforced

| Rule | Threshold | Logic |
|------|-----------|-------|
| **24-Hour Cash Aggregation** | > $10,000.00 | Sums all `CASH`-type transactions for the account in the last 24 hours + current transaction (if ATM/branch cash). Flags if total exceeds threshold. |
| **1-Hour Velocity Attack** | > 5 transactions | Counts all transactions for the account in the last hour + 1 (current). Flags if count exceeds limit. |

### Risk Scoring Matrix
| Breach | Score Added |
|--------|-------------|
| Cash limit only | +45 |
| Velocity only | +50 |
| Both (combined) | +45 + 50 + 5 = **100 (capped)** |

### Alert Generation
- On any breach, an alert record is inserted into `AMLALERTPF` with full metadata: account, timestamp, violation type, risk score, 24H cash total, velocity count, and transaction location.
- An `LK-ALERT-FLAG = 'Y'` and `LK-RETURN-CODE = '01'` (critical) are returned to the calling transaction engine.
- Uses null indicators (`WS-SQL-INDICATORS`) for safe handling of `COALESCE` SQL results.

### Database Tables Used
| Table | Operation |
|-------|-----------|
| `TXNJRNPF` | SELECT aggregate (SUM, COUNT) |
| `AMLALERTPF` | INSERT alert record |

---

## 📈 INTCALC-BLOAT — Daily Interest Calculation (Legacy / Refactor Candidate)

**File:** `INTCALC-BLOAT.CBL`

Calculates and posts daily simple interest for up to 100 accounts. **This module is flagged as a technical debt item** — it uses a manually unrolled structure instead of COBOL arrays (OCCURS), making it extremely verbose and difficult to maintain.

### What It Does
For each account slot (001–100):
```
Interest = (Balance × Rate) / 36500
New Balance = Balance + Interest
UPDATE ACCMASTER SET CURRENT_BALANCE = :new_balance WHERE ACCOUNT_NUM = :account_num
```
A single `COMMIT` is issued at program termination for all 100 updates.

### ⚠️ Known Issues / Refactor Notes

| Issue | Description |
|-------|-------------|
| **No OCCURS clause** | 400 individual data items (`N-001` to `N-100`, `B-`, `R-`, `I-` series) replace a simple `OCCURS 100 TIMES` table. Likely a legacy constraint or tooling limitation. |
| **No SQL error handling** | Each of the 100 `UPDATE` statements has zero `SQLCODE`/`SQLSTATE` checking. A single failure is silently ignored. |
| **No transaction isolation** | No `SET TRANSACTION` or row-level lock before reading/writing balances — susceptible to phantom reads in a concurrent environment. |
| **Single batch COMMIT** | All 100 updates commit together. A failure mid-batch with no per-record check means partial updates may be committed or silently lost. |
| **Not scalable** | Hard-coded to exactly 100 accounts. Adding account 101 requires source code modification and recompile. |

**Recommended Refactor:** Replace with a cursor-driven SQL loop over `ACCMASTER` using `DECLARE CURSOR ... FOR UPDATE`, with per-row error handling and `COMMIT` frequency control.

---

## 🗄️ Database Schema Summary

| Physical File / Table | Key Columns | Used By |
|-----------------------|-------------|---------|
| `TELLERMST` | `TELLER_ID`, `PASSWORD_HASH`, `TELLER_STATUS`, `LOGGED_IN_FLAG`, `FAILED_ATTEMPTS`, `LAST_LOGIN_TS` | BNKSIGN |
| `SECAUDPF` | `TELLER_ID`, `ACTION_CODE`, `TERM_ID`, `ACTION_TS`, `SQL_ST` | BNKSIGN |
| `ACCMASTER` | `ACC_NUM` / `ACCOUNT_NUM`, `ACC_BALANCE` / `CURRENT_BALANCE`, `ACC_MIN_BAL`, `AC-ACCT-STATUS`, `AC-KYC-EXP-DATE` | TXNENG01, ACCMNT01, INTCALC-BLOAT |
| `TXNJRNPF` | `TXN_ACC_NUM` / `ACCT_NO`, `TXN_AMT`, `TXN_TYPE`, `TXN_TIMESTAMP` / `TXN_TS` | TXNENG01, AML_SCAN |
| `KYCDOCPF` | `KYC_ACCT_NUM`, `KYC_EXP_DATE` | ACCMNT01 |
| `AMLALERTPF` | `ACCT_NO`, `ALERT_TS`, `VIOLATION_TYPE`, `RISK_SCORE`, `TOTAL_CASH_AMT`, `TXN_COUNT`, `ALERT_STATUS`, `TXN_LOC` | AML_SCAN |

---

## 🔗 Module Call Chain

```
Teller Terminal (5250)
        │
        ▼
   BNKSIGN ──────────── (2FA token via Linkage) ──── [BNKCRYPTO - external]
        │
        ▼ (on successful login)
   ACCMNT01  ◄──── Teller-initiated account updates
        │
        ▼
   TXNENG01  ◄──── Transaction request (acct, amount, type)
        │
        ▼
   AML_SCAN  ◄──── Called post-posting by TXNENG01
        │
        ▼
   INTCALC-BLOAT  ◄──── Scheduled batch (EOD interest run)
```

---

## 🛠️ Technical Environment

| Item | Value |
|------|-------|
| Platform | IBM i (AS/400 / iSeries) |
| Language | ILE COBOL (IBM i) |
| Database | DB2 for i (Embedded SQL) |
| SQL Commitment | `*CHG` (record-level), explicit `COMMIT`/`ROLLBACK` |
| Date Format | `*ISO` (YYYYMMDD) |
| Numeric Storage | `COMP-3` (packed decimal) for all financial fields |
| Display Files | 5250 DDS (`.DSPF`) for teller UI |
| File I/O | Native RLA (`READ/REWRITE`) + Embedded SQL hybrid |

---

## 📋 Prerequisites for Compilation

1. IBM i OS (V7R3 or later recommended)
2. ILE COBOL compiler (`CRTCBLPGM` or `CRTBNDCBL`)
3. DB2 for i SQL precompiler (`CRTSQLCBLI`)
4. Source Physical Files in the target library (`QCBLLESRC` or equivalent)
5. All referenced physical files (`TELLERMST`, `ACCMASTER`, etc.) must exist before compilation

### Compile Order
```
1. BNKSIGND   (DDS Display File)      → CRTDSPF
2. BNKSIGN    (depends on BNKSIGND)   → CRTSQLCBLI
3. TXNENG01                            → CRTSQLCBLI
4. ACCMNT01                            → CRTSQLCBLI
5. AML_SCAN                            → CRTSQLCBLI
6. INTCALC-BLOAT                       → CRTSQLCBLI
```

---

## 📌 Notes & Known Limitations

- `BNKSIGN` password hashing (`BNKCRYPTO` call) is currently stubbed — production deployment requires integration with a real cryptographic service program.
- `INTCALC-BLOAT` is intentionally named to flag it as a technical debt item pending refactor.
- Column name inconsistencies exist across modules (e.g., `TXN_ACC_NUM` in TXNENG01 vs `ACCT_NO` in AML_SCAN for the same `TXNJRNPF` table) — these should be reconciled against the physical file's actual DDS definition.
- AML account metadata (`WS-CUST-ID`, `WS-CUST-SEGMENT`) is currently hardcoded as mock data and should be replaced with a lookup against a customer master file.

---

## 🏢 About

Developed by **Nexapro Technologies** — IBM i, Mainframe & Legacy Modernization specialists.

📧 nexaprotechnology@gmail.com | 🌐 nexaprotechnologies.com | 📞 +91 63502 82519
