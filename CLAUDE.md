# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a **design document repository** for the Octoco Backend Design Quiz (Intermediate Level). The purpose of this project is to design a backend system according to the requirements set out in `Intermediate Level Backend Design.pdf`. The system manages user deposits and withdrawals using Revio Pay as the third-party payment provider.

**This is NOT a code implementation**. The deliverable is documentation that demonstrates backend design skills including architecture, data modeling, webhook handling, concurrency control, and security.

## Design Approach

The quiz document has numbered sections (1-8). The approach for comparing design options and making design decisions is:

1. **Create a subdirectory for each section** in the quiz PDF (e.g., `1-tech-stack/`, `2-system-architecture/`)
2. **Create markdown files** in that subdirectory which describe the design options available
3. **Make a recommendation** for the best choice, with justification
4. **Consider choices made in previous sections** when evaluating options for later sections

For example, Section 1 chose Python + Django + PostgreSQL, so Section 2 (System Architecture) considers Django's architecture patterns when recommending a modular monolith approach. Section 4 (Flows) references the database locking strategy chosen in Section 3.

**Final Deliverable**: When all sections (1-8) are complete, a final comprehensive design document (markdown) will be created that describes the full system design based on all the decisions made in each section.

## Document Structure

The repository is organized into 8 numbered sections that correspond to the quiz requirements in `Intermediate Level Backend Design.pdf`:

1. **Tech Stack** (`1-tech-stack/`) - Technology choices and justifications (Python, Django, PostgreSQL, Railway)
2. **System Architecture** (`2-system-architecture/`) - Modular monolith architecture with JWT authentication and RBAC
3. **Database Design** (`3-database-design/`) - Schema with pessimistic locking and SERIALIZABLE isolation
4. **Deposit & Withdrawal Flows** (`4-deposit-withdrawal-flows/`) - Sequence diagrams for successful payment flows
5. **Webhook Handling** (`5-webhook-handling/`) - RSA-SHA256 signature verification and idempotency
6. **Security** (`6-security/`) - Authentication, authorization, and attack prevention strategies
7. **Deployment & Hosting** (`7-deployment-hosting/`) - Railway deployment with Docker containers and CI/CD
8. **Failure Handling** (`8-failure-handling/`) - Edge cases, recovery mechanisms, and reconciliation

Additional files:
- `REQUIREMENTS.md` - Checklist tracking completion of all quiz requirements
- `REVIO_API_FEATURES.md` - Summary of Revio Pay API capabilities
- `Intermediate Level Backend Design.pdf` - Original quiz requirements

## Design Philosophy

### Critical Design Principles

**Section 4 focuses on successful flows, Section 8 focuses on failures:**
- Section 4 (Deposit & Withdrawal Flows) should show clean sequence diagrams of the **happy path**
- Error handling should be mentioned briefly in Section 4 with cross-references to Section 8
- Section 8 (Failure Handling) contains detailed failure scenarios, recovery procedures, and edge cases
- Do NOT include extensive failure sequence diagrams in Section 4

**Balance reservation pattern for withdrawals:**
- Balance must be deducted IMMEDIATELY when creating withdrawal (prevents double-spending)
- If payout fails at any stage, the reserved balance is refunded
- Uses pessimistic locking (`SELECT FOR UPDATE`) to prevent race conditions

**Asynchronous payout processing:**
- Withdrawals go through states: PENDING → PROCESSING → COMPLETED/FAILED
- Revio may accept a payout initially but fail later during bank transfer (invalid account, bank rejection)
- This "async failure" scenario is covered in Section 8 as "Payout Accepted But Fails During Bank Processing"

**Idempotency everywhere:**
- Webhooks may be retried by Revio (up to 36 hours with exponential backoff)
- Every webhook handler checks if event was already processed
- Database transactions ensure atomic balance updates

## Key Technical Concepts

**Concurrency Control:**
- SERIALIZABLE isolation level for financial transactions
- Pessimistic locking with `SELECT ... FOR UPDATE` on user balance
- Event ID deduplication for webhooks

**Webhook Trust Model:**
- RSA-SHA256 signature verification using Revio's public key
- Timestamp validation (reject webhooks older than 5 minutes)
- Event ID tracking to prevent replay attacks
- Optional IP allowlist for additional security

**Transaction State Machine:**
- Deposits: PENDING → COMPLETED/FAILED
- Withdrawals: PENDING → PROCESSING → COMPLETED/FAILED
- Reconciliation cron job handles stuck transactions

## Working with Diagrams

All sequence diagrams use Mermaid syntax. Key diagram conventions:

- `activate`/`deactivate` blocks show synchronous processing
- `Note over` annotations explain timing and asynchronous operations
- `alt`/`else` blocks show branching logic (use sparingly in Section 4)
- Database transactions are shown with BEGIN TRANSACTION / COMMIT / ROLLBACK

## Design Tradeoffs Documented

The design explicitly discusses tradeoffs such as:
- Modular monolith vs microservices (chose monolith for simplicity, can evolve later)
- Immediate balance refund vs holding reserved funds (chose refund for better UX)
- Synchronous webhook verification vs queue everything (chose sync verification to reject invalid webhooks fast)
- Manual vs automatic balance reconciliation (chose manual with admin approval for financial safety)

## Section Cross-References

When editing one section, be aware of cross-references:
- Section 3 (Database) defines the `webhook_logs` table used in Section 5
- Section 4 (Flows) references Section 3 for locking strategy and Section 8 for failure handling
- Section 5 (Webhooks) references Section 3 for audit logging table structure
- Section 8 (Failures) provides detailed explanations for scenarios mentioned in Sections 4 and 5

## Terminology Standards

Use proper technical terminology consistently:
- **Pessimistic locking** (not "row locking" or "database locks")
- **SERIALIZABLE isolation** (not "high isolation level")
- **Idempotency** (not "duplicate prevention")
- **Dead letter queue** or **DLQ** (not "failed jobs table")
- **Reconciliation** (for comparing our records with Revio's)
- **Webhook signature verification** (not "webhook validation")
