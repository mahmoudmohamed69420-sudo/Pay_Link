# PayLink — System Requirements Specification (Phase 1)

## 1. System Overview & Problem Summary

The PayLink system is a high-reliability digital wallet and payment backend designed to modernize bill payment in Egypt, operating similarly to services like Fawry and Vodafone Cash. Millions of customers rely on bill-payment services monthly to settle utility bills—including electricity, water, mobile top-ups, and internet subscriptions—either through a mobile application or at physical agent terminals (small retail shops equipped with PayLink terminals). 

By providing a unified digital channel, PayLink eliminates long physical queues and eliminates repetitive, manual tasks for consumers, while enabling utility companies ("billers") to receive payments rapidly and accurately.

PayLink operates on a fee-and-commission business model: charging a small service fee on customer transactions and distributing a percentage as commission to retail agents. Because PayLink holds customer funds in digital wallets, it bears a critical fiduciary and technical responsibility. The system must never overcharge, double-charge, execute unauthorized transactions without customer notice, or permit wallet balances to drop below zero. Every money movement must be atomic, audit-logged, and resilient against network failures, timeouts, and concurrent requests.

---

## 2. Key System Stakeholders

| Stakeholder Role | Description & Primary Interest | Key System Interactions |
| :--- | :--- | :--- |
| **Customer** | End-user paying utility bills via mobile app. Seeks fast, accurate payments, clear fee structures, and immediate transaction confirmation. | Account registration, KYC tier upgrade, wallet top-up, bill inquiry, bill payment, state history viewing. |
| **Agent** | Retail shop owner operating a PayLink terminal. Serves cash-paying walk-in customers and earns commissions on processed payments. | Cash-in top-up execution, paying bills on behalf of walk-in customers, commission wallet accumulation. |
| **Biller (Utility Provider)** | Utility enterprise (ElectroCo, AquaCo, TopUpNet, NetCo) receiving payments via PayLink. Expects accurate settlements and zero balance drift. | Responding to real-time inquiries/payments, issuing asynchronous callbacks, receiving settlement files. |
| **Support Employee** | Internal operations team resolving payment disputes, investigating failed/unknown transactions, and handling fraud alerts. | Searching payment histories by reference ID, reviewing state timelines, manually freezing/unfreezing accounts. |
| **Administrator** | System administrator managing system parameters, pricing strategies, and provider connectivity. | Editing fee rules, configuring promotional zero-fee windows, updating daily tier limits, toggling active billers. |
| **Auditor / Regulator** | Financial compliance bodies (e.g., Central Bank of Egypt) requiring strict data integrity, non-repudiation, and financial safety. | Inspecting append-only immutable ledger records and system audit logs. |

---

## 3. Ambiguities & Architectural Decisions Log

During the requirements elicitation and AI-augmented analysis stage, several domain ambiguities and edge cases were identified. The table below details the questions raised, trade-offs evaluated, and explicit design decisions incorporated into Phase 1.

