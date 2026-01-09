## Perps

### [L-01] Missing empty-limit assertion in matching can enable DoS via null head dereference in CLOBLib

`CLOBLib._matchIncomingBid()` and `CLOBLib._matchIncomingAsk()` dereference `limit.headOrder` at the best price without first asserting that there are still orders at that limit. If, due to a prior fill/cancel/expire/amend or a transient state during the same transaction, a price node remains in the tree while its `Limit` is empty, `limit.headOrder` becomes `0` and subsequent logic operates on a null order. This can lead to revert or gas grief:

- In quote‑denominated paths, `price == 0` on a null maker can lead to division by zero in `BookLib.getTradedAmounts()`, reverting taker matches.
- In base‑denominated paths, `filledAmount == 0` and `incomingOrder.amount` may not decrease, risking excessive gas consumption until the loop breaks.
- The condition is realistic whenever the last order at a price is removed or expires between reading `bestAsk`/`bestBid` and dereferencing `limit.headOrder`.

File: [contracts/perps/types/Book.sol](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/types/Book.sol#L93-L95)

```solidity
function assertOrdersAtLimit(
    Book storage self,
    uint256 price,
    Side side
) internal view {
    if (self.getLimit(price, side).numOrders == 0) revert NoOrdersAtLimit();
}
```

File: [contracts/perps/types/CLOBLib.sol](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/types/CLOBLib.sol#L338-L398)

```solidity
Limit storage limit = ds.askLimits[bestAsk];
Order storage bestAskOrder = ds.orders[limit.headOrder];

if (bestAskOrder.isExpired()) {
    _removeUnfillableOrder(ds, bestAskOrder);
    bestAsk = ds.getBestAsk();
    continue;
}
...
...
Limit storage limit = ds.bidLimits[bestBid];
Order storage bestBidOrder = ds.orders[limit.headOrder];

if (bestBidOrder.isExpired()) {
    _removeUnfillableOrder(ds, bestBidOrder);
    bestBid = ds.getBestBid();
    continue;
}
```

Recommendation

- Before dereferencing `limit.headOrder`, assert the level is non‑empty:
  - In `_matchIncomingBid()`: add `ds.assertOrdersAtLimit(bestAsk, Side.SELL);`
  - In `_matchIncomingAsk()`: add `ds.assertOrdersAtLimit(bestBid, Side.BUY);`

### [L-02] Inconsistent fee tier bounds cause zero‑fee reads or setup reverts in PackedFeeRatesLib

The library packs 16 `uint16` fee tiers into a `uint256`, but [`packFeeRates()`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/types/PackedFeeRatesLib.sol#L14-L31) rejects arrays longer than 15 (`fees.length > 15`), while `getFeeAt()` permits indices up to 15 (reverts only if `index > 15`). This mismatch makes tier 15 unreachable when using `packFeeRates()`: a read at index 15 will always return `0`, potentially applying a zero fee if business logic selects tier 15. Conversely, any configuration expecting 16 tiers will revert during packing via `TooManyFeeTiers()`, causing a setup or upgrade denial of service. The divergence between `packFeeRates()` and `getFeeAt()` also increases the risk of off‑by‑one mistakes in call sites.

```solidity
function packFeeRates(
  uint16[] memory fees
) internal pure returns (PackedFeeRates) {
  if (fees.length > 15) revert TooManyFeeTiers();  // <-- here

  uint256 packedValue;
  for (uint256 i; i < fees.length; i++) {
      packedValue = packedValue | (uint256(fees[i]) << (i * 16));
  }

  return PackedFeeRates.wrap(packedValue);
}

function getFeeAt(
  PackedFeeRates fees,
  uint256 index
) internal pure returns (uint16) {
  if (index > 15) revert IndexOutOfBounds();

  uint256 shiftBits = index * 16;

  return uint16((PackedFeeRates.unwrap(fees) >> shiftBits) & 0xFFFF);
}
```

Recommendation: Standardize the bounds. Define a single constant (e.g., `MAX_FEE_TIERS = 16`) and enforce `fees.length <= 16` in `packFeeRates()` and `index < 16` in `getFeeAt()`. Additionally, ensure call sites validate that a requested tier is within the configured tier count to avoid relying on implicit zeroes.

### [L-03] Incorrect admin-role check in ViewPort causes false negatives for real admins

[`ViewPort.isAdmin()`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/modules/ViewPort.sol#L172-L175) uses `hasAllRoles(account, 7)` where `Constants.ADMIN_ROLE` is defined as `1 << 7`. With `OwnableRoles.hasAllRoles()` performing a strict bitmask check (`rolesOf(user) & roles == roles`), the mask `7` (bits 0–2) does not match the admin bit (bit 7). As a result, real admins (with `Constants.ADMIN_ROLE`) will fail the check (`(1 << 7) & 7 = 0 != 7`), while accounts that happen to be granted low bits 0–2 could incorrectly pass. This leads to misleading admin status in any off-chain or cross-contract logic that relies on `isAdmin()` (even if core authorization paths use the correct checks elsewhere).

```solidity
function isAdmin(address account) external view returns (bool) {
    return hasAllRoles(account, 7);
}
```

Recommendation: Update `isAdmin()` to check the correct role bit:

- Use `hasAllRoles(account, Constants.ADMIN_ROLE)` (or `hasAnyRole(account, Constants.ADMIN_ROLE)` if semantics allow).

### [L-04] Missing condition in CLOBLib order update prevents updating only cancel timestamp

The order update logic in [`contracts/perps/types/CLOBLib.sol`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/types/CLOBLib.sol#L514-L531) does not handle the case where only `args.cancelTimestamp` differs from `order.cancelTimestamp`. As a result, supplying a new cancel timestamp has no effect unless another field also changes, leaving `order.cancelTimestamp` stale. This reduces flexibility for callers and may lead to unexpected cancellation behavior for off-chain consumers relying on the field.

```solidity
function _processAmend(
    Book storage ds,
    Order storage order,
    AmendLimitOrderArgs calldata args
) internal returns (int256 notionalDelta, int256 collateralDelta) {
    if (
        args.expiryTime.isExpired() ||
        args.baseAmount <
        StorageLib
            .loadBookSettings(ds.config.asset)
            .minLimitOrderAmountInBase
    ) {
        revert InvalidAmend();
    } else if (order.side != args.side || order.price != args.price) {
        // change place in book
        return _executeAmendNewOrder(ds, order, args);
    } else {
        // change amount
        return _executeAmendAmount(ds, order, args);
    }
```

Recommendation: Add an explicit condition that compares `args.cancelTimestamp` to `order.cancelTimestamp` and updates it when they differ, ensuring it executes independently of other field updates (with any required validation such as bounds checks or zero-as-disabled semantics), e.g., set `order.cancelTimestamp = uint32(args.cancelTimestamp)`.

## Launchpad

### [L-01] Same-block fee accrual enables LP mint/burn against pending fees in `GTELaunchpadV2Pair`

When swaps accrue launchpad fees within the same block, [`_update()`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/uniswap/GTELaunchpadV2Pair.sol#L117-L159) writes `reserve0`/`reserve1` as `balance0 - totalLaunchpadFee0` and `balance1 - totalLaunchpadFee1`. In that state, `mint()` measures deposits as `balance - reserve`, which equals the pending fees even if no tokens were transferred in the transaction. Subsequently, `burn()` calculates payouts from raw `balance0`/`balance1` (which still include the pending fees), allowing an LP to mint cheaply and then burn to withdraw a proportional share of those fees. This causes fee leakage and unfair distribution to existing LPs, and can be exploited opportunistically within the same block.

```solidity
function _update(
   uint256 balance0,
   uint256 balance1,
   uint112 _reserve0,
   uint112 _reserve1,
   uint112 newLaunchpadFee0,
   uint112 newLaunchpadFee1
) private {
...
...
...
   reserve0 = _reserve0 = uint112(balance0) - totalLaunchpadFee0;
   reserve1 = _reserve1 = uint112(balance1) - totalLaunchpadFee1;
```

Recommendation: Normalize fee handling across `mint()` and `burn()` by computing deltas and payouts from balances net of `totalLaunchpadFee0`/`totalLaunchpadFee1`, or require fee settlement/distribution before executing `mint()`/`burn()` when accrued fees are non-zero. Ensuring both reserve updates and LP accounting use the same fee-excluded basis removes the discrepancy.

### [L-02] Bitwise OR in launchpad fee checks can skip fee distribution in `GTELaunchpadV2Pair`

The pair contract inside [`_update`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/uniswap/GTELaunchpadV2Pair.sol#L117-L159) uses the bitwise OR operator `|` in conditions intended to check whether any launchpad fee amounts are non-zero, for example `totalLaunchpadFee0 | totalLaunchpadFee1 > 0` and `newLaunchpadFee0 | newLaunchpadFee1 > 0`. This is a logical check that should use `||`. Relying on `|` and operator precedence can lead to incorrect branching, potentially skipping calls to `_distributeLaunchpadFees()` or clearing `accruedLaunchpadFee0`/`accruedLaunchpadFee1` without proper distribution. As a result, fees may remain in the contract and be retrievable via `skim()` instead of being routed to the intended fee recipient.

```solidity
function _update(
   uint256 balance0,
   uint256 balance1,
   uint112 _reserve0,
   uint112 _reserve1,
   uint112 newLaunchpadFee0,
   uint112 newLaunchpadFee1
) private {
...
...
if (launchpadFeeDistributor > address(0)) {
    if (totalLaunchpadFee0 | totalLaunchpadFee1 > 0) { // <-- here
...
...
 else if (
    launchpadFeeDistributor > address(0) &&
    newLaunchpadFee0 | newLaunchpadFee1 > 0 // <-- here
) {
```

Recommendation: replace bitwise OR with logical OR and use explicit comparisons with clear parentheses, e.g.:

- `if (totalLaunchpadFee0 > 0 || totalLaunchpadFee1 > 0) { ... }`
- `... && (newLaunchpadFee0 > 0 || newLaunchpadFee1 > 0) { ... }`

### [L-03] Accrued fee rewards may be cleared and unintentionally redistributed to LPs in `GTELaunchpadV2Pair` by `endRewardsAccrual()`

[`endRewardsAccrual()`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/uniswap/GTELaunchpadV2Pair.sol#L97-L115) clears the fee accrual state before distributing rewards. If fees are still pending at the time of the call, those funds are implicitly absorbed into pool reserves (`liquidity`), increasing LP share value and causing a pro‑rata gift to LPs rather than sending the intended amount to the `distributor`. This misallocates protocol/distributor revenue and makes rewards accounting dependent on call timing.

```solidity
function endRewardsAccrual() external {
if (msg.sender != launchpadFeeDistributor)
    revert("GTEUniV2: FORBIDDEN");

delete accruedLaunchpadFee0;
delete accruedLaunchpadFee1;
delete rewardsPoolActive;

_update(
    IERC20(token0).balanceOf(address(this)),
    IERC20(token1).balanceOf(address(this)),
    reserve0,
    reserve1,
    uint112(0),
    uint112(0)
);

emit RewardsPoolDeactivated();
}
```

Recommendation: distribute accrued fees first and only then clear state. Snapshot the `fee accrual state` at the start of `endRewardsAccrual()`, transfer that amount to the `distributor`, then update/zero internal counters. Consider asserting the post‑distribution state and adding tests to prevent regressions.

### [L-04] Launchpad allows zero-fee token launches due to missing fee initialization

The `Launchpad` fee is not initialized in [`initialize()`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/Launchpad.sol#L169-L195), leaving `launchFee` at its default value of `0` until it is later set via `updateLaunchFee()`. This permits the token launch flow to proceed without charging any fee, enabling fee circumvention and causing protocol revenue loss.

Recommendation: Set a non-zero `launchFee` during `initialize()` (e.g., pass it as a parameter) and enforce `require(newLaunchFee > 0)` in both `initialize()` and `updateLaunchFee()`. Optionally, define a `minLaunchFee` and enforce `newLaunchFee >= minLaunchFee` to prevent accidental or malicious zero-fee configurations.

### [L-05] Bonding curve update in `Launchpad` does not call `init()`, leaving virtual reserves unset

When the active bonding curve is changed inside [`updateBondingCurve`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/Launchpad.sol#L369-L378), the new curve instance stored in `currentBondingCurve` is not initialized via `init(bondingCurveInitData)`. This means its virtual reserves remain unset until a privileged address later calls `setVirtualReserves()` or `setReserves()`. During this window, interactions that rely on the curve’s state may be exposed to front‑running attempts and unexpected reverts. Even if dust-attack checks mitigate certain exploits, the gap creates avoidable operational risk and a potential temporary DoS until initialization occurs.

```solidity
function updateBondingCurve(address newBondingCurve) external onlyOwner {
   if (!ERC165Checker.supportsInterface(newBondingCurve, type(IBondingCurveMinimal).interfaceId)) {
       revert InvalidCurve();
   }

   emit BondingCurveUpdated(address(currentBondingCurve), newBondingCurve, LaunchpadEventNonce.inc());

   currentBondingCurve = IBondingCurveMinimal(newBondingCurve);
}
```

Recommendation: After updating `currentBondingCurve`, immediately call `currentBondingCurve.init(bondingCurveInitData)` in the same transaction, and gate external interactions on an `initialized` flag. Alternatively, require the reserves/initialization data to be provided atomically with the update and revert if the curve remains uninitialized post-update.

### [L-06] End‑rewards in LaunchToken only callable pre‑AMM, not reachable in practice

The logic `if (totalFeeShare == 0 && !unlocked) _endRewards();` inside [`_decreaseFeeShares`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/LaunchToken.sol#L134-L150) ties the `_endRewards()` trigger to a state where `unlocked` is false (pre‑launch), i.e., before the AMM pair is created. When `_endRewards()` try to interacts with the AMM pair, calling it before pair deployment will either revert or no‑op, preventing the end‑rewards flow from functioning as intended. This makes the mechanism practically unreachable in normal operation and may leave reward accounting unsettled.

```solidity
function _decreaseFeeShares(address account, uint256 amount) internal {
...
...
        if (totalFeeShare == 0 && !unlocked) _endRewards();
...
...
function _endRewards() internal {
        ILaunchpad(launchpad).endRewards();
...
...
function endRewards() external onlyLaunchAsset {
    address quote = _launches[msg.sender].quote;

    IGTELaunchpadV2Pair pair = IGTELaunchpadV2Pair(
        address(pairFor(address(uniV2Factory), msg.sender, quote))
    );

    distributor.endRewards(pair);
}
...
...
function endRewards(IGTELaunchpadV2Pair pair) external onlyLaunchpad {
    pair.endRewardsAccrual();
}
```

Recommendation: Allow `_endRewards()` to be callable after AMM creation by adjusting the condition (e.g., require `unlocked` or presence of a pair address), or add a guard inside `_endRewards()` to handle the absence of the pair.

```diff
+ if (totalFeeShare == 0 && unlocked) _endRewards();
```

### [L-07] Using msg.sender in Launchpad \_swapRemaining() can cause router path reverts

[`_swapRemaining()`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/Launchpad.sol#L535-L559) uses `msg.sender` for both pulling funds (`safeTransferFrom(msg.sender, address(this), data.quoteAmount)`) and refunding on failure (`safeTransfer(msg.sender, data.quoteAmount)`). When this path is invoked via a router, `msg.sender` is the router, which typically has neither allowance nor balance for `data.quote`, causing the call to revert before the Uniswap swap is attempted. Even if allowances were somehow present, refunds wouldn't be sent to the router instead of the end user, because we dont expect to have any balance inside the router. This creates a DoS for the router execution path. The issue occurs within `Launchpad.sol` in `_swapRemaining()`, which also calls `uniV2Router.swapTokensForExactTokens()`.

```solidity
    function _swapRemaining(
        SwapRemainingData memory data
    ) internal returns (uint256, uint256) {
        // Transfer the remaining quote from the user
        data.quote.safeTransferFrom(
            msg.sender, // <-- here
            address(this),
            data.quoteAmount
        );

        address[] memory path = new address[](2);

        path[0] = data.quote;
        path[1] = data.token;

        data.quote.safeApprove(address(uniV2Router), data.quoteAmount);

        try
            uniV2Router.swapTokensForExactTokens(
                data.baseAmount,
                data.quoteAmount,
                path,
                data.recipient,
                block.timestamp + 1
            )
        {
            return (data.baseAmount, data.quoteAmount);
        } catch {
            data.quote.safeApprove(address(uniV2Router), 0);
            data.quote.safeTransfer(msg.sender, data.quoteAmount); // <-- here
            return (0, 0);
        }
    }
```

Recommendation: Pass explicit `payer` and `refundTo` addresses into `SwapRemainingData` (or as parameters) and use those instead of `msg.sender`. Pull funds with `data.quote.safeTransferFrom(payer, address(this), data.quoteAmount)` and, in the failure path, refund with `data.quote.safeTransfer(refundTo, data.quoteAmount)`. Ensure the router propagates the original buyer as `payer`/`refundTo` or transfers user funds to the Launchpad before invoking `_swapRemaining()`, avoiding reliance on `msg.sender` inside internal helpers.
