# DESK Orderbook - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
	- ### [[M-01] User Funds in Vault Due to Lack of Rejected Deposit Recovery Mechanism](#M-01) 

# <a id='contest-summary'></a>Contest Summary

### Sponsor: DESK Orderbook

### Chain: EVM

### Dates: January 6th, 2025 - January 20th, 2025

[See more contest details here](https://cantina.xyz/competitions/bd43bdd1-bc7f-473b-96c0-d35d37f3db33)
# <a id='results-summary'></a>Results Summary

### Number of findings:

- Medium: 1

# Medium Risk Findings

## <a id='M-01'></a>M-01 Irretrievable User Funds in Vault Due to Lack of Rejected Deposit Recovery Mechanism

## Summary
Users' funds can become permanently locked in `Vault.sol` if their deposit is rejected by the `DepositHandler.sol`, as there is no mechanism to reclaim rejected deposits and the standard withdrawal flow will fail.

## Description:
The Vault contract implements a two-step deposit process where user funds are first transferred to the Vault, and then the DepositHandler must process the deposit to credit the user's account. However, if the DepositHandler rejects the deposit, the user's funds remain in the Vault with no way to retrieve them.

The issue arises from the disconnect between fund custody and accounting. When a user deposits funds through `Vault.sol::deposit()`, their tokens are immediately transferred to the Vault contract and a deposit request is created. The DepositHandler can then either process or reject this deposit request. In the case of rejection, the user's funds remain in the Vault, but no balance is credited to their account in the AssetService.

When the user attempts to withdraw their funds after a rejection, they will fail the withdrawable amount check in `Vault.sol::withdraw()`:

```js
if (_amount > IWithdrawHandler(withdrawHandler).getWithdrawableAmount(_subaccount, _tokenAddress)) {
    revert Vault_ExceedWithdrawableAmount();
}

```

This check fails because `getWithdrawableAmount()` returns 0 since no balance was credited to their account when the deposit was rejected. The result is that user funds become permanently locked in the Vault contract with no mechanism for recovery.
## Impact:
This is a High severity issue because:

1. It results in a permanent loss of user funds with no recovery mechanism
2. It affects the core deposit/withdrawal functionality of the protocol
3. It can impact any user making deposits that might be rejected
4. The locked funds cannot be recovered even by protocol administrators

## Likelihood:
The likelihood is Medium because:

1. Deposit rejections are a normal part of protocol operation
2. There are multiple legitimate reasons a deposit might be rejected (e.g. market conditions, compliance checks)
3. Users cannot predict whether their deposit will be rejected

## Recommended Mitigation:
There are a few different methods to fix this.

1. Add a dedicated refund mechanism for rejected deposits to the Vault contract, e.g:

```js
function refundRejectedDeposit(uint256 _requestId) external nonReentrant {
    // Verify request exists
    DepositRequest memory request = depositRequests[_requestId];
    if (request.subaccount == bytes32(0)) {
        revert Vault_InvalidRequest();
    }
    
    // Verify caller owns the deposit
    if (msg.sender != address(bytes20(request.subaccount))) {
        revert Vault_InvalidAuthentication();
    }
    
    // Verify deposit was rejected
    if (!depositHandler.isProcessed(_requestId)) {
        revert Vault_RequestNotProcessed();
    }
    
    // Transfer tokens back to user
    if (request.tokenAddress == WETH && shouldUnwrapWeth) {
        IWETH(WETH).withdraw(request.amount);
        msg.sender.safeTransferETH(request.amount);
    } else {
        IERC20(request.tokenAddress).safeTransfer(
            msg.sender, 
            request.amount
        );
    }
    
    emit LogDepositRefunded(
        _requestId,
        msg.sender,
        request.tokenAddress,
        request.amount
    );
}

```

2. Create an accounting mapping within `Vault.sol` to track user deposits into the vault and thereafter modify existing `withdraw()` function.

---
