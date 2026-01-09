# Users unable to claim rewards when campaign is paused

- **Project:** Nudge.xyz  
- **Finding ID:** F-59 / S-170  
- **Severity:** Low  
- **Reporter:** 0xMilenov  

---

## Description

The NudgeCampaign contract implements a reward system where users can participate in campaigns by holding specific tokens and claim rewards after a holding period.

A serious discrepancy exists in the `claimRewards()` function, which is protected by the `whenNotPaused` modifier. This modifier prevents the function from being called when the campaign is paused.

However, this directly conflicts with Nudge's documented policy, which explicitly states that pausing a campaign should only prevent new participations while allowing existing participants to claim their earned rewards.

### Reference from the documentation

**Pausing a campaign**

> If for some reason you would like to “deactivate” your campaign, meaning preventing new participations from happening, please get in touch with us.  
> Note that deactivating a campaign will only stop the campaign smart contract from accepting new participations, it won’t stop users from claiming the rewards they are owed.

### Affected function

    function claimRewards(uint256[] calldata pIDs)
        external
        whenNotPaused
    {

The current implementation means that if a campaign is paused, legitimate participants who have completed their holding period are unable to claim their rightfully earned rewards. This creates a significant risk as user funds could become temporarily or permanently locked in the contract, depending on whether the campaign is ever unpaused.

This issue represents a direct violation of Nudge's documented policy regarding pausing a campaign and reward claims.

---

## Proof of Concept

**Documentation:**  
https://docs.nudge.xyz/technical-documentation/for-protocols/managing-campaigns  
Section: *Pausing a campaign*

### Scenario

1. User participates in an active campaign by calling `handleReallocation()`
2. User waits for the required `holdingPeriodInSeconds`
3. Before the user can claim their rewards, Nudge admin calls  
   `pauseCampaigns(address[] calldata campaigns)`
4. User attempts to call `claimRewards()`, but the transaction reverts due to the `whenNotPaused` modifier
5. User's rewards are locked in the contract until the campaign is unpaused

---

## Recommended mitigation steps

Remove the `whenNotPaused` modifier from the `claimRewards()` function to allow reward claims regardless of campaign status. The modifier should only be present on functions that create new participations, such as `handleReallocation()`, or change/update the documentation.

### Suggested change

    - function claimRewards(uint256[] calldata pIDs) external whenNotPaused {
    + function claimRewards(uint256[] calldata pIDs) external {

---

## Links to affected code

- `NudgeCampaign.sol` — lines 256–300
