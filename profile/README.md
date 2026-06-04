<div align="center">

<img src="https://raw.githubusercontent.com/SpryFinance/spry-contracts/main/assets/SPRY-Logo.png" width="110" alt="Spry logo" />

# Spry

**Dynamic-fee Uniswap V4 hooks that protect liquidity providers from arbitrage-driven impermanent loss.**

[![License: GPL-3.0](https://img.shields.io/badge/License-GPL--3.0-blue.svg)](https://github.com/SpryFinance/spry-contracts/blob/main/LICENSE)
[![Built on Uniswap V4](https://img.shields.io/badge/Built%20on-Uniswap%20V4-ff007a.svg)](https://github.com/Uniswap/v4-core)
![Status: testnet / pre-audit](https://img.shields.io/badge/status-testnet%20%C2%B7%20pre--audit-orange.svg)

</div>

---

## What is Spry?

Spry is a **Uniswap V4 hook**: a small contract that V4's singleton `PoolManager` consults on every swap to set the fee. Instead of one flat fee, a Spry pool charges a fee that scales with how much each trade moves the price — and with how much the *whole block* has already moved it.

Ordinary trades pay a low base rate. Large, arbitrage- and MEV-sized swaps pay much more, and that excess flows back to liquidity providers through V4's standard fee channel. The result: LPs are compensated for the impermanent loss those swaps would otherwise inflict.

## Why it exists

Constant-product AMMs leak value to arbitrageurs every time the price moves — that's impermanent loss. A flat fee (e.g. 0.30%) compensates badly: it's too high on tiny trades and far too low on the large rebalances that actually cause IL, so small traders end up subsidizing the cost created by big ones.

Spry replaces the flat fee with a curve that prices each swap by its *own* contribution to IL, so the takers causing the loss are the ones paying for it.

## How it works

- **Five fee tiers**, selected by a pool's `tickSpacing`, matched to asset volatility:

  | Tier | `tickSpacing` | Example pairs | Base fee |
  |---|---|---|---|
  | STABLE | 1 | USDC/USDT, stETH/ETH | 0.01% |
  | LIKE-ASSET | 10 | wstETH/ETH | 0.05% |
  | BLUE-CHIP | 60 | ETH/USDC, WBTC/ETH | 0.30% |
  | VOLATILE | 200 | ETH/SHIB | 0.50% |
  | EXOTIC | 1000 | low-cap pairs | 1.00% |

- **A four-zone curve per tier** — flat *safe* zone, a *linear* ramp, an *exponential* ramp, then a *cap* (up to 9.9%) — so the fee rises smoothly as a swap pushes the pool further from fair price.
- **Block-windowed MEV protection.** Each pool keeps a running cumulative of the block's net price shift; a swap is priced over the *path* it traverses, not in isolation.
- **Path-independence.** The fee is the integral of the curve over that path, so splitting one big swap into many small ones within a block costs **at least as much** — closing the multicall/sandwich-splitting loophole.
- **LP-friendly unwinds.** Swaps that push the pool back toward neutral pay only the base rate.

The full derivation lives in the whitepaper — [PDF](https://github.com/SpryFinance/spry-contracts/blob/main/assets/Spry-Whitepaper.pdf) (figure-driven) or [Markdown](https://github.com/SpryFinance/spry-contracts/blob/main/assets/Spry-Whitepaper.md).

## For liquidity providers

You provide liquidity through Uniswap's canonical V4 **`PositionManager`** — no custom Spry LP contract, no new token to hold. Your position is a standard ERC-721, with per-position fee accounting handled by V4. The dynamic fee simply means a larger share of arbitrage flow accrues to you.

## For traders & integrators

Spry pools are ordinary V4 pools. **Any V4-aware router or aggregator can trade against them** — the hook prices each swap automatically. Spry also ships a thin **swap-only router** with native-ETH, multi-hop, and Permit2 support.

## For developers

Everything is in **[`spry-contracts`](https://github.com/SpryFinance/spry-contracts)** (Foundry, Solidity ^0.8.26).

```bash
git clone https://github.com/SpryFinance/spry-contracts
cd spry-contracts
forge install      # v4-core, v4-periphery, OpenZeppelin, PRB-Math, Permit2
forge build
forge test         # 256 tests across unit / integration / scenarios / fuzz / fork
```

**Wiring a pool to Spry:** deploy `SpryHook` at a mined CREATE2 address (low 14 bits = `BEFORE_SWAP_FLAG`), then `initialize` a pool with `fee = DYNAMIC_FEE_FLAG` and `tickSpacing` set to your chosen tier. Liquidity goes through `PositionManager`; swaps through any V4 router. See `script/DeploySpry.s.sol`.

**Core surface:** `SpryHook` (dispatch + per-pool cumulative state), `SpryRouter` (swap-only), and the `SmartFeeLib` / `SpryFeeTypes` / `VirtualReserves` libraries.

## Project status

> ⚠️ **Pre-production.** The contracts are extensively tested (256 passing tests, ~100% library coverage, stateful invariant fuzzing) but **not yet externally audited**, and there is **no mainnet deployment**. Do not use with material funds until an independent audit is complete.

## Resources

- 📄 **Whitepaper** — [PDF](https://github.com/SpryFinance/spry-contracts/blob/main/assets/Spry-Whitepaper.pdf) (print-ready, with figures) · [Markdown](https://github.com/SpryFinance/spry-contracts/blob/main/assets/Spry-Whitepaper.md) (renders on GitHub)
- 💻 **Contracts** — [`spry-contracts`](https://github.com/SpryFinance/spry-contracts)

## License

[GPL-3.0-or-later](https://github.com/SpryFinance/spry-contracts/blob/main/LICENSE)