| ID | Issue / Ambiguity Found | AI Question / Context | Resolution & Architectural Decision |
| :--- | :--- | :--- | :--- |
| **DEC-01** | **Handling Pending Biller Timeout** | What happens to held customer funds when a live biller call takes longer than 10 seconds? | Funds remain reserved in `held_balance`. Payment transitions to `UNKNOWN`. Funds are **never refunded automatically** on timeout; automatic retries ($+1\text{ min}$, $+5\text{ min}$, $+15\text{ min}$) or nightly settlement reconciliation must confirm biller non-receipt before refunding. |
| **DEC-02** | **Simultaneous Wallet Payments** | How should the system handle two payments arriving at the exact same millisecond when funds cover only one? | Enforce database row-level pessimistic locking (`SELECT ... FOR UPDATE`) on the `Wallet` entity inside a single ACID transaction. One payment succeeds (`FUNDS_HELD`); the second fails validation (`REJECTED`) with zero negative balance risk. |
| **DEC-03** | **Network Retries & Duplicate Requests** | How to prevent double charges when a mobile app retries a slow POST payment request? | Mandate a unique client-generated `Idempotency-Key` header. The system checks an `IdempotencyRecord` table before processing; duplicate keys return the cached original response without creating new ledger entries or calling billers twice. |
| **DEC-04** | **Floating-Point Rounding Drift** | Standard floating-point types (`float`, `double`) cause rounding inaccuracies in percentage fees ($1.5\%$) and splits ($40\%$). | Enforce exact decimal types (`DECIMAL(19,4)` in SQL, `BigDecimal` / exact decimal structures in application code) across all entities, APIs, and ledger math. |
| **DEC-05** | **Agent Service Fee & Commission Revenue** | Does an agent pay the customer fee, and how is commission credited? | The customer pays `bill_amount + service_fee`. Upon `COMPLETED` payment status, $40\%$ of the service fee is credited to the agent's wallet via a distinct `AGENT_COMMISSION` ledger entry, and $60\%$ remains as PayLink net revenue. |
| **DEC-06** | **Notification Service Outage** | If the SMS/Push gateway is unavailable during payment completion, should the payment fail or roll back? | Core payment completion is decoupled from notifications. Payment completes successfully; notification delivery is enqueued asynchronously and retried up to 3 times before being marked `UNDELIVERED`. |

---

## 4. Functional Requirements (25 User Stories)

### Category 1: Accounts, Wallet & Ledger

#### US-01: Customer Registration & Basic Tier Initialization `[FR-1 | Priority: Must]`
* **User Story:** As a new customer, I want to register an account with my personal details, so that I receive an active digital wallet with a basic transaction tier.
* **Prerequisite:** User has no existing account under the submitted phone number or national ID, and supplies valid registration payload.
* **Result:** A user profile is created with `BASIC` verification status ($5,000.00\text{ EGP}$ daily limit) and an initialized wallet with $0.00\text{ EGP}$ available and $0.00\text{ EGP}$ held balances.
* **Acceptance Criteria:**
  * **Given** an unregistered user submits valid registration details, **When** the registration request is processed, **Then** the system creates the user with verification level `BASIC`, sets the daily limit to $5,000.00\text{ EGP}$, and provisions an empty EGP wallet.
  * **Given** a phone number is already registered, **When** a duplicate registration is attempted, **Then** the system rejects the request with a conflict error ($409\text{ Conflict}$) and creates no secondary wallet.

#### US-02: Account Upgrade to Verified Tier `[FR-2 | Priority: Should]`
* **User Story:** As a basic-tier customer, I want to complete identity verification, so that my account is upgraded to `VERIFIED` and my daily spending limit increases.
* **Prerequisite:** Customer holds an active `BASIC` account that is not blocked.
* **Result:** Customer verification status updates to `VERIFIED`, raising the daily payment limit to $30,000.00\text{ EGP}$.
* **Acceptance Criteria:**
  * **Given** an active customer has a `BASIC` verification level, **When** their identity verification is approved, **Then** their verification status updates to `VERIFIED` and subsequent daily limit checks enforce $30,000.00\text{ EGP}$.

#### US-03: View Dual-Balance Wallet & Transaction History `[FR-3 | Priority: Must]`
* **User Story:** As a customer, I want to view my available balance, held balance, and transaction history, so that I can track reserved and spent funds accurately.
* **Prerequisite:** Customer is authenticated and owns an initialized wallet.
* **Result:** System returns exact decimal figures for `available_balance` and `held_balance`, alongside a chronological ledger history.
* **Acceptance Criteria:**
  * **Given** a customer has $1,000.00\text{ EGP}$ total funds with $352.50\text{ EGP}$ reserved for an in-flight payment, **When** they query their wallet, **Then** the response displays $647.50\text{ EGP}$ available, $352.50\text{ EGP}$ held, and lists the pending transaction.

