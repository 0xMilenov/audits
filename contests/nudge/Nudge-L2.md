# Campaign creation with zero parameters enables operational attacks and DoS risk

- **Project:** Nudge.xyz  
- **Finding ID:** F-117 / S-713  
- **Severity:** Low  
- **Reporter:** 0xMilenov  

---

## Description

The Nudge platform operates with a dual-contract system where `NudgeCampaignFactory.sol` deploys instances of `NudgeCampaign.sol` contracts. These campaign contracts implement a reward mechanism where users who reallocate tokens to a target token can receive rewards proportional to their participation.

The core reward calculation occurs in the `getRewardAmountIncludingFees()` function in `NudgeCampaign.sol`, which uses the `rewardPPQ` parameter (reward in parts per quadrillion) to determine the reward amount for a given token allocation:

    function getRewardAmountIncludingFees(uint256 toAmount)
        public
        view
        returns (uint256)
    {
        // If both tokens have 18 decimals, no scaling needed
        if (targetScalingFactor == 1 && rewardScalingFactor == 1) {
            return toAmount.mulDiv(rewardPPQ, PPQ_DENOMINATOR);
        }

        // Scale amount to 18 decimals for reward calculation
        uint256 scaledAmount = toAmount * targetScalingFactor;

        // Calculate reward in 18 decimals
        uint256 rewardAmountIn18Decimals =
            scaledAmount.mulDiv(rewardPPQ, PPQ_DENOMINATOR);

        // Scale back to reward token decimals
        return rewardAmountIn18Decimals / rewardScalingFactor;
    }

The issue lies in the campaign creation process, where the contract fails to validate two parameters: `rewardPPQ` and `initialRewardAmount`. Both parameters can be set to zero during campaign creation. This is problematic because when `rewardPPQ` is zero, the entire reward calculation will result in zero regardless of token amount. When combined with zero `initialRewardAmount`, this creates a campaign that appears functional but will never provide rewards to participants, as both the reward rate and the reward pool are effectively empty.

When users interact with the campaign through the `handleReallocation()` function, it performs a check to ensure there are sufficient rewards available:

    uint256 rewardsAvailable = claimableRewardAmount();
    if (rewardAmountIncludingFees > rewardsAvailable) {
        revert NotEnoughRewardsAvailable();
    }

However, when `rewardPPQ` is zero, `rewardAmountIncludingFees` will also be zero, meaning this check will always pass, even if there are no actual rewards available in the contract (`rewardsAvailable` will be zero). This creates two significant problems:

- **Operational Misdirection:** Users can successfully participate in a campaign that will never provide rewards, as the `handleReallocation()` function will execute successfully but assign zero rewards.
- **Potential DoS Attack:** A malicious actor can create numerous campaigns with zero reward parameters at minimal cost, populating the factory's campaign tracking mechanisms and potentially making legitimate campaign management more difficult.

---

## Proof of Concept

An attacker creates a campaign with the following parameters:

- `targetToken`: Any valid token address  
- `rewardToken`: Any valid token address  
- `rewardPPQ`: 0  
- `initialRewardAmount`: 0 (no initial funding)  

The factory deploys the campaign contract and registers it within its tracking systems (`isCampaign` mapping and `campaignAddresses` array).

Users interact with the campaign through `handleReallocation()`, which successfully executes because:

- `rewardAmountIncludingFees = getRewardAmountIncludingFees(amountReceived)` evaluates to `0`
- The check `if (rewardAmountIncludingFees > rewardsAvailable)` passes since `0` is not greater than `0`

Users receive zero rewards despite successfully completing the token reallocation process, creating a misleading user experience.

The attacker can repeat this process to create hundreds or thousands of zero-reward campaigns, bloating the factory's state and potentially affecting operations.

---

## Recommended mitigation steps

Add validation during campaign creation to ensure that `rewardPPQ` is non-zero:

    function deployCampaign(
        uint32 holdingPeriodInSeconds,
        address targetToken,
        address rewardToken,
        uint256 rewardPPQ,
        address campaignAdmin,
        uint256 startTimestamp,
        address alternativeWithdrawalAddress,
        uint256 uuid
    ) public returns (address campaign) {
        if (campaignAdmin == address(0)) revert ZeroAddress();
        if (targetToken == address(0) || rewardToken == address(0)) revert ZeroAddress();
        if (holdingPeriodInSeconds == 0) revert InvalidParameter();
        if (rewardPPQ == 0) revert InvalidParameter(); // Add this validation

        // ... rest of the function ...
    }

Additionally, consider adding a minimum initial reward amount requirement when campaigns are deployed to ensure that campaigns are properly funded from the start:

    function deployAndFundCampaign(
        // ... existing parameters ...
    ) external payable returns (address campaign) {
        // ... existing validations ...
        if (initialRewardAmount == 0) revert InvalidParameter(); // Add this validation

        // ... rest of the function ...
    }

These changes will prevent campaigns with zero rewards from being created, protecting both the platform from potential DoS attacks and users from participating in campaigns that won't provide rewards.

---

## Links to affected code

- `NudgeCampaign.sol` — lines 58–233  
- `NudgeCampaignFactory.sol` — lines 57–179
