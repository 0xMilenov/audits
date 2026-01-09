## QUALITY ASSURANCE REPORT

### Findings Overview

| Label | Description                                                                                           | Severity |
| ----- | ----------------------------------------------------------------------------------------------------- | -------- | --- |
| L-01  | Missing validation of constructor parameters enables unsafe jackpot configuration in `Jackpot.sol`    | Low      |     |
| L-02  | Missing jackpot lock check allows entropy provider change during locked draw in Jackpot               | Low      |
| L-03  | Missing `noEmergencyMode` enables payout and referral withdrawals during incidents in `Jackpot.sol`   | Low      |
| L-04  | Missing bounds validation for governance pool cap can misconfigure LP cap calculation in `Jackpot`    | Low      |
| L-05  | Missing validation for initial drawing time can schedule invalid rounds in `Jackpot`                  | Low      |
| L-06  | Missing referrer/recipient distinctness check enables self-referral and incentive gaming in `Jackpot` | Low      |
| L-07  | Missing reentrancy guard on state-changing function can enable reentry attacks in `Jackpot`           | Low      |
| L-08  | Missing reentrancy guard on ticket-claim path in `JackpotBridgeManager`                               | Low      |
| L-09  | Missing `runJackpot()` in `IJackpot` blocks keepers/managers from triggering jackpot via interface    | Low      |
| L-10  | Unused custom errors in `JackpotErrors.sol` reduce clarity and risk missing validations               | Low      |
| L-11  | Stale bridge-side ownership can make batched claims revert in `JackpotBridgeManager.sol`              | Low      |

### [L-01] Missing validation of constructor parameters enables unsafe jackpot configuration in `Jackpot.sol`