#### US-04: Wallet Top-Up via Card or Cash Agent `[FR-4 | Priority: Must]`
* **User Story:** As a customer, I want to top up my wallet balance using a simulated card or cash at a PayLink agent, so that I have funds to pay utility bills.
* **Prerequisite:** Customer account is active and the top-up amount is a positive exact decimal value.
* **Result:** A `TOPUP` credit entry is appended to the ledger, and `available_balance` increases by the top-up amount.
* **Acceptance Criteria:**
  * **Given** an active customer has $100.00\text{ EGP}$ available balance, **When** they complete a $500.00\text{ EGP}$ top-up, **Then** a `TOPUP` ledger entry of $+500.00\text{ EGP}$ is recorded and their available balance becomes $600.00\text{ EGP}$.

#### US-05: Ledger-Derived Balance Integrity `[FR-5 | Priority: Must]`
* **User Story:** As the core accounting engine, I want every monetary movement to be recorded as an immutable ledger entry from which balances are derived, so that balances can never be overwritten directly or drift.
* **Prerequisite:** Any monetary operation (top-up, hold, release, fee capture, commission, refund) executes within an active database transaction.
* **Result:** An immutable ledger record is created using exact decimal math, and the wallet balance equals the mathematical sum of its ledger entries.
* **Acceptance Criteria:**
  * **Given** a wallet has a sequence of top-up, hold, and charge operations, **When** the wallet balance is audited against the `ledger_entries` table, **Then** the wallet balance matches the sum of all immutable entries with zero discrepancy.

---

### Category 2: Biller Catalog & Inquiries

#### US-06: Admin Biller & Service Management `[FR-6 | Priority: Must]`
* **User Story:** As an administrator, I want to add, edit, and disable billers and utility services, so that customers can only pay active, configured providers.
* **Prerequisite:** Actor is authenticated with `ADMIN` privileges.
* **Result:** The catalog reflects updated service states, immediately governing visibility and payment authorization.
* **Acceptance Criteria:**
  * **Given** an admin deactivates the `AquaCo` water service, **When** a customer attempts a bill inquiry or payment for `AquaCo`, **Then** the system rejects the request indicating the biller is currently disabled.

#### US-07: Browse Active Billers & Input Requirements `[FR-7 | Priority: Must]`
* **User Story:** As a customer, I want to view active billers and their required input fields, so that I know what reference information to provide.
* **Prerequisite:** At least one biller is configured as active in the catalog.
* **Result:** System returns active billers, service types, input field schemas (e.g., meter number, phone number), and amount limits.
* **Acceptance Criteria:**
  * **Given** `ElectroCo` (active) and `AquaCo` (active) exist, **When** a customer requests the catalog, **Then** only active billers are returned along with input validation schemas and $5.00\text{ EGP}$–$10,000.00\text{ EGP}$ limits.

#### US-08: Pre-Payment Bill Inquiry & Fee Breakdown `[FR-8 | Priority: Must]`
* **User Story:** As a customer, I want to perform a bill inquiry and view the bill amount, service fee, and total cost before paying, so that I can verify charges prior to committing funds.
* **Prerequisite:** Selected biller is active and supports live inquiry, and valid reference inputs are provided.
* **Result:** System calls the biller adapter and returns `bill_amount`, calculated `service_fee`, and `total_amount`.
* **Acceptance Criteria:**
  * **Given** a customer enters a valid meter number for `ElectroCo` with $350.00\text{ EGP}$ due and no active promotion, **When** they execute a bill inquiry, **Then** the system returns `bill_amount: 350.00`, `service_fee: 2.50`, and `total_amount: 352.50 EGP`.

---

### Category 3: Core Payment Processing

