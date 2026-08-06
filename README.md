# PayFence

## Policy-Controlled Payments for Autonomous AI Agents

PayFence is a payment-control layer for AI agents that need to interact with paid APIs and services autonomously.

The system allows an AI agent to propose a payment, while a deterministic policy layer decides whether that payment is permitted. Approved payments can then be executed through **x402** and **ERC-7710**, with settlement evidence, transaction hashes, relay fees, and remaining budget recorded for auditability.

> **AI decides when to spend.  
> PayFence decides whether it may spend.  
> x402 + ERC-7710 proves how it spent.**

---

## Overview

AI agents are increasingly capable of performing tasks that require paid APIs, premium models, proprietary data, and execution services. Giving an autonomous agent unrestricted access to a wallet creates a major financial-risk problem.

PayFence introduces a controlled payment boundary between the agent's spending intent and the final payment settlement.

The system provides:

- Scoped payment permissions
- Budget enforcement
- Endpoint and method restrictions
- Token and network restrictions
- Merchant / `payTo` allowlisting
- Agent spending decisions
- Pre-payment policy enforcement
- x402 payment discovery
- ERC-7710 delegated payment execution
- 1Shot relay and settlement support
- Transaction and payload evidence
- Payment ledger and budget tracking
- On-chain settlement verification
- Fail-closed payment behavior
- Blocked-payment evidence
- Organization and employee budget management
- Persistent SQLite-backed demo data

---

# Core Concept

The payment flow separates **agent intent** from **payment authority**.

```text
                AI Agent
                   |
                   | Spend Intent
                   v
          +---------------------+
          |     PayFence        |
          |   Policy Engine     |
          +---------------------+
             |             |
          ALLOW           BLOCK
             |             |
             v             v
          x402          No Payment
             |
             v
         ERC-7710
             |
             v
      Settlement / Relay
             |
             v
       AI Service / API
             |
             v
     Ledger + Evidence
```

The agent can recommend a payment, but it does not automatically receive unrestricted control of the user's wallet.

---

# Problem

Autonomous agents may need to purchase:

- Premium AI model calls
- Risk reports
- Proprietary datasets
- Simulation services
- Execution APIs
- SaaS tools
- Other x402-enabled services

Traditional approaches introduce several risks:

### Direct private-key access

Giving an agent a private key can give it direct control over funds.

### Unlimited token approvals

An unlimited allowance can allow an agent to spend beyond the intended budget.

### Frontend-only budgets

A UI budget indicator is not a real payment constraint if the payment rail itself remains unrestricted.

### Post-payment monitoring

Detecting an unsafe payment after settlement is too late. Financial policies should be checked before the payment is submitted.

### Unclear payment attribution

A centralized payment layer can make it difficult to determine which exact request caused a debit.

PayFence addresses these problems by enforcing payment policies before an x402 payment is submitted.

---

# Key Functionalities

## 1. Scoped Wallet Permissions

Users can connect MetaMask and approve a scoped ERC-20 periodic Advanced Permission.

Example:

```text
Network:       Base Sepolia
Asset:         USDC
Budget:        1.00 USDC / 24h
Scope:         Risk-brief API
Expiry:        Configured expiration
```

The system is designed around delegated and bounded authority rather than unrestricted wallet access.

---

## 2. AI Spending Decision

Before a paid request is submitted, the agent produces a spending decision containing information such as:

```text
decision
reason
estimatedCostAtomic
confidence
budgetBefore
projectedBudgetAfter
```

Example:

```text
Decision: spend
Reason: The requested risk analysis requires the paid risk-brief API.
Estimated cost: 0.01 USDC
Confidence: 0.94
```

The AI decision is not sufficient by itself. The deterministic policy layer must also approve the request.

---

## 3. Policy Guard

The policy engine validates the payment request before any paid x402 request is submitted.

It checks:

- Budget
- Endpoint
- HTTP method
- Token
- Network
- `payTo`
- Permission status
- Permission expiry
- Previous ledger spending
- Agent decision
- Payment amount
- Payload freshness
- Delegation caveats

Unsafe requests are rejected before payment.

```text
Agent Request
      |
      v
Policy Validation
      |
  +---+---+
  |       |
ALLOW   BLOCK
  |       |
  v       v
x402    No Payment
```

---

## 4. x402 Payment Flow

The protected service acts as an x402 seller.

The client first makes an unpaid request.

The seller responds with:

```text
HTTP 402 Payment Required
```

The payment requirement specifies details such as:

```text
scheme=exact
network=eip155:84532
asset=Base Sepolia USDC
assetTransferMethod=erc7710
```

The client then constructs the required payment payload.

The AI service only executes the paid task after payment verification and settlement succeed.

---

