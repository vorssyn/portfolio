---
title: "NoRekt (SharwaFinance/NoRektContracts) - Security Audit Report"
auditor: vorsyn
date: 2026-08-06
platform: Arbitrum
target: NoRekt - hedged-lending protocol (ETH deposit → USDC borrow → Hegic put options)
repo: https://github.com/SharwaFinance/NoRektContracts
total_loc_reviewed: ~3,200 (non-mock, Solidity 0.8.20)
audit_type: static review + on-chain parameter verification (cast, Arbitrum mainnet)
verdict: No HIGH/CRITICAL findings. 5 leads (1 confirmed live on-chain), all Low/Medium severity.
findings: F1 (Medium, design risk) · F2 (Low) · F3 (Low, CONFIRMED LIVE) · F4 (Low, mitigated) · F5 (Low, mitigated)
---

# NoRekt - Security Audit Report

**Auditor:** vorsyn
**Date:** 2026-08-06
**Platform:** Arbitrum (mainnet, verified on-chain)
**Repository:** https://github.com/SharwaFinance/NoRektContracts
**Scope:** ~3,200 LOC non-mock, Solidity 0.8.20, all contracts listed below
**Type:** Manual static review + on-chain parameter verification via `cast` (public RPC)

---

## 1. Scope

| File | LOC |
|---|---|
| `contracts/MarginTrading.sol` | 499 |
| `contracts/MarginAccount.sol` | 570 |
| `contracts/LiquidityPool.sol` | 405 |
| `contracts/oneClick/one_click_trading/OneClickOptions.sol` | 436 |
| `contracts/oneClick/no_rekt/OneClickNoRekt.sol` | 210 |
| `contracts/oneClick/facades/OneClickProxy.sol` | 234 |
| `contracts/modularSwapRouter/ModularSwapRouter.sol` | 364 |
| `contracts/modularSwapRouter/hegic/HegicModule.sol` | 192 |
| `contracts/modularSwapRouter/uniswap/UniswapModuleWithChainlink.sol` | 245 |
| `contracts/oneClick/one_click_liquidity_pool/OneClickLiquidityPool.sol` | 116 |
| `contracts/upkeepers/UpkeepOfLiquidations.sol` | 108 |

---

## 2. Executive Summary

NoRekt is a hedged-lending protocol: deposit ETH, borrow USDC, buy American-style put options (via Hegic integration) as downside protection. The "net LTV" accounting treats collateral + hedge together.

**Verdict: No reportable HIGH/CRITICAL exploitable bugs found in static review.**

The codebase is in good shape structurally: OpenZeppelin AccessControl + ReentrancyGuard via a clear role hierarchy (DEFAULT_ADMIN → MANAGER → MARGIN_TRADING → ONE_CLICK_PROXY → FACADE), Chainlink oracles with sequencer checks, and role-gated money movement. The risks that remain are operational (keeper incentives, oracle deviation thresholds, complex interest accounting) more than exploitable contract bugs.

Every finding was checked against the live deployment on Arbitrum mainnet (see §6, On-Chain Verification). One of them, F3, is confirmed live on-chain.

---

## 3. Findings Summary

| ID | Severity | Title | Status |
|---|---|---|---|
| F1 | Medium (design risk) | Liquidation keeper cross-chain deviation guard may block liquidations during divergence | Lead - unverifiable via RPC (contract address unresolved) |
| F2 | Low | `checkValidityERC721` has no explicit `return false` | Fragile but compiles correctly (0.8.20) |
| F3 | Low | `liquidatorFee` defaults to 0 - no liquidation incentive | 🔴 **CONFIRMED LIVE on-chain** |
| F4 | Low | `NO_YELLOW_ROLE` bypasses yellow-zone ratio guard | Mitigated on-chain (role held only by protocol contract) |
| F5 | Low | First-depositor fixed-share pattern in `LiquidityPool.provide()` | Mitigated on-chain (role-gated, no user path) |

---

## 4. Findings