#### US-09: Customer Bill Payment from Wallet `[FR-9 | Priority: Must]`
* **User Story:** As a customer, I want to pay a utility bill directly from my wallet, so that my utility bill is settled immediately.
* **Prerequisite:** Customer is authenticated, active, within daily/velocity limits, and holds `available_balance >= total_amount`.
* **Result:** Funds are held, biller confirms payment, hold converts to final charge, PayLink fee income is credited, and payment reaches `COMPLETED`.
* **Acceptance Criteria:**
  * **Given** a customer has $500.00\text{ EGP}$ available and pays a $350.00\text{ EGP}$ `ElectroCo` bill ($2.50\text{ EGP}$ fee), **When** `ElectroCo` confirms within 10 seconds, **Then** available balance becomes $147.50\text{ EGP}$, held balance is $0.00\text{ EGP}$, PayLink fee revenue is credited $2.50\text{ EGP}$, and status is `COMPLETED`.

#### US-10: Dynamic Fee & Promotion Calculation `[FR-16 | Priority: Must]`
* **User Story:** As the billing engine, I want to calculate service fees dynamically using fixed, percentage, tiered, and promotional rules, so that each bill category is priced correctly.
* **Prerequisite:** Payment amount, biller category, and execution timestamp are known.
* **Result:** Service fee is computed: $0.00\text{ EGP}$ during active promotional dates; otherwise $2.50\text{ EGP}$ (electricity/water), $\max(1.00, 1.5\% \times \text{amount})$ (mobile top-up), or tiered $2.00 / 4.00\text{ EGP}$ (internet).
* **Acceptance Criteria:**
  * **Given** a $50.00\text{ EGP}$ mobile top-up on `TopUpNet` without promotion, **When** fee calculation runs, **Then** $1.5\%$ ($0.75\text{ EGP}$) rounds up to the minimum fee of $1.00\text{ EGP}$.
  * **Given** an active promotion for `ElectroCo`, **When** a customer pays an electricity bill, **Then** the service fee is $0.00\text{ EGP}$.

#### US-11: Agent Payment & Commission Settlement `[FR-17 | Priority: Should]`
* **User Story:** As a PayLink agent, I want to process a payment for a walk-in customer and earn a $40\%$ commission on the service fee, so that I am compensated upon payment completion.
* **Prerequisite:** Agent is authenticated with an active agent account, and payment reaches `COMPLETED`.
* **Result:** $40\%$ of the service fee is credited to the agent's wallet via a ledger entry upon payment completion.
* **Acceptance Criteria:**
  * **Given** an agent processes a $350.00\text{ EGP}$ bill with a $2.50\text{ EGP}$ fee, **When** payment status becomes `COMPLETED`, **Then** $1.00\text{ EGP}$ ($40\%$ of $2.50\text{ EGP}$) is credited to the agent's wallet and $1.50\text{ EGP}$ remains PayLink net fee revenue.

#### US-12: Single-Payment & Daily Limit Enforcement `[FR-18 | Priority: Must]`
* **User Story:** As the compliance engine, I want to reject payments outside the $5.00$–$10,000.00\text{ EGP}$ single range or exceeding daily tier limits, so that system risk thresholds are enforced.
* **Prerequisite:** A payment request enters `INITIATED` state.
* **Result:** Request is `REJECTED` without holding funds if amount is $< 5.00\text{ EGP}$, $> 10,000.00\text{ EGP}$, or exceeds cumulative 24-hour tier limit ($5,000.00\text{ EGP}$ Basic / $30,000.00\text{ EGP}$ Verified).
* **Acceptance Criteria:**
  * **Given** a `BASIC` customer has spent $4,800.00\text{ EGP}$ today, **When** they attempt a $300.00\text{ EGP}$ payment, **Then** the payment transitions from `INITIATED` to `REJECTED` with a clear daily limit error reason.

#### US-13: High-Velocity Fraud Blocking `[FR-19 | Priority: Should]`
* **User Story:** As the fraud engine, I want to block an account for 10 minutes if more than 5 payments occur in 60 seconds, so that automated abuse is prevented.
* **Prerequisite:** Customer submits a payment request in `INITIATED` state.
* **Result:** On the 6th attempt within a rolling 60-second window, payment is `REJECTED` and account status updates to `BLOCKED` for 10 minutes.
* **Acceptance Criteria:**
  * **Given** a customer made 5 payment attempts in the last 45 seconds, **When** they submit a 6th payment request, **Then** the payment is `REJECTED` and the account is blocked for 10 minutes.

