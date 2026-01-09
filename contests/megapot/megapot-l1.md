# Bonusball packing overflow corrupts tiering and enables mis-payouts/DoS

The jackpot system packs tickets into a 256-bit bit vector in `TicketComboTracker.sol` and later computes claim-time tiers from the packed representation in `Jackpot.sol`. Normal balls occupy bits `1..ballMax` and the bonusball is packed at bit position `ballMax + bonusball`. The protocol dynamically sets `bonusballMax` each drawing in `Jackpot.sol::_setNewDrawingState()`, and uses `TicketComboTracker.sol::insert()` to pack tickets during purchase and `Jackpot.sol::_calculateTicketTierId()` to classify tiers at claim time.

At a high level:

- [`TicketComboTracker.sol::insert()`](https://github.com/code-423n4/2025-11-megapot/blob/5cda5779d1a157f847dd13700282dc09558806e4/contracts/lib/TicketComboTracker.sol#L112-L143) builds the ticket bit vector and places the bonusball bit at index `(_tracker.normalMax + _bonusball)`.

```solidity
function insert(
     Tracker storage _tracker,
     uint8[] memory _normalBalls,
     uint8 _bonusball
 )
     internal
     returns (uint256 ticketNumbers, bool isDup)
 {
     require(_normalBalls.length == _tracker.normalTiers, "Invalid pick length");
     uint256 set = toNormalsBitVector(_normalBalls, _tracker.normalMax);
     // Iterate over all tier combos and store the combo counts
     isDup = _tracker.comboCounts[_bonusball][set].count > 0;
     for (uint8 i = 1; i <= _tracker.normalTiers; i++) {
         uint256[] memory subsets = Combinations.generateSubsets(set, i);
         for (uint256 j = 0; j < subsets.length; j++) {
             if (isDup) {
                 _tracker.comboCounts[_bonusball][subsets[j]].dupCount++;
             } else {
                 _tracker.comboCounts[_bonusball][subsets[j]].count++;
             }
         }
     }

     if (isDup) {
         _tracker.bonusballTicketCounts[_bonusball].dupCount++;
     } else {
         _tracker.bonusballTicketCounts[_bonusball].count++;
     }

     // Add the bonusball to the bit vector
     ticketNumbers = set |= 1 << (_bonusball + _tracker.normalMax); // <-- here
 }
```

- [`TicketComboTracker.sol::countTierMatchesWithBonusball()`](https://github.com/code-423n4/2025-11-megapot/blob/5cda5779d1a157f847dd13700282dc09558806e4/contracts/lib/TicketComboTracker.sol#L250-L270) computes exact winner counts across tiers for settlement using explicit `_bonusball`-keyed maps (not relying on packed extraction).

```solidity
function countTierMatchesWithBonusball(
     Tracker storage _tracker,
     uint8[] memory _normalBalls,
     uint8 _bonusball
 )
     internal
     view
     returns (uint256 winningTicket, uint256[] memory uniqueResult, uint256[] memory dupResult)
 {
     uint256 set = toNormalsBitVector(_normalBalls, _tracker.normalMax);
     winningTicket = set | (1 << (_bonusball + _tracker.normalMax));

     // Step 1: Count all subset matches across all bonusballs
     (uint256[] memory matches, uint256[] memory dupMatches) = _countSubsetMatches(_tracker, set, _bonusball);

     // Step 2: Apply inclusion-exclusion principle to remove double counting
     (uniqueResult, dupResult) = _applyInclusionExclusionPrinciple(_tracker, matches, dupMatches);

     // Step 3: Calculate bonusball-only matches (no normal balls matched)
     _calculateBonusballOnlyMatches(_tracker, _bonusball, uniqueResult, dupResult);
 }
```

- [`Jackpot.sol::_calculateTicketTierId()`](https://github.com/code-423n4/2025-11-megapot/blob/5cda5779d1a157f847dd13700282dc09558806e4/contracts/Jackpot.sol#L1651-L1669) extracts the bonusball by right-shifting the packed ticket and winning numbers by `(_normalBallMax + 1)`, and then uses that to decide which of the 12 tiers the ticket falls into for payout.

```solidity
  function _calculateTicketTierId(uint256 _ticketNumbers, uint256 _winningNumbers, uint256 _normalBallMax) internal pure returns (uint256) {
     uint256 matches = 0;

     // Count matching normal numbers by checking overlapping bits
     uint256 matchingBits = _ticketNumbers & _winningNumbers;

     // Count the number of set bits (matches)
     matches = LibBit.popCount(matchingBits);

     // Extract bonusball from both ticket and winning numbers
     // Bonusball is stored in the highest bits after the normal numbers
     uint256 ticketBonusball = _ticketNumbers >> (_normalBallMax + 1);
     uint256 winningBonusball = _winningNumbers >> (_normalBallMax + 1);

     uint256 bonusballMatch = (ticketBonusball == winningBonusball) ? 1 : 0;

     // We count all matches including the bonusball so if the bonusball is a match we need to subtract it from matches
     return 2 * (matches - bonusballMatch) + bonusballMatch;
 }
```

Root cause:

- There is no invariant enforcing `normalBallMax + bonusballMax <= 255`. If `normalBallMax + bonusball > 255`, the shift `1 << (normalBallMax + bonusball)` overflows the 256-bit boundary and collapses to zero, silently erasing the bonusball bit from the packed ticket and from the packed winning numbers.
- This can occur because `@Jackpot.sol::_setNewDrawingState()` computes `bonusballMax` each drawing without clamping against the bit-vector capacity. Admin functions like `setNormalBallMax()` and `setBonusballMin()` also lack boundary checks.
- The constant `MAX_BIT_VECTOR_SIZE = 255` exists in `@Jackpot.sol` but is only used for pool-cap math (`_calculateLpPoolCap()`), not to enforce the packing limit.

Highest-impact scenario:

- Mis-payouts (funds theft/drain): With the bonusball bit missing, `Jackpot.sol::_calculateTicketTierId()` interprets both ticket and winning bonusball as zero, frequently treating the bonusball as “matched” when it is not. Tiers are skewed (e.g., a “2 normals only” ticket may appear as “1 normal + bonusball”), allowing claims from more lucrative tiers than settlement budgeted for. Since settlement used accurate counts keyed by the real `_bonusball`, total paid out can exceed allocated tier totals and the recorded `drawingUserWinnings`, draining funds.
- Claim-time DoS: For tickets with zero normal matches, the tier computation subtracts the (assumed) bonusball match from `matches`, underflowing the arithmetic and causing claim reverts. This can partially DoS user claims.

> [!NOTE]
> The winner counting path (`countTierMatchesWithBonusball()`) is correct because it keys by explicit `_bonusball`. The defect arises at claim time when tiers are derived from packed bit extraction that assumes the bonusball bit exists.

## Proof of Concept

Replace the contents of '/test/poc/C4PoC.spec.ts' with the code below and run `yarn pos`

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
    it("demonstrates bonusball packing overflow causing claim-time revert (DoS)", async () => {
      // 1) Push parameters to overflow domain within combinatorics constraint (n<=128):
      //    Choose normalBallMax = 128 and bonusballMin = 130 so 128 + 130 > 255
      await jackpot.connect(owner.wallet).setNormalBallMax(128);
      await jackpot.connect(owner.wallet).setBonusballMin(130);

      // 2) End current drawing to initialize the next drawing with new params
      const drawingId1 = Number(await jackpot.currentDrawingId());
      const drawingState1 = await jackpot.getDrawingState(drawingId1);
      await time.increaseTo(Number(drawingState1.drawingTime) + 1);
      const fee1 = await jackpot.getEntropyCallbackFee();
      await jackpot.connect(owner.wallet).runJackpot({ value: fee1 });
      // Provide any valid random numbers within previous drawing's ranges
      await entropyProvider
        .connect(owner.wallet)
        .randomnessCallback([[1n, 2n, 3n, 4n, 5n], [1n]]);

      // 3) New drawing (drawingId2) now has ballMax=128 and bonusballMax >= 130
      const drawingId2 = Number(await jackpot.currentDrawingId());
      const drawingState2 = await jackpot.getDrawingState(drawingId2);

      // 4) Attempting to buy a valid-format ticket now causes a panic due to overflow in bonusball packing (DoS)
      await usdcMock.connect(owner.wallet).mint(buyerOne.address, usdc(1000));
      await usdcMock
        .connect(buyerOne.wallet)
        .approve(await jackpot.getAddress(), drawingState2.ticketPrice);

      await expect(
        jackpot.connect(buyerOne.wallet).buyTickets(
          [
            {
              normals: [1, 2, 3, 4, 5],
              bonusball: 130,
            },
          ],
          buyerOne.address,
          [],
          [],
          ethers.id("POC")
        )
      ).to.be.revertedWithPanic(0x11);
    });
  });
});
```

Logs:

```
  C4
    PoC
      ✔ demonstrates bonusball packing overflow causing claim-time revert (DoS)


  1 passing (89ms)
```

## Recommended mitigation steps

- Enforce a global invariant: `normalBallMax + bonusballMax <= 255` at all times.
  - Clamp in `@Jackpot.sol::_setNewDrawingState()`:
    - `newBonusball = min(newBonusball, uint8(255 - normalBallMax))`
    - If clamping would reduce below `bonusballMin`, revert the drawing configuration instead (fail fast).
  - Add `require()` guards to admin setters:
    - `setNormalBallMax()` must ensure `newNormalBallMax + current bonusballMax <= 255`.
    - `setBonusballMin()` must ensure `normalBallMax + newBonusballMin <= 255`.
