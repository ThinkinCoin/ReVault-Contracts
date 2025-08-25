# Recovery Vault (Harmony)

Fixed-redemption vault for **pre-hack wallets** on Harmony. Whitelisted users can redeem **supported depegged tokens** at a **fixed 1:1 USD value** and receive **USDC** (when available) or **wONE** (priced via on-chain oracle). The system enforces **per-wallet daily limits**, **round delays**, and a **dynamic fee** to protect liquidity and ensure fairness.

> This proposal **complements** HIP-30v2. It provides a simple, predictable, and ongoing path for small holders, without mass upfront distributions and without depending on a full re-peg.

---

## Table of Contents

* [Key Features](#key-features)
* [Architecture](#architecture)
* [Smart Contract](#smart-contract)

  * [Core Concepts](#core-concepts)
  * [Public Interfaces (high level)](#public-interfaces-high-level)
  * [Events & Transparency](#events--transparency)
  * [Security](#security)
  * [Units & Math](#units--math)
* [Frontend (React + Vite + Reown AppKit + ethers v6)](#frontend-react--vite--reown-appkit--ethers-v6)

  * [Directory Layout](#directory-layout)
  * [Environment Variables](#environment-variables)
  * [Local Run](#local-run)
* [Off-Chain Components](#off-chain-components)
* [Development (Hardhat)](#development-hardhat)

  * [Setup](#setup)
  * [Scripts](#scripts)
  * [Testing](#testing)
  * [Coverage Targets](#coverage-targets)
* [Governance](#governance)
* [Roadmap](#roadmap)
* [Contributing](#contributing)
* [License](#license)

---

## Key Features

* **Fixed 1:1 USD redemption** for supported depegged tokens.
* **Payout in USDC or wONE** (wONE amount uses trusted on-chain oracle).
* **Per-wallet daily limit**, reset every 24h.
* **Rounds with mandatory delay (24h)** between funding and redemptions.
* **Dynamic fee (tiered)** to incentivize steady participation and sustainability.
* **Whitelist via Merkle Proof** (gas-efficient eligibility validation).
* **Pause & emergency withdraw** (unused funds return to the RMC multisig).
* **Full transparency** through on-chain events for auditability.

---

## Architecture

```
contracts/
  RecoveryVault.sol        # core vault logic (Harmony)
  abis/RecoveryVault.abi.json

frontend/                  # React + Vite UI, Reown AppKit, ethers v6
  src/
    components/            # UI building blocks
    contexts/              # Contract context (ethers v6)
    services/              # vaultService, tokenService, appkit adapter
    store/                 # Redux Toolkit slices (if used)
    styles/                # Global.module.css, etc.

scripts/                   # deployment & maintenance scripts
test/                      # unit & integration tests (Hardhat)
docs/                      # specs, forum post, diagrams (optional)
```

---

## Smart Contract

### Core Concepts

* **Network**: Harmony (Shard 0).
* **Language/Libraries**: Solidity ≤ 0.8.18, OpenZeppelin 4.9.x (Ownable, ReentrancyGuard, Pausable).
* **Rounds**: New liquidity → **24h delay** → redemptions enabled. When allocation is exhausted, new funding + new delay.
* **Per-wallet daily limit**: USD-denominated cap; resets every 24h.
* **Dynamic fee**: 4 tiers (e.g., 1.00% base down to 0.25% at higher usage).
* **Whitelist**: Merkle root controls **which pre-hack wallets** can redeem.
* **Outputs**: USDC (preferred, if available) or wONE using **oracle price**.
* **Normalization**: **All user-facing amounts in USD “integer units”** (no decimals) on read/logic paths. Internal conversions handle token decimals and oracle rates.

> **Note on supported tokens:** the vault accepts a curated set of depegged tokens. Governance can update this list (and any fixed USD semantics when applicable).

### Public Interfaces (high level)

> Names and signatures may differ slightly in your final `RecoveryVault.sol`. The list below reflects the **intended behaviors** and **call flows**.

* **Whitelist & Parameters**

  * `setWhitelistRoot(bytes32 newRoot)` — updates Merkle root.
  * `setDailyLimitUsd(uint256 newLimitUsd)` — updates per-wallet daily cap (USD integer).
  * `setFeeThresholds(uint256[] memory thresholdsUsd, uint16[] memory feeBps)` — updates tier thresholds and fees.
  * `pause()` / `unpause()` — emergency control.
  * `startNewRound()` — starts a new round after funding; enforces **24h delay** window.

* **Read Helpers**

  * `getUserLimit(address user)` → returns remaining daily limit (USD integer).
  * `isLocked()` → true if within the 24h delay window.
  * `quoteRedeem(address tokenIn, uint256 amountUsd, bool preferUSDC)`
    → returns quoted out amount and payout token (USDC/wONE), using oracle for wONE.

* **Action**

  * `redeem(address tokenIn, uint256 amountUsd, bool preferUSDC, bytes32[] proof)`
    → validates whitelist via **Merkle Proof**, enforces **daily limit**, **round state**, **fees**, ensures **vault has liquidity** and processes payout. Emits events.

> **All external calls** are protected by **ReentrancyGuard** and internal checks. The vault **rejects** invalid proofs, insufficient liquidity, or calls outside allowed windows.

### Events & Transparency

Contracts emit events for all critical actions, for example:

* `NewRoundStarted(uint256 startTime /*…*/)`
* `RedeemProcessed(address user, address tokenIn, uint256 amountUsd, bool paidUSDC /*…*/)`
* `BurnToken(address tokenIn, uint256 amount /* when applicable */)`
* `VaultPaused()` / `VaultUnpaused()`
* Parameter changes: `DailyLimitUpdated`, `FeeThresholdsUpdated`, `WhitelistRootUpdated`, etc.

Indexers and dashboards can track these events to produce **monthly on-chain reports** of used vs. unused funds.

### Security

* **Pausable**: Governance can pause redemptions on anomalies.
* **ReentrancyGuard**: Protects state from reentrancy attacks.
* **Strict limits**: Daily caps and round delays guard liquidity.
* **Merkle whitelist**: Only authorized pre-hack wallets can redeem.
* **Oracle**: Trusted on-chain price for fair wONE payouts.
* **Funds isolation**: Emergency withdraw returns **unused** funds to the RMC multisig.

> **Upgradability**: If the current deployment is **non-upgradeable**, future improvements require a new deployment and migration path approved by governance.

### Units & Math

* **USD normalization**: The vault uses **USD integer amounts** (`uint256`) for user limits, thresholds, and fee math.
* **Fees**: Typically handled as **basis points (bps)**.
* **Token decimals**: Internally normalized per token on transfers and accounting.
* **Oracle conversions**: USD ↔ ONE when paying out in wONE.

---

## Frontend (React + Vite + Reown AppKit + ethers v6)

A minimal UI allows users to connect a wallet, see daily limits, get quotes, and redeem.

### Directory Layout

```
frontend/src/
  components/
    wallet/WalletConnection.jsx
    recovery/RecoveryRedeemPanel.jsx
    shared/{AmountInput,TokenSelect,InfoRow,Alert}.jsx
  contexts/ContractContext.jsx
  services/{vaultService,tokenService,appkit}.js
  pages/Recovery.jsx
  styles/Global.module.css
  App.jsx
  main.jsx
```

> **Code style**: all UI texts and logs **must be in English**. Use React hooks purely (`useMemo`, `useCallback`) to avoid re-renders.

### Environment Variables

Create `frontend/.env`:

```bash
VITE_PROJECT_NAME="Recovery Vault"
VITE_REOWN_PROJECT_ID=<your_reown_project_id>
VITE_RPC_URL=https://api.harmony.one
VITE_CHAIN_ID=1666600000
VITE_VAULT_ADDRESS=0xYourVault
VITE_ORACLE_ADDRESS=0xOracle  # if used by UI
```

### Local Run

```bash
cd frontend
yarn
yarn dev
# open the Vite URL shown in the console
```

---

## Off-Chain Components

* **Merkle Proof API**
  Endpoint example: `GET /api/proof?address=0x...` → `{ "proof": ["0x...", ...] }`

  * Generated off-chain from the pre-hack wallet dataset.
  * Rotated when the whitelist changes (update merkle root on-chain).

* **Reporting**

  * Index events (`NewRoundStarted`, `RedeemProcessed`, etc.) and publish **monthly on-chain reports**.

---

## Development (Hardhat)

### Setup

```bash
yarn
yarn add -D hardhat @nomicfoundation/hardhat-toolbox chai mocha ethers
# OpenZeppelin for contracts
yarn add @openzeppelin/contracts
```

Create `.env` in root (Hardhat):

```bash
HARMONY_RPC_URL=https://api.harmony.one
DEPLOYER_PRIVATE_KEY=0xabc...   # test key only
```

`hardhat.config.js` (snippet):

```js
require("@nomicfoundation/hardhat-toolbox");

module.exports = {
  solidity: { version: "0.8.18", settings: { optimizer: { enabled: true, runs: 200 } } },
  networks: {
    harmony: { url: process.env.HARMONY_RPC_URL, accounts: [process.env.DEPLOYER_PRIVATE_KEY] }
  }
};
```

### Scripts

* `yarn compile` — compile contracts
* `yarn test` — unit tests
* `yarn coverage` — coverage (if configured)
* `yarn deploy:harmony` — deployment script (add your script under `scripts/`)

### Testing

* **Mocks**: Use `ERC20Mock`, `MockRouterV2` (if needed), and Merkle helpers.
* **What to test**

  * Whitelist: valid/invalid proofs.
  * Round logic: 24h delay, no redemptions before activation, liquidity exhaustion.
  * Daily limit: enforcement, reset after 24h.
  * Fee tiers: correct bps per threshold; boundary conditions.
  * Quote vs. redeem: USDC vs. wONE path; oracle pricing.
  * Pause/unpause: rejections while paused.
  * Events: emitted with expected fields.
  * Admin functions: only owner/governance can mutate parameters.

### Coverage Targets

* **≥ 95%** for vault core logic (limits, fees, rounds, whitelist).
* **100%** for failure paths of `redeem` (invalid proof, paused, limit exceeded, no liquidity).

---

## Governance

* **Owner**: RMC multisig (recommended).
* **Responsibilities**

  * Fund the vault in USDC and start rounds.
  * Update whitelist root, daily limits, fee tiers, and supported tokens.
  * Pause/unpause and emergency withdraw (returns **unused** funds to the multisig).
  * Publish monthly on-chain reports and forum updates.

> All parameter changes should be publicly announced and, where applicable, ratified via community governance.

---

## Roadmap

* **v1 (MVP)**

  * Fixed 1:1 USD redemption, daily limit, rounds with 24h delay, fee tiers, whitelist, pause/emergency, events.
  * Basic UI with Reown AppKit + ethers v6.

* **v1.1**

  * Admin dashboard (front-end) for monitoring limits, liquidity, and events.
  * Automated monthly report generator from event data.

* **v1.2**

  * Expand supported tokens list and add granular analytics.
  * Community dashboards and public APIs for metrics.

---

## Contributing

1. Open an issue describing the change/bug.
2. Create a feature branch: `feat/...` or `fix/...`.
3. Add tests and keep logs/messages in **English**.
4. Run `yarn lint && yarn test`.
5. Open a PR with context and screenshots (for UI).

**Code style**: ESLint + Prettier; React hooks purity; Ethers v6; consistent error handling with `try/catch` and `console.error`.

---

## License

Specify your license in `LICENSE` (e.g., MIT). If omitted, contributions are assumed under the repository’s chosen license.

---

### TL;DR

* Whitelisted pre-hack wallets redeem depegged tokens at **1:1 USD** into **USDC** or **wONE**.
* **Daily caps**, **24h round delay**, **tiered fees**, **Merkle proofs**, **pause/emergency**, and **on-chain events** ensure fairness and safety.
* Frontend uses **React + Vite + Reown AppKit + ethers v6**.
* Tests cover whitelist, limits, rounds, fees, and security paths.
