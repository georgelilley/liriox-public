# Liriox

A full-stack Web3 prize draw platform. Users enter recurring draws by paying USDC entry fees on Ethereum; winners are selected using Chainlink VRF for provably fair randomness, and payouts execute on-chain. The system supports cash-prize draws (immediate USDC transfer) and giveaway draws (escrow with two-party confirmation and dispute resolution). A postal free-entry pathway provides a non-purchase route into every draw.

The platform is closed-source and under active development. This document describes the architecture at a high level without exposing operational secrets.

---

## Architecture

Monorepo (pnpm workspaces) with four packages:

| Package | Role | Runtime |
|---|---|---|
| `liriox-frontend` | Next.js 16 app (App Router, React 19, RSC) | Vercel / Node |
| `liriox-backend` | Express 5 API server, cron schedulers, workers, CLI tooling | Node |
| `liriox-contracts` | Solidity smart contracts (Foundry / Forge) | EVM (Sepolia) |
| `liriox-shared` | Contract ABIs, public-code generation, policy constants | Imported by frontend + backend |

**Frontend.** Server Components handle all data fetching via direct Postgres queries (`server-only` enforced). Client components receive data as props; there is no client-side state library. Authentication is request-scoped and cached per render pass via React `cache()`. API routes exist only for operations that must originate from the browser (entry signing, balance checks, admin actions).

**Backend.** A thin HTTP layer (two Express route groups) serves internal endpoints. The bulk of the backend is five cron schedulers (round creation, settlement, fee withdrawal, notification dispatch, free-entry processing), a contract-deployment worker, and a set of CLI tools for operational tasks. Domain logic is isolated by business area: draws, fees, refunds, free entry.

**Contracts.** A `PrizeDrawFactory` deploys minimal proxies (EIP-1167 clones) of a single `PrizeDraw` implementation, keeping per-draw deployment gas minimal. A separate `VrfManager` contract isolates Chainlink VRF v2.5 integration. All entry authorisation uses EIP-712 typed-data signatures issued by the backend, so users never need to trust an approval flow. The contracts enforce payout-tier validation, reentrancy guards, and role-based access control on-chain.

**Infrastructure.** A 450-line Makefile orchestrates contract deployment, database seeding, ABI export, cron invocation, and emergency operations. GitHub Actions runs Forge build and test on push. Environment configuration is validated at startup with fail-fast semantics.

---

## Major Subsystems

**Draw lifecycle.** A scheduling cron creates rounds up to 30 days ahead with a minimum 14-day notice period. Each round transitions through: open, closed, VRF-requested, VRF-fulfilled, settlement-in-progress, settled. Settlement is batched (configurable ops-per-tx and max-txs-per-run) to stay within block gas limits, and every phase is idempotent so the cron can safely retry after partial failures.

**Payout tiers.** Two tier types — `PercentOfPool` (fixed winner count, percentage-based allocation) and `FixedAmountPerWinner` (variable winner count, fixed USDC amount, capped by `maxWinners`) — can be combined in a single draw. Each tier has an activation threshold (`minTotalPrize`) so lower tiers only fire when the pool is large enough. Allocation logic, including leftover redistribution, is enforced both in contract storage and validated off-chain before round creation.

**Giveaway escrow.** When a draw is configured as a giveaway, settlement creates per-winner escrow slots rather than immediate transfers. Both winner and creator must confirm fulfilment before funds release. Either party can raise a dispute; the platform owner resolves disputes and directs the funds.

**Free-entry pipeline.** A six-stage pipeline (ingest, extraction, validation, queueing, on-chain execution, confirmation) processes postal free entries. Stage 0 guarantees custody acknowledgement: the ingest endpoint returns 2xx only after durable storage and DB commit. Stage 2 gates on mandatory PII redaction before evidence is published. Stage 4 uses `FOR UPDATE SKIP LOCKED` for concurrency-safe batch processing. A global system control acts as an execution brake. The pipeline enforces a terminal-outcome invariant: every ingested entry reaches either confirmed-on-chain or terminal-rejection, and settlement is blocked if any entry in the valid window is non-terminal.

