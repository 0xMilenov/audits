# Malicious actors can front-run campaign deployments using duplicate UUIDs

- **Project:** Nudge.xyz  
- **Finding ID:** F-6 / S-155  
- **Severity:** Low  
- **Reporter:** 0xMilenov  

---

## Description

The `NudgeCampaignFactory` contract uses `CREATE2` for deterministic address generation when deploying new `NudgeCampaign` contracts. While this approach ensures address predictability, it introduces a security vulnerability since the contract lacks validation for duplicate UUIDs. This vulnerability can lead to front-running attacks where malicious actors can deploy campaigns with identical UUIDs ahead of legitimate users.

### NudgeCampaignFactory

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

This design assumes UUIDs will be unique, but there's no on-chain enforcement mechanism. The issue is compounded by the fact that Nudge's backend systems likely rely on these UUIDs for campaign tracking and management. Furthermore, there is a significant risk that the frontend application uses these UUIDs to identify and interact with specific campaigns, which could lead to users unknowingly interacting with malicious copycat campaigns instead of their intended legitimate targets.

---

## Proof of Concept

1. Alice deploys a campaign with a UUID (e.g., `1234`) using the `deployCampaign()` or `deployAndFundCampaign()` function  
2. Alice submits the transaction to deploy the campaign to the mempool  
3. Bob (an attacker) monitors the mempool and identifies Alice's pending transaction  
4. Bob extracts the UUID from Alice's transaction  
5. Bob creates an identical transaction but with higher gas fees to ensure his transaction is processed first  
6. Bob's campaign with UUID `1234` is deployed before Alice's campaign  
7. Nudge's off-chain systems track Bob's malicious campaign as the legitimate one with UUID `1234`  
8. Alice's legitimate campaign is deployed but causes confusion in backend systems since the UUID is already in use  

The problem exists because the contract only uses the UUID as an input to the address calculation but doesn't track or verify the uniqueness of UUIDs.

---

## Recommended mitigation steps

Implement UUID uniqueness validation by creating a mapping to track used UUIDs:

    mapping(uint256 => bool) public usedUUIDs;

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
        if (usedUUIDs[uuid]) revert DuplicateUUID();

        // Mark UUID as used
        usedUUIDs[uuid] = true;

        // Rest of the function remains unchanged
        ...
    }

Additionally, create a custom error for duplicate UUIDs:

    error DuplicateUUID();

This solution ensures that each UUID can only be used once, preventing front-running attacks that exploit duplicate UUIDs.

---

## Links to affected code

- `NudgeCampaignFactory.sol` — lines 57–179