## 5. ERC-7710 Delegated Payments

ERC-7710 is used to execute delegated payment authority through the x402 payment flow.

For each paid call, the system can construct a fresh child delegation containing payment-specific caveats.

Examples include:

```text
limitedCalls
valueLte
allowedTargets
allowedMethods
timestamp
erc20TransferAmount
```

The server validates these constraints before settlement.

If a caveat is missing or does not match the payment requirement, the request fails closed.

---

## 6. Multiple Bounded Payments

A single scoped permission can support multiple independent paid API calls without requiring a new broad wallet authorization for every request.

Each payment can still have its own:

- Child delegation
- Payload context hash
- Payment amount
- Transaction hash
- Ledger entry
- Settlement evidence

This provides a balance between autonomous operation and spending control.

---

## 7. Budget Management

PayFence tracks the user's configured spending limit and remaining budget.

Example:

```text
Initial budget       1.00 USDC
Call #1              0.01 USDC
Call #2              0.01 USDC
Call #3              0.01 USDC
Remaining            0.97 USDC
```

The service price is counted against the agent's policy budget.

Relay infrastructure fees are tracked separately.

---

## 8. Overspending Protection

Requests that exceed the permitted amount are blocked before payment.

Example:

```text
Available budget:     0.20 USDC
Requested payment:   0.50 USDC

Result: BLOCKED
Transaction: None
Wallet debit: None
```

The blocked request can still be recorded in the ledger as evidence.

---

## 9. Payment Ledger

Every relevant payment attempt can be recorded in the ledger.

Ledger information includes:

- Paid / blocked / revoked status
- Service price
- Relay fee
- Total wallet debit
- Transaction hash
- Payload context hash
- Remaining budget
- Request information
- Policy decision

The ledger distinguishes between:

```text
Service Price
      +
Relay Fee
      =
Total Wallet Debit
```

This makes API spending and settlement infrastructure costs independently visible.

---

## 10. Chain Evidence Verification

The system includes a chain-evidence verification flow for validating that a settlement actually reached the expected on-chain execution path.

Verification can inspect:

- Base Sepolia transaction receipts
- Transaction status
- DelegationManager target
- `redeemDelegations(...)` function call
- Function selector
- USDC service transfer
- Relay fee transfer

This provides protocol-level evidence rather than relying only on frontend status indicators.

---

## 11. Fail-Closed Security

The payment flow is designed to fail closed.

Unsafe conditions do not result in a successful payment.

Examples:

```text
Oversized amount       → BLOCK
Invalid endpoint       → BLOCK
Invalid token          → BLOCK
Wrong network          → BLOCK
Expired permission     → BLOCK
Stale payload          → BLOCK
Duplicate payload      → BLOCK
Missing ERC-7710 data  → BLOCK
Denied agent decision  → BLOCK
```

A successful settlement is only recorded after the settlement process confirms success.

---

## 12. Evidence Dashboard

The dashboard exposes payment evidence rather than hiding the payment process behind a simple success message.

It can display:

- Protected x402 resource
- Selected payment requirement
- Payment scheme
- Transfer method
- Token
- Amount
- Network
- `payTo`
- Advanced Permission summary
- Child delegation target
- Delegation caveats
- ERC-7710 payload context hash
- 1Shot task information
- Relay fee
- Base Sepolia transaction hash
- Service price
- Total wallet debit
- Remaining budget
- Blocked request evidence

---

# Organization Management

The system also supports an organization-oriented spending model.

Organizations can register and manage employees who operate AI agents with controlled budgets.

A possible organizational hierarchy is:

```text
Organization
     |
     +-- Employee / Agent
              |
              +-- Spending Policy
              |
              +-- Budget
              |
              +-- Allowed Services
              |
              +-- Payment Ledger
```

This allows the payment-control model to extend beyond individual users toward team and enterprise scenarios.

---

# Technical Architecture

```text
+-------------------------+
|      User + MetaMask    |
+------------+------------+
             |
             | Scoped Permission
             v
+-------------------------+
|       AI Agent          |
|     Spend Intent        |
+------------+------------+
             |
             v
+-------------------------+
|     Policy Engine       |
|-------------------------|
| Budget                  |
| Endpoint                |
| Method                  |
| Token                   |
| Network                 |
| payTo                   |
| Permission              |
| Agent Decision          |
+------------+------------+
             |
       ALLOW |
             v
+-------------------------+
|    x402 Seller Layer    |
|   HTTP 402 Challenge    |
+------------+------------+
             |
             v
+-------------------------+
|    ERC-7710 Builder     |
| Child Delegation        |
| Payment Caveats         |
+------------+------------+
             |
             v
+-------------------------+
| Server Verification     |
| + Preflight Validation  |
+------------+------------+
             |
             v
+-------------------------+
| 1Shot / Settlement      |
| Relay + Execution       |
+------------+------------+
             |
             v
+-------------------------+
|      AI Provider        |
|   Paid Task Execution   |
+------------+------------+
             |
             v
+-------------------------+
| Dashboard + Ledger      |
| Evidence + Budget       |
+-------------------------+
```