---

### Category 4: Atomicity & Biller Adapters

#### US-14: Idempotent Payment Request Processing `[FR-10 | Priority: Must]`
* **User Story:** As an API client, I want repeated requests with the same idempotency key to return the original transaction outcome without charging twice, so that network retries are safe.
* **Prerequisite:** Client includes an `Idempotency-Key` header with the request payload.
* **Result:** First request executes and caches response; duplicate requests return identical response payload without re-executing holds or biller calls.
* **Acceptance Criteria:**
  * **Given** a client sends a payment request with `Idempotency-Key: KEY-7788` and retries due to network delay, **When** both requests arrive, **Then** only one payment executes, wallet balance is debited once, and the second request receives the cached original result.

#### US-15: Payment Lifecycle State Machine Enforcement `[FR-11 | Priority: Must]`
* **User Story:** As the payment orchestrator, I want to enforce valid state transitions (`INITIATED` $\rightarrow$ `FUNDS_HELD`/`REJECTED` $\rightarrow$ `SENT_TO_BILLER` $\rightarrow$ `COMPLETED`/`FAILED`/`UNKNOWN` $\rightarrow$ `REFUNDED`), so that payments never enter inconsistent states.
* **Prerequisite:** Payment entity exists in one of the defined lifecycle states.
* **Result:** System permits only defined state transitions and throws a domain exception on invalid transition attempts.
* **Acceptance Criteria:**
  * **Given** a payment is in `UNKNOWN` state, **When** an operation attempts to transition directly to `REFUNDED` without confirming `FAILED`, **Then** the system blocks the transition and maintains held funds.

#### US-16: Atomic Fund Reservation & Failure Refund `[FR-12 | Priority: Must]`
* **User Story:** As the wallet service, I want to atomically reserve funds before calling a biller and release them if the biller refuses, so that balances never go negative and failed payments return money.
* **Prerequisite:** Payment passes fraud checks (`INITIATED`) and acquires a database lock on the wallet.
* **Result:** Available balance decreases and held balance increases atomically (`FUNDS_HELD`). If biller refuses (`FAILED`), held balance returns to available balance (`REFUNDED`).
* **Acceptance Criteria:**
  * **Given** a wallet has $400.00\text{ EGP}$ and two $352.50\text{ EGP}$ payments execute simultaneously, **When** processed concurrently, **Then** one transitions to `FUNDS_HELD` ($47.50\text{ EGP}$ remaining available) and the second is `REJECTED` for insufficient funds.
  * **Given** `AquaCo` refuses a payment with an error code, **When** refusal is processed, **Then** payment status moves to `FAILED` and then `REFUNDED`, restoring funds to `available_balance`.

#### US-17: Biller Timeout Handling & Scheduled Retry Backoff `[FR-13 | Priority: Must]`
* **User Story:** As the resilience worker, I want payments timing out after 10 seconds to enter `UNKNOWN` and be re-queried at 1, 5, and 15 minutes, so that ambiguous timeouts are resolved safely.
* **Prerequisite:** Payment in `SENT_TO_BILLER` receives no response within 10 seconds.
* **Result:** Payment moves to `UNKNOWN`, funds stay in `held_balance`, and retry worker queries biller status at $+1\text{ min}$, $+5\text{ min}$, and $+15\text{ min}$.
* **Acceptance Criteria:**
  * **Given** `ElectroCo` takes 12 seconds to respond, **When** the 10-second timeout elapses, **Then** payment transitions to `UNKNOWN`, funds remain held, and retry worker schedules status queries at 1, 5, and 15 minutes.

