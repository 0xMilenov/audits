# All winners will lose payouts or face DoS after payout calculator upgrade

The jackpot system in `Jackpot.sol` orchestrates drawings, settlement, and claims. During settlement, it computes total winner payouts via the currently configured `payoutCalculator` by calling [`calculateAndStoreDrawingUserWinnings()`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/Jackpot.sol#L1633-L1640). Later, individual claims are paid using per-tier payouts fetched from [`payoutCalculator.getTierPayout()`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/Jackpot.sol#L432-L433) for the historical `drawingId`.

The concrete components and interactions:

- `Jackpot.scaledEntropyCallback()` finalizes a drawing and computes user winnings through `payoutCalculator.calculateAndStoreDrawingUserWinnings()`, then transitions to the next drawing.

```solidity
function _calculateDrawingUserWinnings(
    DrawingState storage _currentDrawingState,
    uint256[][] memory _unPackedWinningNumbers
)
    internal
    returns(uint256 winningNumbers, uint256 drawingUserWinnings)
{
    ...
    ...
    drawingUserWinnings = payoutCalculator.calculateAndStoreDrawingUserWinnings(
        currentDrawingId,
        _currentDrawingState.prizePool,
        _currentDrawingState.ballMax,
        _currentDrawingState.bonusballMax,
        uniqueResult,
        dupResult
    );
```

- [`GuaranteedMinimumPayoutCalculator`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/GuaranteedMinimumPayoutCalculator.sol#L412-L438) stores per-drawing tier payouts internally in `tierPayouts[drawingId][tierId]`. Its `getTierPayout()` returns whatever is stored for that drawing and tier.

```solidity
mapping(uint256=>mapping(uint256 => uint256)) tierPayouts;
...
function _calculateAndStoreTierPayouts(
    uint256 _drawingId,
    uint256 _remainingPrizePool,
    uint256 _minPayout,
    uint256[TOTAL_TIER_COUNT] memory _tierWinners,
    uint256[] memory _uniqueResult,
    uint256[] memory _dupResult
)
    internal
    returns(uint256 totalPayout)
{
    DrawingTierInfo storage tierInfo = drawingTierInfo[_drawingId];
    for (uint256 i = 0; i < TOTAL_TIER_COUNT; i++) {
        // If no winners then no payout
        if (_tierWinners[i] != 0) {
            // Calculate the payout for each tier from the (remaining prize pool * weight) / total winning tickets
            //(including LP-owned winning tickets)
            uint256 premiumTierPayoutAmount = _remainingPrizePool * tierInfo.premiumTierWeights[i] / (PRECISE_UNIT * _tierWinners[i]);
            // Add the premium tier payout to the minimum payout if the tier is eligible for the minimum payout
            uint256 tierPayout = tierInfo.minPayoutTiers[i] ? _minPayout + premiumTierPayoutAmount : premiumTierPayoutAmount;
            // Store the payout for the tier in the mapping so it can be queried later and add the total tier payout to the total payout
            tierPayouts[_drawingId][i] = tierPayout; // <-- here
```

- [`Jackpot.claimWinnings()`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/Jackpot.sol#L418-L433) pays users by querying `payoutCalculator.getTierPayout(drawingId, tierId)` at claim time, and immediately transferring USDC to the claimant.

```solidity
function claimWinnings(uint256[] memory _userTicketIds) external nonReentrant {
    if (_userTicketIds.length == 0) revert JackpotErrors.NoTicketsToClaim();

    uint256 totalClaimAmount = 0;
    for (uint256 i = 0; i < _userTicketIds.length; i++) {
        uint256 ticketId = _userTicketIds[i];
        IJackpotTicketNFT.TrackedTicket memory ticketInfo = jackpotNFT.getTicketInfo(ticketId);
        uint256 drawingId = ticketInfo.drawingId;
        if (IERC721(address(jackpotNFT)).ownerOf(ticketId) != msg.sender) revert JackpotErrors.NotTicketOwner();
        if (drawingId >= currentDrawingId) revert JackpotErrors.TicketFromFutureDrawing();

        DrawingState memory winningDrawingState = drawingState[drawingId];
        uint256 tierId = _calculateTicketTierId(ticketInfo.packedTicket, winningDrawingState.winningTicket, winningDrawingState.ballMax);
        jackpotNFT.burnTicket(ticketId);

        uint256 winningAmount = payoutCalculator.getTierPayout(drawingId, tierId); // <-- here will return 0 if update
```

- The protocol allows swapping the `payoutCalculator` at any time via [`setPayoutCalculator()`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/Jackpot.sol#L1157-L1177) in `Jackpot.sol`.

```solidity
function setPayoutCalculator(IPayoutCalculator _payoutCalculator) external onlyOwner {
    if (_payoutCalculator == IPayoutCalculator(address(0))) revert JackpotErrors.ZeroAddress();
    IPayoutCalculator oldPayoutCalculator = payoutCalculator;
    payoutCalculator = _payoutCalculator;

    emit PayoutCalculatorUpdated(currentDrawingId, address(oldPayoutCalculator), address(_payoutCalculator));
}
```

Root cause:

- There is no per-drawing snapshot of which calculator was used to compute/store payouts for that drawing. Claims for past drawings call the “current” `payoutCalculator.getTierPayout()` (whatever the owner most recently set), not the calculator that computed and stored the payouts for that historical drawing.
- Consequently, if the calculator is upgraded after settlement but before users claim, the new calculator will be queried for old drawings. **Because the new calculator does not contain the historical `tierPayouts` for those drawings, `getTierPayout()` can return 0, different amounts, or revert.**

Highest-impact scenario:

- A user wins a large prize (e.g., 1M) in drawing N and delays a bit the claiming.
- Later, the owner upgrades `payoutCalculator` to a new contract that hasn’t migrated historical `tierPayouts` for drawing N because it is impossible.
- When the user claims, `getTierPayout(N, tierId)` returns 0 (or reverts), resulting in:
  - Loss of user funds (e.g, 1M) (paid 0) or
  - DoS of all claims (revert), preventing all winners from receiving payouts.

That said when the owner updates / sets a new payout calculator, all historical winners will immediately lose their stored payouts for all previous drawings (the new calculator has zero historical tierPayouts).
I rate this finding HIGH severity because this would cause large losses if/when an upgrade is performed, and upgrades are very likely to happen over time.

## Recommended mitigation steps

Snapshot the payout calculator per drawing and use that snapshot for all future claims on that drawing:

- On settlement (before incrementing the drawing), persist the calculator used:

```solidity
mapping(uint256 => IPayoutCalculator) public payoutCalculatorForDrawing;
```

- At the end of settlement for the current drawing:

```solidity
payoutCalculatorForDrawing[currentDrawingId] = payoutCalculator;
```

- In `claimWinnings()`, query the snapshot rather than the current global:

```solidity
IPayoutCalculator calc = payoutCalculatorForDrawing[drawingId];
uint256 winningAmount = calc.getTierPayout(drawingId, tierId);
```

## Proof of Concept

Replace the contents of '/test/poc/C4PoC.spec.ts' with the code below and run `yarn test:poc`

```typescript
import { ethers } from "hardhat";
import DeployHelper from "@utils/deploys";

import { getWaffleExpect, getAccounts } from "@utils/test/index";
import { ether, usdc } from "@utils/common";
import { Account } from "@utils/test";

import { PRECISE_UNIT } from "@utils/constants";

import {
  GuaranteedMinimumPayoutCalculator,
  Jackpot,
  JackpotBridgeManager,
  JackpotLPManager,
  JackpotTicketNFT,
  MockDepository,
  ReentrantUSDCMock,
  ScaledEntropyProviderMock,
} from "@utils/contracts";
import {
  Address,
  JackpotSystemFixture,
  RelayTxData,
  Ticket,
} from "@utils/types";
import { deployJackpotSystem } from "@utils/test/jackpotFixture";
import {
  calculatePackedTicket,
  calculateTicketId,
  generateClaimTicketSignature,
  generateClaimWinningsSignature,
} from "@utils/protocolUtils";
import { ADDRESS_ZERO } from "@utils/constants";
import {
  takeSnapshot,
  SnapshotRestorer,
  time,
} from "@nomicfoundation/hardhat-toolbox/network-helpers";

const expect = getWaffleExpect();

describe("C4", () => {
  let owner: Account;
  let buyerOne: Account;
  let buyerTwo: Account;
  let referrerOne: Account;
  let referrerTwo: Account;
  let referrerThree: Account;
  let solver: Account;

  let jackpotSystem: JackpotSystemFixture;
  let jackpot: Jackpot;
  let jackpotNFT: JackpotTicketNFT;
  let jackpotLPManager: JackpotLPManager;
  let payoutCalculator: GuaranteedMinimumPayoutCalculator;
  let usdcMock: ReentrantUSDCMock;
  let entropyProvider: ScaledEntropyProviderMock;
  let snapshot: SnapshotRestorer;
  let jackpotBridgeManager: JackpotBridgeManager;
  let mockDepository: MockDepository;

  beforeEach(async () => {
    [
      owner,
      buyerOne,
      buyerTwo,
      referrerOne,
      referrerTwo,
      referrerThree,
      solver,
    ] = await getAccounts();

    jackpotSystem = await deployJackpotSystem();
    jackpot = jackpotSystem.jackpot;
    jackpotNFT = jackpotSystem.jackpotNFT;
    jackpotLPManager = jackpotSystem.jackpotLPManager;
    payoutCalculator = jackpotSystem.payoutCalculator;
    usdcMock = jackpotSystem.usdcMock;
    entropyProvider = jackpotSystem.entropyProvider;

    await jackpot
      .connect(owner.wallet)
      .initialize(
        usdcMock.getAddress(),
        await jackpotLPManager.getAddress(),
        await jackpotNFT.getAddress(),
        entropyProvider.getAddress(),
        await payoutCalculator.getAddress()
      );

    await jackpot.connect(owner.wallet).initializeLPDeposits(usdc(10000000));

    await usdcMock
      .connect(owner.wallet)
      .approve(jackpot.getAddress(), usdc(1000000));
    await jackpot.connect(owner.wallet).lpDeposit(usdc(1000000));

    await jackpot
      .connect(owner.wallet)
      .initializeJackpot(
        BigInt(await time.latest()) +
          BigInt(jackpotSystem.deploymentParams.drawingDurationInSeconds)
      );

    jackpotBridgeManager =
      await jackpotSystem.deployer.deployJackpotBridgeManager(
        await jackpot.getAddress(),
        await jackpotNFT.getAddress(),
        await usdcMock.getAddress(),
        "MegapotBridgeManager",
        "1.0.0"
      );

    mockDepository = await jackpotSystem.deployer.deployMockDepository(
      await usdcMock.getAddress()
    );

    snapshot = await takeSnapshot();
  });

  beforeEach(async () => {
    await snapshot.restore();
  });

  describe("PoC", async () => {
    it("demonstrates the C4 submission's validity", async () => {
      // Step 1: Setup - Buy tickets and settle a drawing
      // User buys a winning ticket for drawing 1
      const winningTicket: Ticket = {
        normals: [BigInt(1), BigInt(2), BigInt(3), BigInt(4), BigInt(5)],
        bonusball: BigInt(6),
      } as Ticket;

      // Ensure buyerOne has sufficient USDC balance
      const buyerOneBalance = await usdcMock.balanceOf(buyerOne.address);
      if (buyerOneBalance < usdc(10)) {
        await usdcMock
          .connect(owner.wallet)
          .transfer(buyerOne.address, usdc(10));
      }

      await usdcMock
        .connect(buyerOne.wallet)
        .approve(jackpot.getAddress(), usdc(10));

      const tx = await jackpot
        .connect(buyerOne.wallet)
        .buyTickets(
          [winningTicket],
          buyerOne.address,
          [],
          [],
          ethers.encodeBytes32String("test")
        );
      const receipt = await tx.wait();
      // Get ticketIds from the transaction receipt or calculate them
      // Since buyTickets returns the ticketIds, we can get them from events or calculate
      // For simplicity, we'll calculate the ticket ID
      const packedTicket = calculatePackedTicket(
        winningTicket,
        jackpotSystem.deploymentParams.normalBallMax
      );
      const ticketIds: bigint[] = [calculateTicketId(1, 1, packedTicket)];

      // Wait for drawing to be due and settle it
      await time.increase(
        jackpotSystem.deploymentParams.drawingDurationInSeconds
      );
      const drawingState = await jackpot.getDrawingState(1);
      const deploymentParams = jackpotSystem.deploymentParams as any;
      const entropyValue =
        deploymentParams.entropyFee +
        (deploymentParams.entropyBaseGasLimit +
          deploymentParams.entropyVariableGasLimit *
            drawingState.bonusballMax) *
          BigInt(1e7);

      await jackpot.runJackpot({ value: entropyValue });

      // Settle with winning numbers that match the ticket (5 matches + bonusball = tier 11)
      const winningNumbers = [
        [BigInt(1), BigInt(2), BigInt(3), BigInt(4), BigInt(5)],
        [BigInt(6)],
      ];
      await entropyProvider.randomnessCallback(winningNumbers);

      // Step 2: Verify - Check that the original calculator has tier payouts stored
      const drawingId = 1n;
      const tierId = 11n; // 5 matches + bonusball
      const originalPayout = await payoutCalculator.getTierPayout(
        drawingId,
        tierId
      );

      expect(originalPayout).to.be.greaterThan(
        0n,
        "Original calculator should have payout stored"
      );

      // Step 3: Upgrade - Deploy a new calculator and set it
      // The new calculator is a fresh instance with no historical data
      const newPayoutCalculator =
        await jackpotSystem.deployer.deployGuaranteedMinimumPayoutCalculator(
          await jackpot.getAddress(),
          deploymentParams.minimumPayout,
          deploymentParams.premiumTierMinAllocation,
          deploymentParams.minPayoutTiers,
          deploymentParams.premiumTierWeights
        );

      // Verify new calculator has no data for drawing 1
      const newCalculatorPayout = await newPayoutCalculator.getTierPayout(
        drawingId,
        tierId
      );
      expect(newCalculatorPayout).to.equal(
        0n,
        "New calculator should have no payout data for drawing 1"
      );

      // Owner upgrades the payout calculator
      await jackpot
        .connect(owner.wallet)
        .setPayoutCalculator(newPayoutCalculator);

      // Step 4: Demonstrate - User tries to claim winnings from drawing 1
      // The claim will use the new calculator which has no historical data
      const preClaimBalance = await usdcMock.balanceOf(buyerOne.address);

      // Claim winnings - this will call getTierPayout on the NEW calculator
      await jackpot.connect(buyerOne.wallet).claimWinnings(ticketIds);

      // Step 5: Verify the impact - User receives 0 USDC (loss of funds)
      const postClaimBalance = await usdcMock.balanceOf(buyerOne.address);
      const receivedAmount = postClaimBalance - preClaimBalance;

      // Verify the original calculator still has the payout data (proving it's not a data loss issue)
      const originalPayoutAfterUpgrade = await payoutCalculator.getTierPayout(
        drawingId,
        tierId
      );
      expect(originalPayoutAfterUpgrade).to.equal(
        originalPayout,
        "Original calculator still has the data"
      );

      // Log the impact: what should have been received vs what was actually received
      console.log("\n=== VULNERABILITY IMPACT ===");
      console.log(
        `Expected payout (from original calculator): ${originalPayoutAfterUpgrade.toString()} USDC (wei)`
      );
      console.log(
        `Actual payout received: ${receivedAmount.toString()} USDC (wei)`
      );
      console.log(
        `Funds lost: ${originalPayoutAfterUpgrade.toString()} USDC (wei)`
      );
      console.log("==========================\n");

      // The user should have received 0 because the new calculator returns 0 for historical drawings
      expect(receivedAmount).to.equal(
        0n,
        "User should receive 0 USDC after calculator upgrade - funds lost!"
      );

      // The ticket was burned, so the user can never claim again
      await expect(
        jackpotNFT.ownerOf(ticketIds[0])
      ).to.be.revertedWithCustomError(jackpotNFT, "TokenDoesNotExist");

      // Summary: The user lost their winnings (originalPayout amount) because:
      // 1. The payout was stored in the original calculator during settlement
      // 2. The owner upgraded to a new calculator
      // 3. claimWinnings() queries the current calculator (new one) which has no historical data
      // 4. User receives 0 USDC and ticket is burned, making the funds permanently unclaimable
    });
  });
});
```

Logs:

```
  C4
    PoC

=== VULNERABILITY IMPACT ===
Expected payout (from original calculator): 174910540000 USDC (wei)
Actual payout received: 0 USDC (wei)
Funds lost: 174910540000 USDC (wei)
==========================

      ✔ demonstrates the C4 submission's validity
```