---

# Component Responsibilities

| Component | Responsibility |
|---|---|
| MetaMask Smart Accounts Kit | Permission acquisition and delegation environment |
| MetaMask Advanced Permissions | Parent authorization boundary for scoped spending |
| AI Agent | Generates spending intent and performs the requested task |
| Policy Engine | Enforces hard spending and payment constraints |
| x402 | HTTP payment discovery and paid request protocol |
| ERC-7710 | Delegated payment execution within the x402 flow |
| 1Shot | Settlement infrastructure and relay accounting |
| AI Provider | Executes the paid AI task after successful settlement |
| Ledger | Records payment and policy events |
| Chain Evidence Verifier | Validates settlement evidence on Base Sepolia |
| Dashboard | Presents payment, policy, budget, and settlement evidence |
| Organization Layer | Manages organizations, employees, and agent spending boundaries |

---

# Payment Flow

A complete successful payment follows this sequence:

```text
1. User connects MetaMask
          ↓
2. User approves scoped permission
          ↓
3. Agent generates spend intent
          ↓
4. Policy engine validates request
          ↓
5. Client requests x402 challenge
          ↓
6. Seller returns HTTP 402
          ↓
7. Client builds ERC-7710 payment payload
          ↓
8. Server performs preflight validation
          ↓
9. Settlement executes through ERC-7710
          ↓
10. Relay infrastructure processes transaction
          ↓
11. Base Sepolia transaction is produced
          ↓
12. AI provider executes paid task
          ↓
13. Ledger records payment
          ↓
14. Dashboard displays evidence
```

---

# Blocked Payment Flow

Unsafe requests follow a different path:

```text
Agent Request
     ↓
Policy Engine
     ↓
Constraint Violation
     ↓
BLOCK
     ↓
No Paid x402 Header
     ↓
No Settlement
     ↓
No Transaction
     ↓
Ledger Evidence
```

This makes the policy layer a **pre-payment security boundary**, rather than a monitoring system that reacts after money has already moved.

---

# On-Chain Configuration

The current payment configuration uses:

| Parameter | Value |
|---|---|
| Network | Base Sepolia |
| Chain ID | `84532` |
| Payment Asset | Base Sepolia USDC |
| Demo Service Price | `0.01 USDC` |
| Demo Budget | `1.00 USDC / 24h` |
| x402 Scheme | `exact` |
| Transfer Method | `erc7710` |
| ERC-7710 Execution | MetaMask DelegationManager |
| Settlement Infrastructure | 1Shot |

---

# Data and Persistence

The application uses SQLite for demo data and policy-related state.

The database stores information such as:

### Ledger entries

Records paid, blocked, and revoked calls.

### Permissions

Stores the current scoped permission and related authorization information.

### Training examples

Contains labeled policy-decision samples covering scenarios such as:

- In-budget requests
- Over-budget requests
- Exhausted budgets
- Non-allowlisted endpoints
- Non-allowlisted tokens
- Burst-frequency violations
- Expired windows
- Agent skip decisions
- Revoked grants

The application can seed representative data so that the dashboard, ledger, and evidence views can be demonstrated without requiring every interaction to be performed manually.

---

# Policy Decision Examples

## Allowed

```text
Budget:             1.00 USDC
Requested amount:   0.01 USDC
Endpoint:            Allowed
Token:               Allowed
Network:             Base Sepolia
Permission:          Active

Result: ALLOW
```

## Blocked — Budget

```text
Budget remaining:   0.20 USDC
Requested amount:   0.50 USDC

Result: BLOCK
```

## Blocked — Endpoint

```text
Endpoint:
https://unapproved-service.example

Policy:
Endpoint not allowlisted

Result: BLOCK
```

## Blocked — Expired Permission

```text
Permission:
Expired

Result: BLOCK
```

---

# Security Model

The architecture follows several principles.

### Delegated

The agent operates through delegated authority rather than unrestricted access to the user's primary wallet authority.

### Scoped

Permissions can be constrained by:

- Amount
- Time
- Endpoint
- Method
- Token
- Network
- Target
- Call count

### Observable

Payment activity produces evidence that can be inspected through the dashboard and ledger.

### Revocable

The permission model supports revocation and synchronization with the wallet's current authorization state.

### Pre-Checked

The policy layer evaluates requests before the payment is submitted.

### Fail-Closed