### F1 (Medium / design risk) - Liquidation keeper cross-chain deviation guard may block during divergence

**File:** `contracts/upkeepers/UpkeepOfLiquidations.sol:88-107`

```solidity
function performUpkeep(bytes calldata performData) external override {
    uint marginAccountValueChainlink = marginTrading.calculateMarginAccountValue(optionID);
    uint marginAccountValueUniswap = marginTrading.calculateMarginAccountValueUSDC(optionID);
    uint minTotalMarginAccountValue = (marginAccountValueChainlink *
        (COEFFICIENT_DECIMALS * 100 - maxDeviationPercent)) / (COEFFICIENT_DECIMALS * 100);
    require(marginAccountValueUniswap >= minTotalMarginAccountValue, "...");
    oneClickLiquidation.liquidate(optionID, minTotalMarginAccountValue);
}
```

The liquidation keeper cross-checks `calculateMarginAccountValue` (Chainlink-priced) against `calculateMarginAccountValueUSDC` (Uniswap TWAP-priced), and `maxDeviationPercent` defaults to 30%. The guard exists to stop liquidations from firing on a bad price, but a flash crash is precisely when the two valuations diverge by >30%, so the liquidation gets blocked at the moment it is needed most. Underwater positions then sit and accumulate bad debt that the insurance pool ends up covering.

**Exploit path:** none directly (an attacker cannot trigger the divergence); systemic risk during market crashes.

**Recommendation:** reduce `maxDeviationPercent` to 5-10%, or add an admin-emergency liquidation bypass.

---

### F2 (Low) - `checkValidityERC721` has no explicit `return false`

**File:** `contracts/modularSwapRouter/hegic/HegicModule.sol:120-124`

```solidity
function checkValidityERC721(uint id) external returns (bool) {
    if (isOptionActive(id) && getExpirationTime(id) > block.timestamp) {
        return true;
    }
}
```

This compiles and behaves correctly in Solidity 0.8.20 (the default `bool` return is `false`), so it is not a live bug. I flag it anyway because a future refactor that adds a branch before that `return` would silently change the behavior, and implicit returns are exactly the kind of thing that survives review until it does not.

**Recommendation:** add explicit `return false;`.

---

### F3 (Low) - `liquidatorFee` defaults to 0 - no liquidation incentive 🔴 CONFIRMED LIVE

**File:** `contracts/MarginAccount.sol:59`

```solidity
uint public liquidatorFee = 0;
```

The liquidator commission (`userUSDCbalanceAfterRepay * liquidatorFee / COEFFICIENT_DECIMALS`) is 0 by default, and the remaining base-token balance goes to the account *owner*, not the liquidator. That leaves `UpkeepOfLiquidations` (the Chainlink keeper) as the only liquidation trigger, and its automation rewards are typically under $1/check while the liquidation gas cost with swaps can run $10-50+ on Arbitrum. If the keeper does not submit, underwater positions persist with nobody properly incentivized to close them.

**On-chain confirmation:** `MarginAccount.liquidatorFee()` returns **0** on Arbitrum mainnet.

**Recommendation:** set `liquidatorFee` to at least 1-2% in the deploy script.

---

### F4 (Low) - `NO_YELLOW_ROLE` bypasses yellow-zone ratio guard

**File:** `contracts/oneClick/facades/OneClickProxy.sol:152-169, 180-186`

```solidity
function withdrawERC20NoYellow(...) external onlyRole(NO_YELLOW_ROLE) { ... }
function borrowNoYellow(...) external onlyRole(NO_YELLOW_ROLE) { ... }
```

Both functions bypass `ensureYellowCoeffForDebt` (ratio ≥ 1.20), so whoever holds `NO_YELLOW_ROLE` can borrow or withdraw down to the red line (1.17). That only becomes a problem if the role is granted to a user-facing contract with deficient access control: a user could then drain their position to the liquidation line with no warning buffer.

