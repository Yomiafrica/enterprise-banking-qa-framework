# Core Banking Ledger & Compliance Database Audit

## 📌 Project Overview
This module models a high-fidelity relational MySQL database framework tracking core retail banking movements. The core objective is executing data-integrity audits to catch critical business logic failures and regulatory violations before code modifications hit staging environments.

## 🗂️ Relational Schema Tested
- **customer_profiles:** Houses centralized regulatory limits matching standard CBN KYC tiers linked via BVN tracking metrics.
- **ledger_accounts:** Implements 10-digit NUBAN system structures enforcing hard balances and account security lock states.
- **transaction_ledger:** Logs transactional velocity, multi-channel routes (USSD, Web, Mobile App), and processing codes.

## 🔍 Defect Scenarios Identified & Remediated
1. **Security & Isolation Breach:** Intercepted successful debit requests processing out of explicitly `FROZEN` ledger accounts.
2. **Regulatory Limit Underflow:** Isolated transactions matching automated `SUCCESS` states that exceeded strict daily KYC transfer limits for Tier 1 users.

---

## 💻 Source Code & Test Queries

```sql
-- 1. Setup Enterprise Customer Profiles with Regulatory Metadata
CREATE TABLE customer_profiles (
    customer_id INT AUTO_INCREMENT PRIMARY KEY,
    bvn_hash VARCHAR(64) NOT NULL UNIQUE,
    kyc_tier INT NOT NULL CHECK (kyc_tier IN (1, 2, 3)),
    daily_transfer_limit DECIMAL(15, 2) NOT NULL,
    risk_score INT DEFAULT 0 CHECK (risk_score BETWEEN 0 AND 100),
    is_politically_exposed BOOLEAN DEFAULT FALSE
);

-- 2. Setup Ledger Accounts enforcing Strict NUBAN validation patterns
CREATE TABLE ledger_accounts (
    nuban_account_no CHAR(10) PRIMARY KEY,
    customer_id INT,
    account_type VARCHAR(20) CHECK (account_type IN ('SAVINGS', 'CURRENT', 'DOMICILIARY')),
    account_status VARCHAR(20) DEFAULT 'ACTIVE' CHECK (account_status IN ('ACTIVE', 'FROZEN', 'DORMANT')),
    available_balance DECIMAL(15, 2) NOT NULL DEFAULT 0.00,
    book_balance DECIMAL(15, 2) NOT NULL DEFAULT 0.00,
    FOREIGN KEY (customer_id) REFERENCES customer_profiles(customer_id)
);

-- 3. Setup Transaction Ledger with Channel and Compliance Audit fields
CREATE TABLE transaction_ledger (
    transaction_reference VARCHAR(50) PRIMARY KEY,
    source_nuban CHAR(10),
    destination_nuban CHAR(10),
    amount DECIMAL(15, 2) NOT NULL CHECK (amount > 0),
    transaction_type VARCHAR(10) CHECK (transaction_type IN ('DEBIT', 'CREDIT')),
    channel VARCHAR(15) CHECK (channel IN ('MOBILE_APP', 'WEB', 'USSD', 'ATM', 'POS')),
    status_code VARCHAR(10) CHECK (status_code IN ('SUCCESS', 'PENDING', 'FAILED', 'REVERSED')),
    flagged_by_aml BOOLEAN DEFAULT FALSE,
    initiated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ==========================================
-- SEEDING COMPLEX ENTERPRISE MOCK DATA
-- ==========================================
INSERT INTO customer_profiles (bvn_hash, kyc_tier, daily_transfer_limit, risk_score, is_politically_exposed) VALUES 
('bvn_hash_chidera_001', 3, 5000000.00, 10, FALSE),
('bvn_hash_festus_002', 1, 50000.00, 45, FALSE),  
('bvn_hash_suspect_003', 2, 200000.00, 85, TRUE);  

INSERT INTO ledger_accounts (nuban_account_no, customer_id, account_type, account_status, available_balance, book_balance) VALUES 
('0123456789', 1, 'CURRENT', 'ACTIVE', 12500000.00, 12500000.00),
('0987654321', 2, 'SAVINGS', 'ACTIVE', 45000.00, 45000.00),
('0555555555', 3, 'CURRENT', 'FROZEN', 8900000.00, 8900000.00), 
('0444444444', 2, 'SAVINGS', 'ACTIVE', 12000.00, 12000.00);

INSERT INTO transaction_ledger (transaction_reference, source_nuban, destination_nuban, amount, transaction_type, channel, status_code, flagged_by_aml) VALUES 
('TXN-20260601-001', '0123456789', '0987654321', 25000.00, 'DEBIT', 'MOBILE_APP', 'SUCCESS', FALSE),
('TXN-20260601-002', '0555555555', '0123456789', 500000.00, 'DEBIT', 'WEB', 'SUCCESS', FALSE), 
('TXN-20260601-003', '0987654321', '0123456789', 95000.00, 'DEBIT', 'USSD', 'SUCCESS', FALSE);


-- ==========================================
-- QA COMPLIANCE TESTING QUERIES
-- ==========================================

-- Test 1: Identify Security & Account Lock Violations (Frozen Account Leak)
SELECT t.transaction_reference, t.source_nuban, la.account_status, t.amount, t.status_code
FROM transaction_ledger t
JOIN ledger_accounts la ON t.source_nuban = la.nuban_account_no
WHERE la.account_status = 'FROZEN' AND t.status_code = 'SUCCESS';

-- Test 2: Detect AML/Compliance Engine Failure (Daily Transfer Limit Violation)
SELECT t.transaction_reference, t.source_nuban, cp.kyc_tier, t.amount, cp.daily_transfer_limit
FROM transaction_ledger t
JOIN ledger_accounts la ON t.source_nuban = la.nuban_account_no
JOIN customer_profiles cp ON la.customer_id = cp.customer_id
WHERE t.status_code = 'SUCCESS' AND t.amount > cp.daily_transfer_limit;
