# PayLink — Digital Wallet & Bill Payment Backend (Phase 1)

**PayLink** is a high-reliability digital wallet and payment processing backend built to modernize utility bill payments in Egypt (operating similarly to Fawry and Vodafone Cash). It enables millions of consumers to settle monthly utility bills—including electricity, water, mobile top-ups, and internet subscriptions—seamlessly through a mobile application or at physical retail agent terminals.

---

## 📖 Overview & Core Mission

By providing a unified digital channel, PayLink eliminates long physical queues and repetitive manual processes for consumers while ensuring utility companies (**billers**) receive rapid, accurate settlements.

### Business Model & Fiduciary Responsibility
PayLink operates on a **fee-and-commission model**, charging a nominal service fee on customer transactions and distributing a percentage as commission to retail agents. Because PayLink holds customer funds in digital wallets, it adheres to strict technical and fiduciary standards:
* **Zero Balance Drift:** Wallet balances are strictly prohibited from dropping below zero.
* **No Double-Charging:** Customers are never overcharged or billed twice for the same intent.
* **Strict Authorization:** No unauthorized transactions occur without explicit customer notice.
* **Atomic & Resilient:** Every movement of funds is ACID-compliant, immutably audit-logged, and resilient against network failures, timeouts, and concurrent requests.

---

## 👥 Key System Stakeholders

| Stakeholder Role | Description & Primary Interest | Key System Interactions |
| :--- | :--- | :--- |
| **Customer** | End-user paying utility bills via the mobile app. Expects fast, accurate payments, transparent fees, and instant confirmation. | Account registration, KYC tier upgrades, wallet top-ups, bill inquiries, bill payments, and transaction history viewing. |
| **Agent** | Retail shop owner operating a PayLink POS terminal. Serves cash-paying walk-in customers to earn commissions. | Cash-in wallet top-ups, paying bills on behalf of walk-in customers, and accumulating commission balances. |
| **Biller (Utility Provider)** | Utility enterprises (`ElectroCo`, `AquaCo`, `TopUpNet`, `NetCo`) receiving payments. Expects accurate settlements and zero balance drift. | Responding to real-time inquiries/payments, issuing asynchronous callbacks, and receiving settlement files. |
| **Support Employee** | Internal operations team resolving disputes, investigating stuck transactions, and handling fraud alerts. | Searching payment histories via Reference ID, reviewing transaction state timelines, and freezing/unfreezing accounts. |
| **Administrator** | System administrator managing platform parameters, pricing strategies, and provider connectivity. | Editing fee rules, configuring promotional zero-fee windows, updating daily tier limits, and toggling active billers. |
| **Auditor / Regulator** | Financial compliance bodies (e.g., Central Bank of Egypt) requiring strict data integrity, non-repudiation, and financial safety. | Inspecting append-only immutable ledger records and system audit logs. |

---

## 🏗️ Architecture & Design Decisions Log

During requirements elicitation and edge-case analysis, several domain ambiguities were resolved to guarantee financial safety in Phase 1:

| ID | Issue / Ambiguity | Context / Question | Architectural Decision & Resolution |
| :--- | :--- | :--- | :--- |
| **DEC-01** | **Pending Biller Timeouts** | What happens to held customer funds when a live biller API call exceeds $10\text{s}$? | Funds remain reserved in `held_balance` and the payment transitions to `UNKNOWN`. Funds are **never** refunded automatically on timeout; scheduled retries ($+1\text{ min}$, $+5\text{ min}$, $+15\text{ min}$) or nightly settlement reconciliation must confirm biller non-receipt before issuing a refund. |
| **DEC-02** | **Concurrent Wallet Payments** | How do we handle two payments arriving at the exact same millisecond when funds cover only one? | Enforce database row-level pessimistic locking (`SELECT ... FOR UPDATE`) on the `Wallet` entity inside a single ACID transaction. One payment succeeds (`FUNDS_HELD`), while the second fails validation (`REJECTED`) with zero negative balance risk. |
| **DEC-03** | **Network Retries & Duplicates** | How do we prevent double charges when a mobile client retries a slow `POST` payment request? | Mandate a unique client-generated `Idempotency-Key` header. The system checks an `IdempotencyRecord` table prior to execution; duplicate keys return the cached original response without creating new ledger entries or re-calling billers. |
| **DEC-04** | **Floating-Point Rounding Drift** | Standard `float`/`double` types cause rounding errors in percentage fees ($1.5\%$) and revenue splits ($40\%$). | Enforce exact decimal types (`DECIMAL(19,4)` in SQL; `BigDecimal` or equivalent exact decimal structures in application code) across all entities, APIs, and ledger calculations. |
| **DEC-05** | **Agent Fee & Commission Split** | Does an agent pay the customer fee, and how is commission credited? | The customer pays `bill_amount + service_fee`. Upon `COMPLETED` status, $40\%$ of the `service_fee` is credited to the agent's wallet via a distinct `AGENT_COMMISSION` ledger entry, while $60\%$ is retained as PayLink net revenue. |
| **DEC-06** | **Notification Service Outages** | If the SMS/Push gateway fails during payment completion, should the payment roll back? | Core payment processing is strictly decoupled from notifications. The payment completes normally, while notification delivery is enqueued asynchronously and retried up to $3$ times before being marked `UNDELIVERED`. |
