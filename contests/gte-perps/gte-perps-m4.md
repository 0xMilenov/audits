# Permanent disablement of launchpad rewards by pre-creating the pair

The launchpad stack comprises:

- [`GTELaunchpadV2PairFactory.sol`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/uniswap/GTELaunchpadV2PairFactory.sol): deploys Uniswap V2-style pairs via `createPair()` and records them in `getPair[token0][token1]`.
- [`GTELaunchpadV2Pair.sol`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/uniswap/GTELaunchpadV2Pair.sol): a modified pair that accrues a launchpad swap-fee share and forwards it to `Distributor` when `launchpadFeeDistributor` is set; the fee share is proportional to `launchpadLp`’s LP balance.
- [`Launchpad.sol`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/Launchpad.sol): upon graduation, it calls the factory to ensure/create the pair and then adds liquidity to `launchpadLPVault`. Subsequent trades are expected to accrue rewards for the launchpad program.
- [`Distributor.sol`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/Distributor.sol): maintains reward pools, receives the forwarded fees from pairs, and accounts/pays them to participants.

Flow and interactions:

- In `@GTELaunchpadV2PairFactory.sol`, `createPair()` deploys the pair with constructor params baked in through `initialize()`; however, it sets:
  - `_launchpadLp = launchpadLp` and `_launchpadFeeDistributor = launchpadFeeDistributor` only when `msg.sender == launchpad`.
  - Otherwise it sets both to `address(0)`.

```solidity
function createPair(
    address tokenA,
    address tokenB
) external returns (address pair) { // <--- here everybody can create the pair
    if (tokenA == tokenB) revert("UniswapV2: IDENTICAL_ADDRESSES");
    (address token0, address token1) = tokenA < tokenB
        ? (tokenA, tokenB)
        : (tokenB, tokenA);
    if (token0 == address(0)) revert("UniswapV2: ZERO_ADDRESS");
    if (getPair[token0][token1] != address(0))
        revert("UniswapV2: PAIR_EXISTS"); // single check is sufficient
    bytes memory bytecode = type(GTELaunchpadV2Pair).creationCode;

    (address _launchpadLp, address _launchpadFeeDistributor) = msg.sender ==
        launchpad
        ? (launchpadLp, launchpadFeeDistributor)
        : (address(0), address(0));

    bytes32 salt = keccak256(
        abi.encodePacked(
            token0,
            token1,
            _launchpadLp, // <-- here
            _launchpadFeeDistributor // <-- here
        )
    );
    assembly {
        pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
    }
    IUniswapV2Pair(pair).initialize(
        token0,
        token1,
        _launchpadLp,
        _launchpadFeeDistributor
    );
    getPair[token0][token1] = pair;
    getPair[token1][token0] = pair; // populate mapping in the reverse direction
    allPairs.push(pair);
    emit PairCreated(token0, token1, pair, allPairs.length);
}
```