**On-chain confirmation:** role held **only** by `OneClickEphemeralSwapOutput` (protocol's own contract, itself guarded by `onlyApprovedOrOwner`). No EOA or insurance-pool exposure. **Mitigated in the live deployment.**

---

### F5 (Low) - First-depositor fixed-share pattern in `LiquidityPool.provide()`

**File:** `contracts/LiquidityPool.sol:168-170`

```solidity
uint shareChange = totalLiquidity > 0
    ? (depositShare * amount) / totalLiquidity
    : 1e18;
```

The first depositor gets exactly `1e18` shares no matter the deposit size (so a 1-wei deposit gets a share price of 1e-18). The classic inflation attack is only partially mitigated here: `currentPoolBalance` is accounting-based rather than `balanceOf`-based, which removes the direct donation vector, but if `getTotalLiquidity()` could be inflated through borrow+interest timing, the share price becomes exploitable.

**Mitigation confirmed on-chain:** `provide()` is gated by `PROTOCOL_ROUTER_ROLE`; on both live pools (USDC + WETH) that role is held **only** by `MarginAccount`. No user-callable path exists in the live deployment.

---

## 5. Verified Safe (surfaces cleared during review)

| Surface | Status | Reason |
|---|---|---|
| MarginAccount ERC20/721 tracking | ✅ Clean | role-gated nonReentrant, balance pre-checks |
| MarginAccount swap / borrow / repay / exercise | ✅ Clean | balance checks, caps at debt, role-gated |
| MarginTrading pass-throughs | ✅ Clean | ratio checks after state changes |
| OneClickProxy provide/exercise/swap | ✅ Clean | ownership checks, option metadata stored |
| ModularSwapRouter liquidate / swap | ✅ Clean | role-gated, per-module delegation |
| HegicModule exercise/liquidate | ✅ Clean | active+ITM+not-expired checks |
| UniswapModuleWithChainlink | ✅ Clean | sequencer check, grace period, staleness, price>0 |
| LiquidityPool borrow/repay/provide/withdraw | ✅ Clean | 80% utilization cap, role-gated |
| OneClickNoRekt | ✅ Clean | onlyApprovedOrOwner, multi-exercise |
| UpkeepOfLiquidations.checkUpkeep | ✅ Clean | ratio <= redCoeff scan |

**Call chain (safe, clear separation):**

```
User → OneClickNoRekt (user facade, onlyApprovedOrOwner)
     → OneClickProxy (FACADE_ROLE - yellowCoeff checks + option metadata)
         → MarginTrading (ONE_CLICK_PROXY_ROLE - redCoeff checks + exercise/liquidate)
             → MarginAccount (MARGIN_TRADING_ROLE - pure storage, nonReentrant)
                 → ModularSwapRouter (MARGIN_ACCOUNT_ROLE - valuation + swap routing)
                     → HegicModule / UniswapModuleWithChainlink
```

Each layer adds orthogonal checks; no single layer can move money without upstream authorization.

**Design strengths worth naming:** valuation is Chainlink-only, so there is no TWAP manipulation surface; the oracle guards include a 1h grace period, staleness checks, and price>0 checks; the ratio checks are conservative, with options excluded from borrow power but included at liquidation, which is how the hedge is supposed to work; and actual swaps run with 0.3% slippage.

---

## 6. On-Chain Verification (Arbitrum mainnet, 2026-08-06)

**Method:** `cast call` on deployed addresses via public RPC (`arb1.arbitrum.io/rpc`). All contracts source-verified on Arbiscan (Solidity 0.8.20).

**Deployed addresses:**

| Contract | Address | Source |
|---|---|---|
| MarginAccount | `0x97855B5E8F454a0953470Fe13E99E331c7193a72` | frontend bundle |
| MarginTrading | `0xdebA477012d831a2eEB7922c2c9b45b632f4F4C7` | `OneClickProxy.marginTrading()` |
| OneClickProxy | `0xFF97d777b8080A101Ac72196fEe280ed7318F846` | frontend bundle |
| MarginAccountManager | `0x512DecE525b1Fe2EE072983f7fF477AD87273B25` | frontend bundle |
| ModularSwapRouter | `0x8b97c7078292033E2DCC3991826F50e5933E59Fe` | frontend bundle |
| USDC LiquidityPool | `0xFCCC86B1759CF6bD37F420C225F5e74EB6F664CE` | frontend bundle |
| WETH LiquidityPool | `0xE79826F7e7813c306e11095929e023F8FdbBFec3` | `MarginAccount.tokenToLiquidityPool(WETH)` |
| OneClickNoRekt | `0x662B6ea40b2aE771469Cb0718D0b6074A00a7d43` | frontend bundle |
| OneClickEphemeralSwapOutput | `0x2DA1990A6D1Ea9d161A3D613c954f900aF0dc225` | `OneClickNoRekt.oneClickEphemeralSwapOutput()` |

**Check results:**

| Check | Call | Result | Verdict |
|---|---|---|---|
| F3: `liquidatorFee` | `MarginAccount.liquidatorFee()` | **0** | 🔴 CONFIRMED LIVE |
| Context: `redCoeff` | `MarginTrading.redCoeff()` | 117000 (1.17) | ✅ matches code |
| Context: `yellowCoeff` | `OneClickProxy.yellowCoeff()` | 120000 (1.20) | ✅ matches code |
| F4: `NO_YELLOW_ROLE` → OneClickEphemeralSwapOutput | `hasRole` | **true** | 🟡 only protocol contract, matches deploy scripts |
| F4: `NO_YELLOW_ROLE` → insurance pool / MarginAccount | `hasRole` | false | ✅ no rogue grants |
| F5: `PROTOCOL_ROUTER_ROLE` → MarginAccount (USDC pool) | `hasRole` | **true** | ✅ provide() only by MarginAccount |
| F5: `PROTOCOL_ROUTER_ROLE` → insurance pool (USDC pool) | `hasRole` | false | ✅ |
| F5: `PROTOCOL_ROUTER_ROLE` → MarginAccount (WETH pool) | `hasRole` | true | ✅ |
| USDC pool TVL | `getTotalLiquidity()` | ~$10.45k | live deposits |
| WETH pool TVL | `getTotalLiquidity()` | 0 | empty |
| Margin accounts | `balanceOf(MarginAccount)` | 0 | no borrowers yet |

**Post-verification status:**

- **F3**: LIVE. `liquidatorFee = 0` confirmed on-chain, so there is no financial incentive for external liquidators; liquidation depends entirely on the Chainlink keeper or manual admin action.
- **F4**: exposure limited to the protocol's own contract (downgraded from lead to informational).
- **F5**: mitigated. `PROTOCOL_ROUTER_ROLE` held only by MarginAccount on both pools; `provide()` not user-callable (downgraded from lead to informational).
- **F1**: unverifiable via RPC. The `UpkeepOfLiquidations` address is not present in the frontend bundle, so deployment status is unresolved.

**Key takeaway:** the protocol is live with ~$10.4k TVL but **zero margin accounts** (no borrowers, no option positions), so the money at risk is LP deposits only. All role grants match the deploy scripts and no rogue roles turned up. The single confirmed live parameter issue is F3 (`liquidatorFee = 0`).

---

## 7. Recommendations

1. **Set `liquidatorFee` to 1-2%** in deploy scripts. Right now liquidations depend entirely on Chainlink keeper economics, and they may fail during gas spikes.
2. **Reduce `maxDeviationPercent`** or add an emergency liquidation bypass in `UpkeepOfLiquidations`.
3. **Add explicit `return false`** to `checkValidityERC721`.
4. **Fuzz test `_repay`** interest-accounting math. The 5-variable simultaneous update is too complex to clear with static verification alone.
5. **Keep NO_YELLOW_ROLE and PROTOCOL_ROUTER_ROLE tightly held**; add events on role grants/revocations.

---

*Report by vorsyn. All on-chain checks performed via `cast` against Arbitrum mainnet public RPC; results reproducible with the addresses above.*