#### US-18: Multi-Protocol Biller Adapter Isolation `[FR-15 | Priority: Must]`
* **User Story:** As the integration layer, I want each biller protocol (`ElectroCo` REST, `AquaCo` SOAP, `TopUpNet` async REST) encapsulated behind a common adapter interface, so that protocol differences do not affect core logic.
* **Prerequisite:** Payment core invokes `BillerPort` interface.
* **Result:** Adapter translates canonical requests into native biller format (JSON, SOAP XML, Async) and maps responses back into domain outcomes.
* **Acceptance Criteria:**
  * **Given** `AquaCo` expects SOAP XML and returns HTTP 200 with a SOAP fault code, **When** `AquaCoAdapter` handles the response, **Then** it translates the fault into a domain `BillerRefused` result without exposing XML to core payment logic.

#### US-19: Asynchronous Webhook Processing & Deduplication `[FR-14 | Priority: Must]`
* **User Story:** As the callback endpoint, I want to process asynchronous callbacks from `TopUpNet` and ignore duplicate webhooks, so that asynchronous top-ups finalize accurately once.
* **Prerequisite:** A `TopUpNet` payment is in `SENT_TO_BILLER` or `UNKNOWN` state.
* **Result:** First callback transitions payment to `COMPLETED` or `FAILED` $\rightarrow$ `REFUNDED`; duplicate callbacks are logged and ignored with HTTP 200.
* **Acceptance Criteria:**
  * **Given** `TopUpNet` sends a `success` callback twice for transaction `TX-9090`, **When** both callbacks arrive, **Then** the first completes the payment and settles ledger entries, while the second is logged and ignored.

---

### Category 5: Notifications

#### US-20: Multi-Channel Outcome Notifications `[FR-22 | Priority: Must]`
* **User Story:** As a customer, I want to receive an SMS and a push notification when my payment completes or fails, so that I have immediate confirmation.
* **Prerequisite:** Payment reaches a final state (`COMPLETED` or `FAILED`/`REFUNDED`).
* **Result:** System dispatches asynchronous notification events for SMS and Push channels using predefined templates.
* **Acceptance Criteria:**
  * **Given** a payment completes successfully, **When** completion event triggers, **Then** SMS and Push notifications containing reference number, biller, and amount are queued for delivery.

#### US-21: Resilient Notification Retries & Dead-Letter Handling `[FR-23 | Priority: Should]`
* **User Story:** As the notification service, I want to retry failed message deliveries up to 3 times before marking them `UNDELIVERED`, so that temporary gateway outages do not drop alerts.
* **Prerequisite:** External SMS or notification gateway fails or times out.
* **Result:** Core payment remains unaffected; notification worker retries delivery 3 times before setting status to `UNDELIVERED`.
* **Acceptance Criteria:**
  * **Given** SMS gateway is down when a payment completes, **When** initial delivery fails, **Then** payment status remains `COMPLETED`, delivery is retried 3 times, and status updates to `UNDELIVERED` if all retries fail.

---

### Category 6: Audit & Reconciliation

#### US-22: Immutable Financial & Administrative Audit Trail `[FR-25 | Priority: Must]`
* **User Story:** As an auditor, I want every money movement and administrative update recorded in an immutable audit log, so that all system operations are fully traceable.
* **Prerequisite:** A ledger entry is created or an admin/support user modifies biller rules, limits, fees, or account statuses.
* **Result:** An append-only audit record capturing actor, role, action, target entity, before/after state, timestamp, and reason is persisted.
* **Acceptance Criteria:**
  * **Given** an admin modifies a fee rule or a ledger charge executes, **When** transaction commits, **Then** an audit record is created capturing actor ID, timestamp, changes made, and rationale.

