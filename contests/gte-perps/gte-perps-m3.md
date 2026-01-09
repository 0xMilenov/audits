# User rewards are auto‑claimed to `Launchpad` during stake/unstake, causing permanent loss

In Launchpad.sol, LaunchToken.sol, and Distributor.sol, the bonding “fee share” accounting is wired so that user share changes are driven by the launch flow:

- In LaunchToken.sol, bonding transfers during the pre‑AMM phase trigger `ILaunchpad.increaseStake()` or `ILaunchpad.decreaseStake()` via [`_beforeTokenTransfer()`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/LaunchToken.sol#L120-L145). This happens on user buys and sells while bonding is active.

```solidity
function _beforeTokenTransfer(
        address from,
        address to,
        uint256 amount
    ) internal override {
    if (
        !unlocked && from != launchpad && to != launchpad && to != gteRouter
    ) {
        revert TransfersDisabledWhileBonding();
    }

    if (!unlocked) {
        if (from != launchpad && to != launchpad && to != gteRouter)
            revert TransfersDisabledWhileBonding();

        if (from == launchpad && to != launchpad)
            _increaseFeeShares(to, amount);
        else if (to != launchpad && to != gteRouter)
            revert TransfersDisabledWhileBonding();
    }

    if (from != launchpad) _decreaseFeeShares(from, amount);
}
```

- In [Launchpad.sol](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/Launchpad.sol#L582-L594), these calls are forwarded to `Distributor.increaseStake()` / `Distributor.decreaseStake()`.

```solidity
function increaseStake(
    address account,
    uint96 shares
) external onlyLaunchAsset {
    distributor.increaseStake(msg.sender, account, shares);
}

function decreaseStake(
    address account,
    uint96 shares
) external onlyLaunchAsset {
    distributor.decreaseStake(msg.sender, account, shares);
}
```

- In Distributor.sol, those functions interact with the rewards storage/tracker. Critically, both `increaseStake()` and `decreaseStake()` immediately call `_distributeAssets(...)` to pay any pending base/quote rewards returned by the tracker.

```solidity
function increaseStake(
    address launchAsset,
    address account,
    uint96 shares
) external onlyLaunchpad returns (uint256 baseAmount, uint256 quoteAmount) {
   ...
   ...
    _distributeAssets(launchAsset, baseAmount, rs.quoteAsset, quoteAmount);
}
...
...
function decreaseStake(
    address launchAsset,
    address account,
    uint96 shares
) external onlyLaunchpad returns (uint256 baseAmount, uint256 quoteAmount) {
   ...
   ...
    _distributeAssets(launchAsset, baseAmount, rs.quoteAsset, quoteAmount);
}
```

The crux is that [`_distributeAssets(...)`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/Distributor.sol#L227-L249) pays out to `msg.sender`. When invoked from `increaseStake()` / `decreaseStake()`, `msg.sender` is the `Launchpad` contract (not the user whose stake changed). As a result, any pending rewards triggered by a user’s share change are transferred to `Launchpad`, not to the user.

```solidity
function _distributeAssets(
    address base,
    uint256 baseAmount,
    address quote,
    uint256 quoteAmount
) internal {
    if (baseAmount > 0) {
        _decreaseTotalPending(base, baseAmount);
        base.safeTransfer(msg.sender, baseAmount); // <-- here the msg.sender is the launchpad if we increase/decrease
    }

    if (quoteAmount > 0) {
        _decreaseTotalPending(quote, quoteAmount);
        quote.safeTransfer(msg.sender, quoteAmount); // <-- here the msg.sender is the launchpad if we increase/decrease
    }
}
```

Highest‑impact scenario:

- Once any rewards are added for a bonding market, any subsequent buy/sell that adjusts bonding shares will auto‑claim accrued rewards for the affected user(s) and pay them to `Launchpad`. Users can permanently lose 100% of their earned rewards across both base and quote assets, while those tokens sit unclaimable in `Launchpad`.

> [!NOTE]
> If no third party adds rewards before bonding completes, no diversion occurs. However, adding rewards before graduation is a supported path (addRewards() can be invoked from anyone), so the higher‑impact scenario is 100% realistic. For example, teams frequently bootstrap a freshly launched token by seeding early incentives or liquidity mining during bonding—i.e., calling addRewards() pre‑graduation—to drive adoption and trading activity.

Root cause:

- `Distributor._distributeAssets(...)` transfers payouts to `msg.sender`.
- `Distributor.increaseStake()` and `Distributor.decreaseStake()` are callable only by `Launchpad`, so `msg.sender` is always `Launchpad` in those flows.
- Launchpad.sol does not expose ERC20 forwarding or recovery functions for these misdirected tokens (it only has `pullFees()` for native ETH). Tokens received there become trapped.

> [!NOTE] > `claimRewards()` in Distributor.sol still pays the caller (user) correctly because `msg.sender` is the user in that path. The diversion strikes specifically when share changes happen through bonding flows that route via `Launchpad`.

## Proof of Concept

To replicate the exact exploit, please add this inside `PoCLaunchpad.t.sol` and run:  
`forge test --match-test test_submissionValidity -vvv`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {LaunchpadTestBase} from "./LaunchpadTestBase.sol";
import {ERC20Harness} from "../harnesses/ERC20Harness.sol";
import {Distributor} from "contracts/launchpad/Distributor.sol";
import {ILaunchpad} from "contracts/launchpad/interfaces/ILaunchpad.sol";

contract PoCLaunchpad is LaunchpadTestBase {
    /**
     * PoC can utilize the following variables to access the relevant contracts:
     * - factory: ERC1967Factory.sol
     * - launchpad: Launchpad.sol
     * - distributor: Distributor.sol
     * - curve: SimpleBondingCurve.sol
     * - launchpadLPVault: LaunchpadLPVault.sol
     * - quoteToken: Quote token used in Launchpad system
     * - uniV2Router: Uniswap V2 Router used in Launchpad system
     */
    function test_submissionValidity() external {
        // 1) Give the user initial bonding shares by transferring base from Launchpad
        vm.prank(address(launchpad));
        ERC20Harness(token).transfer(user, 1 ether);

        // 2) Seed QUOTE rewards via an incentivizer while bonding
        address incentivizer = makeAddr("incentivizer");
        uint256 quoteReward = 10 ether;

        // Fund incentivizer with quote (mint)
        quoteToken.mint(incentivizer, quoteReward);

        vm.startPrank(incentivizer);
        quoteToken.approve(distributor, type(uint256).max);
        Distributor(distributor).addRewards(
            token,
            address(quoteToken),
            0,
            uint128(quoteReward)
        );
        vm.stopPrank();

        // 3) Verify the user has pending quote rewards
        (, uint256 pendingQuoteBefore) = Distributor(distributor)
            .getPendingRewards(token, user);
        assertGt(pendingQuoteBefore, 0);
        emit log_named_uint("Pending quote before the buy", pendingQuoteBefore);

        // Prepare funds for a buy and record Launchpad balances before
        // Use this contract as `account` to satisfy operator gating (account == msg.sender)
        uint256 quoteFund = 100 ether;
        quoteToken.mint(address(this), quoteFund);
        quoteToken.approve(address(launchpad), type(uint256).max);

        uint256 lpQuoteBefore = quoteToken.balanceOf(address(launchpad));
        emit log_named_uint(
            "Launchpad quote balance before the buy",
            lpQuoteBefore
        );

        // 4) Trigger a share change via a BUY, which auto-claims pending to Launchpad (msg.sender)
        ILaunchpad.BuyData memory buyData = ILaunchpad.BuyData({
            account: address(this),
            token: token,
            recipient: user,
            amountOutBase: 1 ether,
            maxAmountInQuote: type(uint256).max
        });

        (uint256 amountOutBaseActual, uint256 amountInQuote) = launchpad.buy(
            buyData
        );

        // 5) After buy, pending quote should be consumed for the user
        (, uint256 pendingQuoteAfter) = Distributor(distributor)
            .getPendingRewards(token, user);
        assertEq(pendingQuoteAfter, 0);

        uint256 lpQuoteAfter = quoteToken.balanceOf(address(launchpad));
        emit log_named_uint(
            "Launchpad quote balance after the buy",
            lpQuoteAfter
        );

        // 6) Validate diversion with precise accounting around the BUY transfer flows
        // Quote at Launchpad increases by trade amountInQuote plus misdirected pendingQuoteBefore
        uint256 diverted = lpQuoteAfter - lpQuoteBefore - amountInQuote;
        emit log_named_uint("Amount in quote", amountInQuote);
        emit log_named_uint(
            "Quote diverted to Launchpad due to auto-claim",
            diverted
        );
        assertEq(diverted, pendingQuoteBefore);
        assertEq(
            lpQuoteAfter,
            lpQuoteBefore + amountInQuote + pendingQuoteBefore
        );
    }
}
```

Logs:

```
Ran 1 test for test/c4-poc/PoCLaunchpad.t.sol:PoCLaunchpad
[PASS] test_submissionValidity() (gas: 453474)
Logs:
  Pending quote before the buy: 10000000000000000000
  Launchpad quote balance before the buy: 0
  Launchpad quote balance after the buy: 10000000010000000010
  Amount in quote: 10000000010
  Quote diverted to Launchpad due to auto-claim: 10000000000000000000
```

## Recommended mitigation steps

Preferred fix: Make the rewards recipient explicit and pay the intended user in stake/unstake flows.

- Change `_distributeAssets(...)` to accept a `recipient` parameter.
- In `increaseStake()` / `decreaseStake()`, pass the `account` (the user) as the recipient.