- Rewards accrual in `@GTELaunchpadV2Pair.sol` is gated on `launchpadFeeDistributor > address(0)` inside `_update()`; only then can `_distributeLaunchpadFees()` be reached and `IDistributor.addRewards()` be called.

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
if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
    // * never overflows, and + overflow is desired
    price0CumulativeLast +=
        uint256(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) *
        timeElapsed;
    price1CumulativeLast +=
        uint256(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) *
        timeElapsed;

    if (launchpadFeeDistributor > address(0)) { // <-- here
        if (totalLaunchpadFee0 | totalLaunchpadFee1 > 0) {
            delete accruedLaunchpadFee0;
            delete accruedLaunchpadFee1;
            _distributeLaunchpadFees(
                totalLaunchpadFee0,
                totalLaunchpadFee1
            );
        }
    }
} else if (
    launchpadFeeDistributor > address(0) && // <-- here
    newLaunchpadFee0 | newLaunchpadFee1 > 0
) {
    accruedLaunchpadFee0 = totalLaunchpadFee0;
    accruedLaunchpadFee1 = totalLaunchpadFee1;
    emit LaunchpadFeesAccrued(newLaunchpadFee0, newLaunchpadFee1);
}
```

- The deployed pair address is derived via `CREATE2` using a salt that includes these two parameters, but the factory’s uniqueness map `getPair[token0][token1]` is keyed only by token addresses.

Root cause:

- `createPair()` is permissionless. Any account can front-run or pre-create the canonical token pair, which will be persisted in `getPair[token0][token1]`. Because `msg.sender != launchpad`, the pair is initialized with `launchpadLp = address(0)` and `launchpadFeeDistributor = address(0)`. From that point on, the legitimate launchpad can no longer create the correct pair (the mapping reports “exists”), and the already-deployed pair will never accrue nor forward launchpad fees because all reward logic is gated behind `launchpadFeeDistributor > address(0)`.

Highest-impact scenario:

- An attacker pre-creates the pair for a soon-to-graduate token/quote using the factory. Graduation then proceeds (liquidity can still be added to the attacker-created pair), trading functions normally, but the launchpad reward share is permanently disabled:
  - Rewards accrual is skipped in `_update()` because `launchpadFeeDistributor == address(0)`.
  - No mechanism exists to later set `launchpadLp`/`launchpadFeeDistributor` on the pair or to replace the entry in `getPair`. This locks the market into a “no rewards” mode indefinitely without an on-chain recovery path.
    > [!NOTE]
    > While the `CREATE2` salt correctly encodes the launchpad parameters, the factory’s uniqueness check uses only `token0`/`token1`. Therefore, the attacker-occupied mapping entry prevents the correct variant (with the non-zero parameters) from ever being deployed, creating a durable DoS on rewards.

Secondary consequences:

- The fee-share math in `@GTELaunchpadV2Pair.sol::_getLaunchpadFees()` references `launchpadLp`’s LP balance; with `launchpadLp == address(0)`, the share is effectively non-functional for program accrual even if other gates were bypassed.
- Operationally, `@Distributor.sol` can create pools and track stakes, but pairs will never call `addRewards()`; program incentives are silently starved.

## Proof of Concept

Add this file inside [`PoCLaunchpad.t.sol`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/test/c4-poc/PoCLaunchpad.t.sol) and run `forge test --match-test test_submissionValidity -vvv`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {LaunchpadTestBase} from "./LaunchpadTestBase.sol";
import {GTELaunchpadV2PairFactory} from "contracts/launchpad/uniswap/GTELaunchpadV2PairFactory.sol";
import {GTELaunchpadV2Pair} from "contracts/launchpad/uniswap/GTELaunchpadV2Pair.sol";

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
        // Deploy the launchpad V2 pair factory
        GTELaunchpadV2PairFactory v2Factory = new GTELaunchpadV2PairFactory(
            owner,
            address(launchpad),
            address(launchpadLPVault),
            address(distributor)
        );

        // Create the pair as a normal user (not the launchpad)
        vm.prank(user);
        address pairAddr = v2Factory.createPair(token, address(quoteToken));

        // Inspect the deployed pair
        GTELaunchpadV2Pair pair = GTELaunchpadV2Pair(pairAddr);

        // Assert rewards params are zeroed when created by non-launchpad
        assertEq(pair.launchpadLp(), address(0), "launchpadLp should be zero");
        assertEq(
            pair.launchpadFeeDistributor(),
            address(0),
            "launchpadFeeDistributor should be zero"
        );

        // Log values for visibility
        emit log_named_address("pair", pairAddr);
        emit log_named_address("launchpadLp", pair.launchpadLp());
        emit log_named_address(
            "launchpadFeeDistributor",
            pair.launchpadFeeDistributor()
        );
    }
}
```

Logs:

```
Ran 1 test for test/c4-poc/PoCLaunchpad.t.sol:PoCLaunchpad
[PASS] test_submissionValidity() (gas: 4945119)
Logs:
  pair: 0x466a6f2FB8D893c69Eb2dCC5FC4122FC734AADBe
  launchpadLp: 0x0000000000000000000000000000000000000000
  launchpadFeeDistributor: 0x0000000000000000000000000000000000000000
```

## Recommended mitigation steps

- Restrict pair creation to the launchpad.
- Additionally (or alternatively) add a one-time activation path to remediate already-created pairs that have zeroed parameters.

1. Gate factory creation to the launchpad only:

```solidity
function createPair(address tokenA, address tokenB) external returns (address pair) {
    if (msg.sender != launchpad) revert("GTEUniV2: FORBIDDEN");
    ...
}
```

2. Add a one-time activation on the pair, callable by the factory/launchpad, only when both params are zero:

```solidity
contract GTELaunchpadV2Pair {
    address public launchpadLp;
    address public launchpadFeeDistributor;
    address public factory;

    function activateRewards(address _lp, address _distributor) external {
        if (msg.sender != factory) revert("FORBIDDEN");
        if (launchpadLp != address(0) || launchpadFeeDistributor != address(0)) revert("ALREADY_SET");
        if (_lp == address(0) || _distributor == address(0)) revert("BAD_PARAMS");

        launchpadLp = _lp;
        launchpadFeeDistributor = _distributor;
    }
}
```