If required validation fails, the system does not proceed with settlement.

---

# Technology Stack

## Frontend

- React-based dashboard
- Component-based UI
- Wallet connection interface
- Payment evidence views
- Budget and ledger visualization

## Blockchain / Wallet

- MetaMask
- MetaMask Smart Accounts Kit
- MetaMask Advanced Permissions
- ERC-7710
- DelegationManager
- Base Sepolia
- USDC

## Payment Protocol

- x402
- ERC-7710 payment payloads
- Exact payment scheme

## Settlement

- 1Shot settlement / relayer infrastructure
- Transaction status tracking
- Relay fee accounting

## Backend

- Server-side policy validation
- x402 seller endpoint
- ERC-7710 verification
- Chain evidence verification
- SQLite persistence

## AI

- AI agent spending decision layer
- Configurable downstream AI provider
- Risk-brief task workflow

---

# Application Pages

The application is organized into dedicated pages rather than placing every feature on a single screen.

### Home

Provides the product overview and explains the purpose of the payment-control layer.

### Demo

Demonstrates the complete end-to-end flow:

```text
Wallet
→ Permission
→ Agent Decision
→ Policy Check
→ x402
→ ERC-7710
→ Settlement
→ AI Result
```

### Evidence

Displays the evidence generated during the payment flow, including permissions, payment requirements, delegation information, payload hashes, and settlement information.

### Ledger

Shows paid and blocked requests, service costs, relay fees, transaction hashes, and remaining budget.

### Chain Evidence

Allows settlement records to be verified against Base Sepolia on-chain evidence.

### About

Explains the problem, architecture, use cases, and overall approach.

### Organization

Provides organization registration and employee management for controlled agent budgets.

---

# Example Use Cases

## AI Research Agent

An AI research agent can purchase premium datasets while being limited to a predefined budget.

## Financial Risk Agent

An agent can purchase risk reports but only from approved endpoints and within a daily spending limit.

## Data Agent

An autonomous data-retrieval agent can pay for APIs without receiving unrestricted wallet access.

## Developer Agent

A coding agent can purchase specialized APIs or services while enforcing per-tool spending policies.

## Enterprise Agents

Organizations can assign employees and their agents controlled budgets and service permissions.

---

# Current Capabilities

The system currently supports the following demonstrated capabilities:

- Real MetaMask wallet detection
- Account reading
- Base Sepolia network enforcement
- Scoped Advanced Permission semantics
- x402 seller endpoint
- ERC-7710 x402 client flow
- Generated child delegations
- Server-side ERC-7710 verification
- Requirement and grant matching
- Delegator and DelegationManager validation
- Caveat validation
- Payload freshness validation
- Base Sepolia settlement evidence
- 1Shot-supported settlement path
- AI spending decision layer
- Over-budget blocking
- Ledger evidence
- Dashboard evidence
- Remaining-budget tracking
- Chain evidence verification
- SQLite persistence
- Organization and employee management flow

---

# Demonstration Flow

A typical demonstration can follow these steps:

1. Open the application dashboard.
2. Connect MetaMask.
3. Confirm Base Sepolia.
4. Approve a scoped USDC permission.
5. Start the AI agent.
6. Inspect the generated spending decision.
7. Observe the policy checks.
8. Submit an unpaid request to the x402 seller.
9. Receive the `402 Payment Required` challenge.
10. Build the ERC-7710 payment payload.
11. Inspect the delegation and caveats.
12. Run settlement.
13. Verify the resulting transaction.
14. Receive the paid AI result.
15. Inspect the ledger entry.
16. Check the remaining budget.
17. Run an oversized request.
18. Observe that the request is blocked before payment.

---

# Future Expansion

The architecture can be extended into a general payment-control platform for autonomous agents.

Potential extensions include:

- Real privacy-preserving AI provider integration
- Agent tool marketplaces
- x402-priced agent tools
- Production database infrastructure
- Signed policy snapshots
- On-chain policy registries
- Multi-agent budgets
- Manager-to-worker budget delegation
- Dynamic risk scoring
- Per-endpoint spending limits
- Recurring spending windows
- Emergency global payment pause
- Expanded payment support for data APIs
- Model inference payments
- Simulation services
- Relayer services
- Autonomous SaaS subscriptions
- Reusable payment-policy middleware for developers

---

# Vision

As AI agents become capable of participating in economic activity, autonomous payment systems need more than a wallet connection.

They need:

```text
Delegated Authority
        +
Scoped Permissions
        +
Pre-Payment Policy
        +
Transparent Evidence
        +
Revocable Access
```

PayFence provides this control boundary while preserving the autonomous experience of x402-powered services.

The goal is simple:

> **Let AI agents act autonomously without giving them unlimited financial authority.**