#### US-23: Nightly Settlement File Reconciliation `[FR-26 | Priority: Should]`
* **User Story:** As the finance team, I want the system to process daily biller settlement files and flag discrepancies, so that mismatched records are highlighted for investigation.
* **Prerequisite:** Daily settlement CSV file is ingested at end-of-day.
* **Result:** Matching `UNKNOWN` payments are updated to `COMPLETED`, unlisted `UNKNOWN` payments are set to `FAILED` $\rightarrow$ `REFUNDED`, and mismatches are flagged in a discrepancy report.
* **Acceptance Criteria:**
  * **Given** a nightly settlement file lists a payment as settled that PayLink recorded as `FAILED`, **When** reconciliation runs, **Then** the payment is flagged as a critical discrepancy in the daily reconciliation report.

---

### Category 7: Support

#### US-24: Payment Reference Lifecycle Lookup `[FR-20 | Priority: Must]`
* **User Story:** As a support employee, I want to search payments by unique reference ID and view their full state history, so that I can resolve customer inquiries.
* **Prerequisite:** Support employee is authenticated with `SUPPORT` role permissions.
* **Result:** System returns payment summary, state transition timeline with timestamps, biller response codes, and ledger history.
* **Acceptance Criteria:**
  * **Given** a customer reports issue with reference `REF-1002`, **When** support employee searches `REF-1002`, **Then** system displays full state timeline (`INITIATED` $\rightarrow$ `FUNDS_HELD` $\rightarrow$ `SENT_TO_BILLER` $\rightarrow$ `UNKNOWN`), retry logs, and held balance status.

#### US-25: Account Freeze & Unfreeze Controls `[FR-21 | Priority: Should]`
* **User Story:** As a support employee, I want to block and unblock user accounts with a mandatory reason note, so that compromised accounts can be frozen quickly.
* **Prerequisite:** Support employee provides target account ID and documented reason note.
* **Result:** User `status` updates to `BLOCKED` (rejecting new payments) or `ACTIVE`, and the action is logged in audit records.
* **Acceptance Criteria:**
  * **Given** support employee blocks account `USR-8821` with reason `"Suspicious login activity"`, **When** account attempts a bill payment, **Then** request is `REJECTED` at fraud check and block action is recorded in audit log.

---

## 5. Non-Functional Requirements & Evaluation Metrics

To ensure system reliability, financial accuracy, and modifiability under peak operating loads, PayLink defines 7 non-functional requirements.

| ID | Quality Attribute | Target Metric & Bound | Evaluation & Verification Method |
| :--- | :--- | :--- | :--- |
| **NFR-01** | **Correctness & Concurrency** | **0** double charges and **0** negative balances during concurrent execution and network retries. | Verified via automated multi-threaded integration tests (Hard Cases #1, #4, #6) enforcing pessimistic locking and unique idempotency constraints. |
| **NFR-02** | **Performance & Throughput** | $\ge 100$ payments/second sustained for **1 minute** against mock billers with p95 response time $< 500\text{ ms}$. | Measured via k6/JMeter load testing against Docker Compose deployment setup. |
| **NFR-03** | **Resilience & Isolation** | **100%** of payments to healthy billers succeed during single-biller outages; strict **10.0s** biller timeout limit. | Fault-injection integration testing simulating 12s delays and HTTP 500 responses (Hard Cases #2, #5). |
| **NFR-04** | **Modifiability** | **0 lines of code modified** outside `Biller Integration` module and config when adding billers or upgrading ElectroCo v2. | Architectural dependency checks and git diff analysis on Phase 2 pull requests. |
| **NFR-05** | **Auditability & Traceability** | **100%** of ledger movements and admin/support operations have corresponding audit records (`actor`, `timestamp`, `reason`). | Database constraint integrity checks and automated test assertions for 1:1 audit log parity. |
| **NFR-06** | **Monetary Precision** | **0.0000 EGP** rounding error across $10,000$ fractional fee/commission calculations. | Static analysis enforcing `DECIMAL(19,4)` column definitions and property-based unit tests. |
| **NFR-07** | **Testability** | $\ge 70\%$ domain logic line coverage measured automatically in CI workflow. | Enforced by GitHub Actions CI workflow using code coverage plugins (JaCoCo / Istanbul / Coverlet / PyTest-Cov). |