The [`constructor()`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/Jackpot.sol#L296-L325) assigns core configuration values directly from unvalidated inputs.

This can lead to undesirable or unsafe behavior, such as fee/share parameters exceeding `PRECISE_UNIT` or violating aggregate constraints causing >100% allocations, mis-sized `drawingDurationInSeconds` enabling overlapping or trivially short drawing windows, an excessively low `protocolFeeThreshold` skimming LP profits on every round, or too‑low `entropyBaseGasLimit` increasing callback failure risk during drawings.
Downstream functions (e.g., `initialize()`, ticket purchasing, payout calculations, draw settlement, and LP accounting) may implicitly assume sane ranges and suffer from distorted economics, reverts, or operational failures when initialized with out‑of‑bounds values.

```solidity
constructor(
    uint256 _drawingDurationInSeconds,
    uint8 _normalBallMax,
    uint8 _bonusballMin,
    uint256 _lpEdgeTarget,
    uint256 _reserveRatio,
    uint256 _referralFee,
    uint256 _referralWinShare,
    uint256 _protocolFee,
    uint256 _protocolFeeThreshold,
    uint256 _ticketPrice,
    uint256 _maxReferrers,
    uint32 _entropyBaseGasLimit
) Ownable(msg.sender) {
    drawingDurationInSeconds = _drawingDurationInSeconds;
    normalBallMax = _normalBallMax;
    bonusballMin = _bonusballMin;
    lpEdgeTarget = _lpEdgeTarget;
    reserveRatio = _reserveRatio;
    referralFee = _referralFee;
    referralWinShare = _referralWinShare;
    protocolFee = _protocolFee;
    protocolFeeThreshold = _protocolFeeThreshold;
    ticketPrice = _ticketPrice;
    maxReferrers = _maxReferrers;
    entropyBaseGasLimit = _entropyBaseGasLimit;

    entropyVariableGasLimit = uint32(250000);
    protocolFeeAddress = msg.sender;
}
```

Recommendation: Add explicit validation in `constructor()` using `require` or custom errors (e.g., from `JackpotErrors.sol`) to enforce business‑driven bounds and invariants, such as:

- `drawingDurationInSeconds >= 180`
- Percentages (`lpEdgeTarget`, `reserveRatio`, `referralFee`, `referralWinShare`, `protocolFee`) within `[0, PRECISE_UNIT]` and any aggregate constraints if applicable
- `ticketPrice > 0`, `maxReferrers > 0`
- `entropyBaseGasLimit` within a safe operational range for `entropy` callbacks, etc

**Mirror these checks in ALL admin setter functions to prevent later drift and add unit tests that assert reverts on invalid inputs.**

### [L-02] Missing jackpot lock check allows entropy provider change during locked draw in Jackpot

[`setEntropy()`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/Jackpot.sol#L1193-L1199) updates the `IScaledEntropyProvider` without first asserting that the current drawing is unlocked (e.g., `drawingState[currentDrawingId].jackpotLock == false`). Allowing the randomness provider to change while a draw is locked/in-progress can violate timing and determinism assumptions, risk inconsistent entropy usage, and undermine fairness guarantees for a round that participants consider finalized. Although the setter is owner-only, changing critical randomness infrastructure mid-round is a governance risk and can introduce revert-prone edge cases if the new provider’s state is not aligned.

```solidity
  function setEntropy(IScaledEntropyProvider _entropy) external onlyOwner {
    if (_entropy == IScaledEntropyProvider(address(0)))
      revert JackpotErrors.ZeroAddress();
    IScaledEntropyProvider oldEntropy = entropy;
    entropy = _entropy;

    emit EntropyUpdated(
      currentDrawingId,
      address(oldEntropy),
      address(_entropy)
    );
  }
```

Recommendation:

- Before updating, require the current drawing to be unlocked and revert with a dedicated error (e.g., `JackpotLocked()`).
- Optionally also block updates when there is any pending randomness request or in-flight settlement tied to the old provider.

### [L-03] Missing `noEmergencyMode` enables payout and referral withdrawals during incidents in `Jackpot.sol`

[`claimWinnings`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/Jackpot.sol#L418-L452) and [`claimReferralFees()`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/Jackpot.sol#L618-L624) functions lack an emergency/paused-state restriction. During an incident (e.g., manipulated winning conditions or abusive referral accrual), operators cannot promptly halt user-initiated withdrawals. This allows claims and referral fee withdrawals to continue, increasing the risk of accelerated draining of `usdc`. The referral path reads and zeroes `referralFees[msg.sender]` before transferring `usdc`, and claim payout entry points similarly finalize user-facing transfers, but neither is gated by an emergency flag.

```solidity
function claimWinnings(
  uint256[] memory _userTicketIds
) external nonReentrant {
...
...
 function claimReferralFees() external nonReentrant {
```

Recommendation: Add the `noEmergencyMode` modifier.

### [L-04] Missing bounds validation for governance pool cap can misconfigure LP cap calculation in Jackpot

In [`Jackpot.sol`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/Jackpot.sol#L820-L831), the assignment to `governancePoolCap` from an external input occurs before deriving `lpPoolCap`, but no minimum/maximum or relational validation is performed. An excessively low or overly large `governancePoolCap` can lead to a misconfigured `lpPoolCap`, break expected invariants, or trigger downstream arithmetic/logic edge cases. While this is an admin-controlled parameter, the impact includes incorrect pool sizing and potential revert conditions in flows that rely on these caps.

```solidity
  function initializeLPDeposits(uint256 _governancePoolCap) external onlyOwner {
    if (!initialized) revert JackpotErrors.ContractNotInitialized();
    if (jackpotLPManager.getDrawingAccumulator(0) != 0)
      revert JackpotErrors.LPDepositsAlreadyInitialized();
    if (_governancePoolCap == 0)
      revert JackpotErrors.InvalidGovernancePoolCap();

    // Set governance pool cap first so that it is available for lpPoolCap calculation
    governancePoolCap = _governancePoolCap;

    // Set lpPoolCap and drawingAccumulator to be able to start taking deposits
    jackpotLPManager.initializeLP();
    jackpotLPManager.setLPPoolCap(
      currentDrawingId,
      _calculateLpPoolCap(normalBallMax)
    );
  }
```

Recommendation: Validate the input against explicit bounds and invariants before assignment, for example:

- Enforce `MIN_GOVERNANCE_POOL_CAP <= _governancePoolCap <= MAX_GOVERNANCE_POOL_CAP`.
- Ensure relational constraints, such as `governancePoolCap` being consistent with any total cap and the `lpPoolCap` derivation.

### [L-05] Missing validation for initial drawing time can schedule invalid rounds in Jackpot

`_initialDrawingTime` is passed directly into [`_setNewDrawingState(newLpValue, _initialDrawingTime)`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/Jackpot.sol#L852-L862) without validation. Accepting a zero or past timestamp, an excessively distant future time, or a value misaligned with the protocol’s drawing cadence can cause immediate/retroactive rounds, skipped accrual periods, broken time-based assumptions, or revert-prone logic. Even though this is admin-controlled, misconfiguration can impact round scheduling and participant expectations.

```solidity
function initializeJackpot(uint256 _initialDrawingTime) external onlyOwner {
    if (jackpotLPManager.getDrawingAccumulator(0) == 0)
      revert JackpotErrors.LPDepositsNotInitialized();
    if (currentDrawingId != 0) revert JackpotErrors.JackpotAlreadyInitialized();

    if (jackpotLPManager.getLPDrawingState(0).pendingDeposits == 0)
      revert JackpotErrors.NoLPDeposits();

    allowTicketPurchases = true;
    (uint256 newLpValue, ) = jackpotLPManager.processDrawingSettlement(
      0,
      0,
      0,
      0
    ); // Drawing 0 and no winnings or lp earnings
    _setNewDrawingState(newLpValue, _initialDrawingTime);
  }
```

Recommendation: Add explicit bounds and relational checks before calling `_setNewDrawingState()`, such as:

- Ensure `_initialDrawingTime > block.timestamp`.
- Constrain the value within a reasonable future horizon (e.g., not more than a configured maximum).

### [L-06] Missing referrer/recipient distinctness check enables self-referral and incentive gaming in Jackpot

The referral flow in [`Jackpot.sol`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/Jackpot.sol#L365-L393) does not validate that `referrer` and `recipient` are different addresses. Without a guard such as `referrer != recipient`, a user can self-refer, collecting referral rewards on their own activity. This undermines the intended incentive model, can dilute rewards for legitimate referrals, and may skew accounting/metrics tied to referral performance. Even though users could sybil with multiple wallets, preventing trivial self-referral by the same address reduces abuse and improves data quality.

```solidity
function buyTickets(
    Ticket[] memory _tickets,
    address _recipient,
    address[] memory _referrers,
    uint256[] memory _referralSplit,
    bytes32 _source
) external nonReentrant noEmergencyMode returns (uint256[] memory ticketIds) {
```

Recommendation:

- Enforce `referrer != recipient` at the point of referral attribution.

### [L-07] Missing reentrancy guard on state-changing function can enable reentry attacks in Jackpot

A state-changing function in [`Jackpot.sol`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/Jackpot.sol#L506-L511) performs external calls/asset transfers without a reentrancy guard.

```solidity
  function initiateWithdraw(
    uint256 _amountToWithdrawInShares
  ) external noEmergencyMode {
```

Recommendation:

- Add the `nonReentrant` modifier.

### [L-08] Missing reentrancy guard on ticket-claim path in `JackpotBridgeManager`

[`claimTickets()`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/JackpotBridgeManager.sol#L268-L278) is an external function that ultimately calls `_updateTicketOwnership()`, which performs `safeTransferFrom` on the ERC-721 `jackpotTicketNFT`. The function currently executes without a `nonReentrant` guard, contrary to the documented security expectations for this path. While the current order of operations deletes ownership from the `ticketOwner` mapping before each transfer—reducing the likelihood of double-claims—reentrancy remains possible into other external entry points, expanding the attack surface and violating the intended invariant that this flow be reentrancy-protected.

```solidity
 function claimTickets(
    uint256[] memory _ticketIds,
    address _recipient,
    bytes memory _signature
  ) external {
    if (_recipient == address(0)) revert JackpotErrors.ZeroAddress();
    if (_recipient == address(this)) revert JackpotErrors.InvalidRecipient();

    bytes32 eipHash = createClaimTicketEIP712Hash(_ticketIds, _recipient);
    address signer = ECDSA.recover(eipHash, _signature);

    _validateTicketOwnership(_ticketIds, signer);

    _updateTicketOwnership(_ticketIds, _recipient);
  }
```

Recommendation: Add the `nonReentrant` modifier to `claimTickets()`.

### [L-09] Missing `runJackpot()` in `IJackpot` blocks keepers/managers from triggering jackpot via interface

[`IJackpot`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/interfaces/IJackpot.sol#L1-L30) does not declare `runJackpot()`. As a result, any on-chain keeper or manager compiled against `IJackpot` cannot invoke `runJackpot()` through the interface, forcing dependencies on the concrete `Jackpot` contract or low-level calls. This weakens abstraction boundaries, complicates integrations and testing, and may prevent scheduled jackpot execution in deployments that rely on interface-based coupling. Impact is low because it is an integration/ergonomics issue rather than a direct security risk.

Recommendation: Expose `runJackpot()` in `IJackpot` with the exact signature used by the implementation, preserving existing access controls and invariants.

### [L-10] Unused custom errors in `JackpotErrors.sol` reduce clarity and risk missing validations

[`contracts/lib/JackpotErrors.sol`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/lib/JackpotErrors.sol#L15-L73) defines several custom errors that appear unused across the system:

```solidity
error TicketAlreadyMinted();
error EntropyAlreadyCalled();
error InvalidNormalBallMax();
```

Recommendation: Either remove these unused errors to keep the error catalog accurate, or implement the corresponding validations that revert with them where appropriate.

### [L-11] Stale bridge-side ownership can make batched claims revert in `JackpotBridgeManager.sol`

`JackpotBridgeManager.sol` tracks logical ownership using `ticketOwner[ticketId]`, which is used by `_validateTicketOwnership()` inside `claimWinnings()` to authorize batches. In `Jackpot.sol::claimWinnings()`, each NFT is the source of truth: it checks `ownerOf(ticketId) == msg.sender` and then burns the ticket.

When a ticket was already claimed and burned on `Jackpot.sol`, **the bridge’s `ticketOwner[ticketId]` entry is not cleared**.

As a result, `_validateTicketOwnership()` may still accept a burned `ticketId`, but the subsequent call to `Jackpot.sol::claimWinnings()` reverts at the `ownerOf()` check, causing the entire batch to fail. This does not enable theft, but it degrades availability/UX by making batches with any stale `ticketId` revert and wasting gas until off-chain filtering removes burned IDs.

- [`buyTickets()`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/JackpotBridgeManager.sol#L166-L198) mints tickets on `Jackpot.sol` to the bridge address and records ownership in the bridge’s mappings (`userTickets` and `ticketOwner`).

```solidity
function buyTickets(
    IJackpot.Ticket[] memory _tickets,
    address _recipient,
    address[] memory _referrers,
    uint256[] memory _referralSplitBps,
    bytes32 _source
)
    external
    nonReentrant
    returns (uint256[] memory)
{
    ...
    ...
    // Store the tickets in the user's mapping
    UserTickets storage userDrawingTickets = userTickets[_recipient][currentDrawingId];
    uint256 userTicketCount = userDrawingTickets.totalTicketsOwned;
    for (uint256 i = 0; i < _tickets.length; i++) {
        userDrawingTickets.ticketIds[userTicketCount + i] = ticketIds[i];
        ticketOwner[ticketIds[i]] = _recipient; // <-- here
    }
    userDrawingTickets.totalTicketsOwned += _tickets.length;
    ...
    ...
}
```

- [`claimWinnings()`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/JackpotBridgeManager.sol#L225-L244) on `JackpotBridgeManager.sol` validates that the provided `_userTicketIds` are owned by the signer using the **bridge’s `ticketOwner` mapping**, then calls `Jackpot.sol`’s `claimWinnings()` to burn the NFTs and receive USDC for bridging.

```solidity
function claimWinnings(uint256[] memory _userTicketIds, RelayTxData memory _bridgeDetails, bytes memory _signature) external nonReentrant {
    if (_userTicketIds.length == 0) revert JackpotErrors.NoTicketsToClaim();

    bytes32 eipHash = createClaimWinningsEIP712Hash(_userTicketIds, _bridgeDetails);
    address signer = ECDSA.recover(eipHash, _signature);

    _validateTicketOwnership(_userTicketIds, signer);

    uint256 preUSDCBalance = usdc.balanceOf(address(this));
    jackpot.claimWinnings(_userTicketIds);
    uint256 postUSDCBalance = usdc.balanceOf(address(this));
    uint256 claimedAmount = postUSDCBalance - preUSDCBalance;

    if (claimedAmount == 0) revert InvalidClaimedAmount();

    // @audit missing 'delete ticketOwner[ticketId];

    _bridgeFunds(_bridgeDetails, claimedAmount);

    emit WinningsClaimed(signer, _bridgeDetails.to, _userTicketIds, claimedAmount);
}
```

Recommendation: after a successful `claimWinnings()` on `JackpotBridgeManager.sol`, iterate `_userTicketIds` and `delete ticketOwner[ticketId]` to keep the bridge’s view in sync with the canonical state. Optionally, ensure any helper that enumerates user tickets avoids returning burned/transferred IDs or relies on on-chain `ownerOf()` checks when assembling batches.
