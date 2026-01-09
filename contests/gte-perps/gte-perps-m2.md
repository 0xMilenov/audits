# Updating the LP vault address desynchronizes pairs and permanently disables launchpad rewards for new markets

The launch flow coordinates three components: the `Launchpad` contract mints and locks initial liquidity at graduation via `addLiquidity()` on a Uniswap V2 router; the `GTELaunchpadV2PairFactory` deploys pairs and writes launchpad-specific configuration into each pair at creation; the `GTELaunchpadV2Pair` accrues and forwards a portion of swap fees to the launchpad’s distributor based on the LP balance held by the launchpad’s vault.

- In [`@Launchpad._createPairAndSwapRemaining`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/Launchpad.sol#L658-L729), liquidity minted at graduation is sent to `launchpadLPVault` (the `to` of `addLiquidity()`), and the address can be changed/updated post-deploy via [`updateLaunchpadLPVault()`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/Launchpad.sol#L578-L590).

```solidity
function _createPairAndSwapRemaining(
    address token,
    IUniswapV2Pair pair,
    LaunchData memory data,
    uint256 remainingBase,
    uint256 remainingQuote,
    address recipient
) internal returns (uint256 additionalQuoteUsed) {
...
...
uniV2Router.addLiquidity({
    tokenA: token,
    tokenB: address(data.quote),
    amountADesired: tokensToLock,
    amountBDesired: quoteToLock,
    amountAMin: 0,
    amountBMin: 0,
    to: address(launchpadLPVault), // <-- here
    deadline: block.timestamp
});
```

```solidity
function updateLaunchpadLPVault(
    address newLaunchpadLPVault
) external onlyOwner {
    launchpadLPVault = LaunchpadLPVault(newLaunchpadLPVault);
}
```

- In [`@GTELaunchpadV2PairFactory.sol`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/uniswap/GTELaunchpadV2PairFactory.sol), the constructor stores `launchpadLp` and `launchpadFeeDistributor` as **constructor-time constants**. On `createPair()`, the factory embeds these into the pair (via `initialize()`).

```solidity
contract GTELaunchpadV2PairFactory is IUniswapV2Factory {
...
...
address immutable launchpadLp;
address immutable launchpadFeeDistributor;
...
...
function createPair(
        address tokenA,
        address tokenB
    ) external returns (address pair) {
...
...
IUniswapV2Pair(pair).initialize(
    token0,
    token1,
    _launchpadLp,
    _launchpadFeeDistributor
);
```

- In `@GTELaunchpadV2Pair.sol`, fee accounting uses `balanceOf(launchpadLp)` (plus `MINIMUM_LIQUIDITY`) inside [`_getLaunchpadFees()`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/uniswap/GTELaunchpadV2Pair.sol#L380-L404) to compute the launchpad’s fee share, and [`_distributeLaunchpadFees()`](https://github.com/code-423n4/2025-08-gte-perps/blob/f43e1eedb65e7e0327cfaf4d7608a37d85d2fae7/contracts/launchpad/uniswap/GTELaunchpadV2Pair.sol#L406-L425) forwards those fees to `launchpadFeeDistributor`.

```solidity
function swap(
        uint256 amount0Out,
        uint256 amount1Out,
        address to,
        bytes calldata data
    ) external lock {
...
...
(
    uint112 launchpadFee0,
    uint112 launchpadFee1
) = launchpadFeeDistributor > address(0) && rewardsPoolActive > 0
        ? _getLaunchpadFees(amount0In, amount1In) // <-- here
        : (uint112(0), uint112(0));

_update(
    balance0,
    balance1,
    _reserve0,
    _reserve1,
    launchpadFee0,
    launchpadFee1
);
```

```solidity
function _getLaunchpadFees(
    uint256 amount0In,
    uint256 amount1In
) internal view returns (uint112 fee0, uint112 fee1) {

    uint256 totalLpBal = this.totalSupply();
    uint256 launchpadLpBal = this.balanceOf(launchpadLp) + // <-- this will be always 0, because is pointing to the old lp address
        MINIMUM_LIQUIDITY;

    if (amount0In > 0)
        fee0 = uint112(
            amount0In.mul(REWARDS_FEE_SHARE).mul(launchpadLpBal) /
                (totalLpBal * 1000)
        );
    if (amount1In > 0)
        fee1 = uint112(
            amount1In.mul(REWARDS_FEE_SHARE).mul(launchpadLpBal) /
                (totalLpBal * 1000)
        );

    return (fee0, fee1);
}
```

The root cause is that `updateLaunchpadLPVault()` in `Launchpad` changes where new LP tokens are minted, but **the factory and pairs continue to reference the old `launchpadLp`**. New pairs created after the update will cache the **old vault address** because the factory’s `launchpadLp` is fixed at construction and written into each pair at `initialize()`. As a result, pairs will measure `balanceOf(launchpadLp)` on the stale address, while **all LP tokens are minted to the new vault**.

Highest impact scenario:

- For every market that graduates after the LP vault address update, `GTELaunchpadV2Pair._getLaunchpadFees()` reads `balanceOf(launchpadLp)` on the old address, which is zero, so **the computed fee share is zero**. No rewards accrue and nothing is distributed to the `distributor`, effectively DoSing the protocol’s fee revenue for all subsequently graduated markets or with other words all fees will be lost.
- This persists indefinitely because pairs cache `launchpadLp` at initialization; the factory fields are also fixed, so subsequent markets remain misconfigured unless the factory is redeployed and the `Launchpad` is redeployed too.

## Proof of Concept

To be able to reproduce the whole flow we need to improve the `LaunchpadTestBase.sol` file and the `MockUniV2Router.sol` file, implementing a deploy of a real `GTELaunchpadV2PairFactory` and properly playing with it.

Please add this file inside `LaunchpadTestBase.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Distributor} from "contracts/launchpad/Distributor.sol";
import {Launchpad} from "contracts/launchpad/Launchpad.sol";
import {ILaunchpad} from "contracts/launchpad/interfaces/ILaunchpad.sol";
import {SimpleBondingCurve} from "contracts/launchpad/BondingCurves/SimpleBondingCurve.sol";
import {IBondingCurveMinimal} from "contracts/launchpad/BondingCurves/IBondingCurveMinimal.sol";
import {LaunchToken} from "contracts/launchpad/LaunchToken.sol";
import {LaunchpadLPVault} from "contracts/launchpad/LaunchpadLPVault.sol";
import {IDistributor} from "contracts/launchpad/interfaces/IDistributor.sol";

import {ERC1967Factory} from "@solady/utils/ERC1967Factory.sol";

import {ERC20Harness} from "../harnesses/ERC20Harness.sol";

import {MockUniV2Router} from "../mocks/MockUniV2Router.sol";

import {FixedPointMathLib} from "@solady/utils/FixedPointMathLib.sol";
import {ICLOBManager} from "contracts/clob/ICLOBManager.sol";
import {IOperatorPanel} from "contracts/utils/interfaces/IOperatorPanel.sol";

import {GTELaunchpadV2PairFactory} from "contracts/launchpad/uniswap/GTELaunchpadV2PairFactory.sol";

import "forge-std/Test.sol";

// Implements the LaunchpadTest setup
contract LaunchpadTestBase is Test {
    using FixedPointMathLib for uint256;

    ERC1967Factory factory;
    Launchpad launchpad;
    address distributor;
    IBondingCurveMinimal curve;
    LaunchpadLPVault launchpadLPVault;

    ERC20Harness quoteToken;
    MockUniV2Router uniV2Router;

    address owner = makeAddr("owner");
    address user = makeAddr("user");
    address dev = makeAddr("dev");

    uint256 constant MIN_BASE_AMOUNT = 100_000_000;

    address token;

    uint256 BONDING_SUPPLY;
    uint256 TOTAL_SUPPLY;

    function setUp() public virtual {
        quoteToken = new ERC20Harness("Quote", "QTE");

        factory = new ERC1967Factory();

        // Router will be wired to a real GTELaunchpadV2PairFactory below

        bytes32 launchpadSalt = bytes32(
            abi.encode("GTE.V1.TESTNET.LAUNCHPAD", owner)
        );

        launchpad = Launchpad(
            factory.predictDeterministicAddress(launchpadSalt)
        );

        address c_logic = address(new SimpleBondingCurve(address(launchpad)));
        address v_logic = address(new LaunchpadLPVault());

        curve = SimpleBondingCurve(factory.deploy(address(c_logic), owner));
        launchpadLPVault = LaunchpadLPVault(
            factory.deploy(address(v_logic), owner)
        );

        address clobManager = makeAddr("clob manager");
        address operatorAddr = makeAddr("operator");
        vm.mockCall(
            operatorAddr,
            abi.encodeWithSelector(
                IOperatorPanel.getOperatorRoleApprovals.selector,
                user,
                address(0)
            ),
            abi.encode(0)
        );

        distributor = address(new Distributor());
        Distributor(distributor).initialize(address(launchpad));

        // Deploy real GTE Pair Factory and wire the router to it
        GTELaunchpadV2PairFactory realFactory = new GTELaunchpadV2PairFactory({
            _feeToSetter: owner,
            _launchpad: address(launchpad),
            _launchpadLp: address(launchpadLPVault),
            _launchpadFeeDistributor: distributor
        });
        uniV2Router = new MockUniV2Router(address(realFactory));

        address l_logic = address(
            new Launchpad(
                address(uniV2Router),
                address(0), // TODO - here we are missing the router address!
                clobManager,
                operatorAddr,
                distributor
            )
        );

        vm.prank(owner);
        Launchpad(
            factory.deployDeterministicAndCall({
                implementation: l_logic,
                admin: owner,
                salt: launchpadSalt,
                data: abi.encodeCall(
                    Launchpad.initialize,
                    (
                        owner,
                        address(quoteToken),
                        address(curve),
                        address(launchpadLPVault),
                        abi.encode(200_000_000 ether, 10 ether)
                    )
                )
            })
        );

        token = _launchToken();

        BONDING_SUPPLY = curve.bondingSupply(token);
        TOTAL_SUPPLY = curve.totalSupply(token);

        vm.startPrank(user);
        quoteToken.approve(address(launchpad), type(uint256).max);
        ERC20Harness(token).approve(address(launchpad), type(uint256).max);
        vm.stopPrank();
    }

    function _launchToken() internal returns (address) {
        uint256 fee = launchpad.launchFee();
        deal(dev, 30 ether);

        vm.prank(dev);
        return
            launchpad.launch{value: fee}(
                "TestToken",
                "TST",
                "https://testtoken.com"
            );
    }
}
```

Now please add this file inside `MockUniV2Router.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {IUniswapV2RouterMinimal} from "contracts/launchpad/interfaces/IUniswapV2RouterMinimal.sol";
import {IUniswapV2FactoryMinimal} from "contracts/launchpad/interfaces/IUniswapV2FactoryMinimal.sol";
import {IUniswapV2Pair} from "contracts/launchpad/interfaces/IUniswapV2Pair.sol";
import {MockERC20} from "forge-std/mocks/MockERC20.sol";
import "test/harnesses/ERC20Harness.sol";
import "@solady/utils/SafeTransferLib.sol";
import {LaunchToken} from "contracts/launchpad/LaunchToken.sol";

// import "forge-std/console.sol";
import "forge-std/Test.sol";

contract MockUniV2Router is IUniswapV2RouterMinimal, Test {
    using SafeTransferLib for address;

    bool public constant IS_SCRIPT = true;
    address private immutable _factory;

    constructor(address factory_) {
        _factory = factory_;
    }

    function factory() external view returns (address) {
        return _factory;
    }

    function addLiquidity(
        address tokenA,
        address tokenB,
        uint256 amountADesired,
        uint256 amountBDesired,
        uint256,
        uint256,
        address to,
        uint256 deadline
    ) external returns (uint256 amountA, uint256 amountB, uint256 liquidity) {
        if (block.timestamp > deadline) revert("expired");

        address pair = IUniswapV2FactoryMinimal(_factory).getPair(
            tokenA,
            tokenB
        );
        require(pair != address(0), "pair not created");

        // Pull tokens from the caller (Launchpad) into the pair and mint LP to `to`
        tokenA.safeTransferFrom(msg.sender, pair, amountADesired);
        tokenB.safeTransferFrom(msg.sender, pair, amountBDesired);

        amountA = amountADesired;
        amountB = amountBDesired;
        liquidity = IUniswapV2Pair(pair).mint(to);
    }

    function swapExactTokensForTokens(
        uint256 amountIn,
        uint256 /*amountOutMin*/,
        address[] calldata path,
        address to,
        uint256 /*deadline*/
    ) external returns (uint256[] memory amounts) {
        path[0].safeTransferFrom(msg.sender, address(this), amountIn);

        amounts = getAmountsOut(amountIn, path);

        address target = path[path.length - 1];

        uint256 tokens = amounts[amounts.length - 1];

        try ERC20Harness(target).mint(to, tokens) {} catch {
            vm.prank(msg.sender);
            try LaunchToken(target).mint(tokens) {} catch {
                revert(
                    "cant mint mock token for swap. is not an ERC20 harness or LaunchToken"
                );
            }
        }
    }

    function swapTokensForExactTokens(
        uint256 amountOut,
        uint256,
        address[] calldata path,
        address to,
        uint256
    ) external returns (uint256[] memory amounts) {
        uint256 amountIn = getAmountIn(amountOut, 0, 0);

        path[0].safeTransferFrom(msg.sender, address(this), amountIn);

        address target = path[path.length - 1];

        amounts = new uint256[](2);
        amounts[0] = amountIn;
        amounts[1] = amountOut;

        try ERC20Harness(target).mint(to, amountOut) {} catch {
            vm.prank(msg.sender);
            try LaunchToken(target).mint(amountOut) {} catch {
                revert(
                    "cant mint mock token for swap. is not an ERC20 harness or LaunchToken"
                );
            }
        }
    }

    function getAmountIn(
        uint256 amountOut,
        uint256,
        uint256
    ) public pure returns (uint256 amountIn) {
        amountIn = amountOut - (amountIn * 1 ether) / 0.5 ether;
    }

    function getAmountsOut(
        uint256 amountIn,
        address[] memory /*path*/
    ) public pure returns (uint256[] memory amounts) {
        amounts = new uint256[](2);
        amounts[0] = amountIn;
        amounts[1] = amountIn + (amountIn * 0.5 ether) / 1 ether;
    }
}

```

Now please update the `PoCLaunchpad.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {LaunchpadTestBase} from "./LaunchpadTestBase.sol";
import {LaunchpadLPVault} from "contracts/launchpad/LaunchpadLPVault.sol";
import {ILaunchpad} from "contracts/launchpad/interfaces/ILaunchpad.sol";
import {IUniswapV2FactoryMinimal} from "contracts/launchpad/interfaces/IUniswapV2FactoryMinimal.sol";
import {IUniswapV2Pair} from "contracts/launchpad/interfaces/IUniswapV2Pair.sol";
import {IGTELaunchpadV2Pair} from "contracts/launchpad/uniswap/interfaces/IGTELaunchpadV2Pair.sol";

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
        LaunchpadLPVault launchpadLPVault;

        // Log current LP vault address from Launchpad
        address oldVault = address(launchpad.launchpadLPVault());
        emit log_named_address("old launchpadLPVault address", oldVault);

        // Deploy a new LaunchpadLPVault via ERC1967Factory
        address v_logic = address(new LaunchpadLPVault());
        LaunchpadLPVault newVault = LaunchpadLPVault(
            factory.deploy(address(v_logic), owner)
        );

        // Update Launchpad's LP vault as owner
        vm.prank(owner);
        launchpad.updateLaunchpadLPVault(address(newVault));

        // Log updated LP vault address and assert change
        address updatedVault = address(launchpad.launchpadLPVault());
        emit log_named_address(
            "updated launchpadLPVault address",
            updatedVault
        );

        assertTrue(
            updatedVault != oldVault,
            "LP vault address should be updated"
        );

        // Verify no pair exists before graduation
        address factoryAddr = uniV2Router.factory();
        address prePair = IUniswapV2FactoryMinimal(factoryAddr).getPair(
            token,
            address(quoteToken)
        );
        emit log_named_address("pair address before graduation", prePair);
        assertEq(
            prePair,
            address(0),
            "pair should not exist before graduation"
        );

        // Buy full bonding supply to trigger graduation
        uint256 bondedBase = curve.bondingSupply(token);
        uint256 bondedQuote = curve.quoteQuoteForBase(token, bondedBase, true);

        // Fund user and execute the graduating buy
        quoteToken.mint(user, bondedQuote + 50 ether);
        vm.prank(user);
        (uint256 baseActual, uint256 quoteActual) = launchpad.buy(
            ILaunchpad.BuyData({
                account: user,
                token: token,
                recipient: user,
                amountOutBase: bondedBase,
                maxAmountInQuote: bondedQuote + 50 ether
            })
        );
        assertEq(baseActual, bondedBase, "did not purchase full bonded base");

        // Verify pair is created by the real factory and LP tokens minted to vault
        address pairAddr = IUniswapV2FactoryMinimal(factoryAddr).getPair(
            token,
            address(quoteToken)
        );
        emit log_named_address("pair address after graduation", pairAddr);
        assertTrue(pairAddr != address(0), "pair was not created");

        uint256 lpBal = IUniswapV2Pair(pairAddr).balanceOf(
            address(launchpad.launchpadLPVault())
        );

        emit log_named_uint(
            "the LP balance of the NEW LP vault address",
            lpBal
        );
        assertGt(lpBal, 0, "no LP tokens minted to vault");

        // Log the pair's configured launchpadLp and its LP balance
        address pairLaunchpadLp = IGTELaunchpadV2Pair(pairAddr).launchpadLp();
        emit log_named_address(
            "the launchpadLp address INSIDE THE PAIR pointing to the old one",
            pairLaunchpadLp
        );
        uint256 pairLaunchpadLpBal = IUniswapV2Pair(pairAddr).balanceOf(
            pairLaunchpadLp
        );
        emit log_named_uint(
            "the LP balance in pair.launchpadLp",
            pairLaunchpadLpBal
        );
    }
}

```

And run `forge test --match-test test_submissionValidity -vvv`

Log results:

```
Ran 1 test for test/c4-poc/PoCLaunchpad.t.sol:PoCLaunchpad
[PASS] test_submissionValidity() (gas: 3416947)
Logs:
  old launchpadLPVault address: 0x8d2C17FAd02B7bb64139109c6533b7C2b9CADb81
  updated launchpadLPVault address: 0x76006C4471fb6aDd17728e9c9c8B67d5AF06cDA0
  pair address before graduation: 0x0000000000000000000000000000000000000000
  pair address after graduation: 0x5bc640Ea6e0e37E7f47f2cF286299599961B5cF9
  the LP balance of the NEW LP vault address: 89442719099991587855366
  the launchpadLp address INSIDE THE PAIR pointing to the old one: 0x8d2C17FAd02B7bb64139109c6533b7C2b9CADb81
  the LP balance in pair.launchpadLp: 0
```

## Recommended mitigation steps

Prioritize one of the following designs:

- Make the vault immutable:

  - Remove or hard-freeze `updateLaunchpadLPVault()` after initialization (or after the first launch), ensuring the vault used as `addLiquidity()` recipient always matches the address embedded in pairs via the factory.

- If configurability is required, make pair and factory config rotatable:

  - Replace factory constructor-time constants with storage variables and add controlled setters (e.g., `setLaunchpadConfig(launchpadLp, launchpadFeeDistributor)` restricted to a governance role).
  - In `GTELaunchpadV2Pair`, add a guarded `setLaunchpadLp()` (and optionally `setLaunchpadFeeDistributor()`) callable by the factory or a dedicated role, to update existing pairs when the vault/distributor rotates.
  - Update `Launchpad.updateLaunchpadLPVault()` to also update factory config for future pairs and to invoke the per-pair setter for any already created pairs of active markets (if necessary).

- Alternative (more robust, slightly more gas):
  - In pairs, store only the `launchpad` address and query the current `launchpadLPVault()` and `distributor()` from `Launchpad` at use-time when computing/distributing fees, eliminating configuration drift between components.
