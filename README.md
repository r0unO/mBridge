[mach-bridge-README.md](https://github.com/user-attachments/files/32661789/mach-bridge-README.md)
# MachBridge

A Mach Enterprises prototype of an mBridge-style **wholesale multi-CBDC settlement
platform**: permissioned participants, one token per currency, and atomic
payment-versus-payment (PvP) cross-border settlement. Solidity 0.8.24, built and tested
with Foundry, targeting Ubuntu.

> **Status: design prototype.** It is not the real mBridge ledger (whose client, code and
> deployment approval process are not public), it has **not been audited**, and the tokens
> are *simulated* CBDC for a private consortium, not legal tender. It was written without
> access to a compiler, so run `forge build` and `forge test` first and fix anything they flag.

## Quick start (Ubuntu 22.04 / 24.04)

```bash
git clone https://github.com/<your-username>/mach-bridge.git
cd mach-bridge
chmod +x scripts/*.sh         # git preserves exec bits, but set them if it ever complains
./scripts/setup-ubuntu.sh     # installs Foundry, pulls OpenZeppelin v5.0.2 + forge-std, builds, runs tests
./scripts/dev-chain.sh        # starts Anvil (chain id 7777) and deploys the demo setup
kill $(cat .anvil.pid)        # stop the local chain
```

Replace `<your-username>` with the account or org the repo lives under. If the repo is private, clone over SSH instead: `git clone git@github.com:<your-username>/mach-bridge.git`.

## Architecture

| mBridge concept                       | MachBridge component                                                                 |
|---------------------------------------|--------------------------------------------------------------------------------------|
| Central banks operate validator nodes | `ParticipantRegistry` tier `CentralBank`, admitted by the governance admin           |
| Commercial banks join via their CB    | `onboardCommercialBank()`, callable only by an active central bank; jurisdiction is inherited |
| Observers                             | Tier `Observer`: registered, but cannot hold or move tokens                          |
| Per-currency CBDC                     | `CBDCToken`: only the issuing CB mints/burns; only registered, non-frozen holders    |
| Atomic cross-border FX payment        | `SettlementEngine`: escrow, then LP `fill()` swaps both legs in one transaction      |
| Governance / rulebook                 | Admin role over corridors and pausing (a multisig of central banks in production)    |

### Settlement flow

1. Governance enables a corridor (`setCorridor(mCAD, mAED, true)`).
2. A liquidity-provider bank posts a quote: rate, max size, expiry.
3. The payer bank approves the engine, then calls `submitPayment(...)`. `tokenIn` is escrowed and the rate is locked.
4. The LP calls `fill(id)`: LP's `tokenOut` goes to the payee and the escrowed `tokenIn` goes to the LP, atomically.
5. If the LP does not fill, anyone can `cancel(id)` after the deadline and the payer is refunded. The LP can also reject early. `cancel` is not pausable, so escrow is never trapped.

Rates are "whole tokenOut per whole tokenIn" scaled by 1e18; decimals are handled by the engine.

## Design decisions to review

- **EVM target is `paris`** (no `PUSH0`) so bytecode runs on more permissioned clients. Match it to whatever your validator client supports.
- **`via_ir = true`** is enabled as insurance against "stack too deep" in the struct-heavy engine. Remove it if you prefer faster builds and it compiles without.
- **LP-quoted rates, not a central oracle.** This avoids a single rate authority, but LPs carry the FX risk and can decline to fill. Central-bank rate bands are an obvious extension.
- **Frozen or deregistered payer** cannot be refunded on-chain (the refund transfer reverts); resolve through governance.
- **Anvil is single-node.** It is for development only.

## Path to a real multi-validator network

Anvil has one signer. For one validator per central bank, run an EVM-compatible permissioned client such as Hyperledger Besu with QBFT consensus, node/account permissioning, and the same contracts deployed via `forge script --rpc-url <node>`. Before that:

1. Replace the single admin with a multisig or governance contract of the central banks.
2. Move participant identity and compliance evidence off-chain (only hashes on-chain) and decide what your privacy model needs; balances here are visible to all validators.
3. Encode the legal rulebook (finality, dispute handling, liability) alongside the code.
4. Independent audit, invariant/fuzz tests (escrow conservation, supply = sum of balances), and key management (HSM/KMS, no dev keys).
5. Confirm regulatory position with counsel before handling anything resembling real CBDC or client funds.

## Layout

```
src/ParticipantRegistry.sol   membership + sponsorship
src/CBDCToken.sol             restricted wholesale token
src/SettlementEngine.sol      quotes, escrow, atomic PvP
test/Settlement.t.sol         end-to-end + failure-path tests
script/DeployDev.s.sol        local demo deployment (Anvil keys, dev only)
scripts/                      Ubuntu setup and local-chain helpers
```
