# Initialization and admin updates revert for normalBallMax <5 or >128

After an internal discussion ([thread #4](https://code4rena.com/audits/2025-11-megapot/inbox/4)), the protocol owner requires `normalBallMax` to support the full 1–255 range. Under the current design, any configuration with `normalBallMax < 5` or `normalBallMax > 128` reverts during LP cap recomputation, making the requested range unattainable and blocking administration/business logic.

At a high level, the system sizes ticket space via binomial combinatorics and uses that to derive LP pool sizing. The `Combinations` library implements [`choose()`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/lib/Combinations.sol#L7-L29) (binomial coefficient), which feeds upstream pool sizing logic.

```solidity
function choose(uint256 n, uint256 k) internal pure returns (uint256 result) {

    assert(n >= k); // <-- here
    assert(n <= 128); // Artificial limit to avoid overflow // <-- here

    unchecked {
      uint256 out = 1;
      for (uint256 d = 1; d <= k; ++d) {
        out *= n--;
        out /= d;
      }
      return out;
    }
  }
```

In `Jackpot`, the admin function [`setNormalBallMax()`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/Jackpot.sol#L886-L893) updates the game’s normal ball range and, as part of that operation, recalculates the LP pool cap by calling `jackpotLPManager.setLPPoolCap()` with a value produced by [`_calculateLpPoolCap()`](https://github.com/code-423n4/2025-11-megapot/blob/f0a7297d59c376e38b287b2c56740617dbbfbdc7/contracts/Jackpot.sol#L1469-L1476) - a calculation that ultimately depends on `choose(n, k)` where `n` is the candidate `normalBallMax` and `k` is the fixed draw size (e.g., 5).

```solidity
function setNormalBallMax(uint8 _normalBallMax) external onlyOwner {

    uint8 oldNormalBallMax = normalBallMax;
    jackpotLPManager.setLPPoolCap(currentDrawingId, _calculateLpPoolCap(_normalBallMax));
    normalBallMax = _normalBallMax;

    emit NormalBallMaxUpdated(currentDrawingId, oldNormalBallMax, _normalBallMax);
}
```

`setNormalBallMax()` performs the LP cap recomputation before assigning the new `normalBallMax` to state and because `Combinations.choose()` hard-limits `n` to 128 and requires `n >= k`, the recomputation reverts for any `_normalBallMax` outside `5–128`. As a result, the administrative update fails for intended configurations within the 1–255 business range. Initialization flows that perform the same recomputation are affected in the same way.

```solidity
function _calculateLpPoolCap(uint256 _normalBallMax) internal view returns (uint256) {

    uint256 maxAllowableTickets = Combinations.choose(_normalBallMax, NORMAL_BALL_COUNT) * (MAX_BIT_VECTOR_SIZE - _normalBallMax);
    uint256 maxPrizePool = maxAllowableTickets * ticketPrice * (PRECISE_UNIT - lpEdgeTarget) / PRECISE_UNIT;

    return Math.min(maxPrizePool * PRECISE_UNIT / (PRECISE_UNIT - reserveRatio), governancePoolCap);
}
```

Root cause:

- A hardcoded upper bound in `Combinations.choose()` limits `n` to 128 and a precondition enforces `n >= k`. This prevents legitimate configurations such as `_normalBallMax = 130` (desired protocol range is up to 255) and also rejects values below the draw size (e.g., `_normalBallMax < 5`) even if business logic would otherwise allow shrinking the range.

Highest-impact scenario:

- The protocol cannot be configured to the intended 1–255 range. Attempting to set `_normalBallMax` to any value >128 (or <5) reverts before state is updated, blocking administration. In deployments where initialization relies on the same computation, we have the same problem.

## Recommended mitigation steps

If the protocol insists on the full `1–255` range, update/improve `Combinations.choose()` to correctly support that range.  
A good example is to remove the artificial cap and apply symmetry to minimize intermediate growth:

```solidity
function choose(uint256 n, uint256 k) internal pure returns (uint256 result) {
  require(n >= k, "Combinations: n < k");
  unchecked {
    if (k > n - k) k = n - k; // symmetry: C(n,k) = C(n,n-k)
    uint256 out = 1;
    for (uint256 d = 1; d <= k; d++) {
      out *= n--;
      out /= d;
    }
    return out;
  }
}
```

However, this must be validated end-to-end to ensure it does not break dependent logic or embedded assumptions elsewhere.

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
      // setNormalBallMax() reverts for values outside 5-128 range
      // due to hardcoded limits in Combinations.choose() function

      const initialNormalBallMax = await jackpot.normalBallMax();
      expect(initialNormalBallMax).to.equal(BigInt(30));

      // Test 1: Setting normalBallMax to 130 (> 128) should revert
      // This demonstrates the upper bound limitation
      await expect(jackpot.connect(owner.wallet).setNormalBallMax(130)).to.be
        .reverted;

      // Verify normalBallMax was not updated
      const normalBallMaxAfterFirstFailedUpdate = await jackpot.normalBallMax();
      expect(normalBallMaxAfterFirstFailedUpdate).to.equal(
        initialNormalBallMax
      );

      // Test 2: Setting normalBallMax to 4 (< 5) should revert
      // This demonstrates the lower bound limitation (n >= k where k=5)
      await expect(jackpot.connect(owner.wallet).setNormalBallMax(4)).to.be
        .reverted;

      // Verify normalBallMax was not updated
      const normalBallMaxAfterSecondFailedUpdate =
        await jackpot.normalBallMax();
      expect(normalBallMaxAfterSecondFailedUpdate).to.equal(
        initialNormalBallMax
      );
    });
  });
});
```

Logs:

```
C4
    PoC
      ✔ demonstrates the C4 submission's validity
```
