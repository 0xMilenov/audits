# Early markets can be DoSed from bootstrapping due to undercollateralized mint block in LatentSwapLEX and LatentSwapLogic

- **Project:** Covenant
- **Finding ID:** F-54 / S-73
- **Severity:** Low
- **Reporter:** 0xMilenov

---

## Description

In early markets with low `baseTokenSupply`, an attacker can acquire a large proportion of zTokens (e.g., by swapping aToken → zToken near the debt limit). A small adverse price move then flips the market to undercollateralized in `_calculateMarketState()` (when `dexAmounts[DEBT] > maxDebtValue`, `underCollateralized` is set).

Once undercollateralized, `mint()` and base-in `swap()` paths revert via `_checkUnderCollateralized()`, preventing honest users from adding base liquidity and effectively stalling market bootstrapping.

This exact issue was previously reported by **Pashov Audit Group** and marked as **RESOLVED**, yet the current implementation still contains the same undercollateralized blocking logic, indicating the mitigation was never applied.

---

## Affected Code

**LatentSwapLogic.sol**

    function _calcMint(
        LexParams memory lexParams,
        LexFullState memory marketState,
        uint256 baseTokenAmountIn
    ) internal pure returns (uint256 aTokenAmountOut, uint256 zTokenAmountOut) {
        ///////////////////////////////
        // Validate inputs

        // Check if undercollateralized (reverts if so)
        _checkUnderCollateralized(marketState);

        // Check LTV before mint (reverts if LTV above MAX_LIMIT_LTV)
        // @dev - minting aTokens when above MAX_LIMIT_LTV is blocked to avoid excessive aToken dilution
        // in high LTV or undercollateralized markets.
        _checkLTV(
            marketState.lexState.lastSqrtPriceX96,
            lexParams.limMaxSqrtPriceX96
        );

        // Check if mint amount is too large given market size
        _checkMintCap(
            marketState.baseTokenSupply,
            marketState.lexState.lastETWAPBaseSupply,
            baseTokenAmountIn,
            marketState.lexConfig.noCapLimit
        );
    }

---

## Impact

In early stages, a single actor can force the market into a state where further base additions are blocked, delaying or preventing organic growth until external conditions resolve the undercollateralization.

---

## Recommendation

Allow base-in liquidity additions while undercollateralized if they strictly improve or do not worsen LTV (e.g., ensure `nextSqrtPriceX96 <= currentSqrtPriceX96` and within limits), or gate such actions to admin / `CovenantCore` with the same monotonic LTV constraint.

Concretely, relax `_checkUnderCollateralized()` for `mint()` and base → synth exact-in `swap()` when the computed `nextSqrtPriceX96` indicates non-increasing LTV, while keeping z-side redemptions / swaps that worsen LTV blocked.