**Authentication.** Passwordless login via Magic SDK (email OTP). Magic issues a DID token stored client-side and sent as a cookie for server-side validation. Each Magic identity is automatically linked to an Ethereum wallet. The backend validates tokens independently via the Magic Admin SDK before processing authenticated requests.

**Notification dispatch.** A cron queries settled draws for unsent winner and payout notifications, marks each as sent only after successful dispatch, and logs per-draw errors without aborting the batch.

---

## Reliability and Compliance

**Settlement safety.** A global `settlement_allowed` system control acts as a kill switch. A separate guard blocks settlement for any draw that has unresolved free entries received within the past seven days. Both checks fail closed — if the control row is missing, settlement does not proceed. Settlement error codes (`settlement_execution_paused`, `free_entry_backlog_blocks_settlement`, `draw_free_entry_not_fully_resolved`) provide structured diagnostics.

**Idempotency.** All cron phases (round creation, settlement initiation, batch execution, finalisation) use status-based transitions and are safe to re-run. The deployment worker uses an atomic `pending` → `deploying` guard to prevent double-deploy.

**On-chain safety.** Contracts use OpenZeppelin's `ReentrancyGuard` on all state-mutating external calls that involve token transfers. Entry authorisation via EIP-712 signatures with per-user nonces prevents replay. The implementation contract is self-locked on construction (`_initialized = true`, ownership burned) so it cannot be used directly. Typed custom errors (35+ distinct errors) replace `require` strings for gas efficiency and debuggability.

**Validation.** Input validation is enforced at two layers: Zod schemas in the backend HTTP layer, and on-chain `_validatePayoutTiers()` in the contract. Payout tier rules (contiguous indices, non-decreasing thresholds, allocation caps, ordering constraints) are checked identically in both places.

**Audit trail.** The free-entry pipeline maintains a full chain of custody from email receipt to on-chain confirmation across six database tables. Evidence storage paths are write-once. Redacted and unredacted paths are tracked separately.

---

## What Makes the Implementation Non-Trivial

- **Multi-step on-chain settlement** that must handle arbitrarily large entry sets within block gas limits, resume after partial completion, and coordinate off-chain state (DB) with on-chain events — including a recovery path that re-indexes events directly from the chain if receipt-based indexing fails.
- **Dual payout models** with activation thresholds, leftover redistribution, and a variable-winner tier type whose count is only known at settlement time.
- **EIP-712 signature-authorised entries** where the backend acts as a trusted signer, requiring nonce management, expiry enforcement, and domain-separator correctness across contract upgrades (clone re-deployments).
- **Factory/clone deployment** orchestrated from both a Makefile (manual ops) and a programmatic worker (user-initiated draw group creation), with foreign-key tracking across factory, implementation, and VRF manager deployment records.
- **A six-stage free-entry pipeline** with custody guarantees, PII redaction gating, concurrency-safe batch execution (`SKIP LOCKED`), block-confirmation thresholds, and a settlement-blocking invariant that prevents premature draw resolution.
- **Escrow with dispute resolution** — a state machine (confirmed / disputed / resolved / released) that handles the coordination problem between an anonymous winner and an external creator, with platform arbitration as a fallback.

---

## Stack

**Frontend:** Next.js 16, React 19, Server Components, Server Actions, Tailwind CSS, Radix UI, Framer Motion, Vitest, Testing Library

**Backend:** Express 5, Node.js, cron schedulers, Mailgun, Sharp, Multer, dayjs

**Blockchain:** Solidity 0.8.20+, OpenZeppelin (Clones, Ownable, ReentrancyGuard, EIP-712), Chainlink VRF v2.5, Foundry (Forge, forge-script), ethers.js 6, viem 2

**Data:** PostgreSQL (Supabase), raw SQL via `postgres` driver, Supabase client for admin operations

**Auth:** Magic SDK (email OTP, DID tokens, automatic wallet provisioning)

**Tooling:** pnpm workspaces, TypeScript (strict), Zod, ESLint, Prettier, GitHub Actions CI, Make
