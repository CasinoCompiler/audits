# Crestral Network - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
    
- ## High Risk Findings
	- ### [[H-01] Anyone can transfer approved ERC20 tokens as the payWithERC20 function lacks authorization controls](#H-01)
# <a id='contest-summary'></a>Contest Summary

### Sponsor: Crestral Network

### Chain: EVM

### Dates: March 11th, 2025 - March 14th, 2025

[See more contest details here](https://audits.sherlock.xyz/contests/755?filter=questions)
# <a id='results-summary'></a>Results Summary

### Number of findings:

High: 1

# High Risk Findings

## <a id='H-01'></a>H-01 Anyone can transfer approved ERC20 tokens as the payWithERC20 function lacks authorization controls

## Summary

The public visibility of the payWithERC20 function without proper authorization checks will cause unauthorized token transfers for users who have approved the contract as anyone can call the function and transfer tokens from any address that has provided allowance to the contract.

## Root Cause

In [Payment.sol:25](https://github.com/sherlock-audit/2025-03-crestal-network/blob/main/crestal-omni-contracts/src/Payment.sol#L25) the `payWithERC20` function is declared as public but lacks a verification that `msg.sender == fromAddress`, which means anyone can call this function to transfer tokens from any address that has approved the contract.

## Internal Pre-conditions

1. A user needs to approve the contract to spend their ERC20 tokens (which happens naturally during normal protocol operations like `createAgent`, `userTopUp`, or `updateWorkerDeploymentConfig`)
2. The user must have a balance of tokens greater than zero in the specified ERC20 token

## External Pre-conditions

None required

## Attack Path

1. Alice approves the contract to spend her tokens for legitimate protocol operations
2. The attacker observes this approval transaction on-chain
3. The attacker calls `payWithERC20` directly, specifying Alice's address as `fromAddress`, their own address as `toAddress`, Alice's approved token as `erc20TokenAddress`, and an amount within Alice's approval limit
4. The contract executes `token.safeTransferFrom(fromAddress, toAddress, amount)` which transfers tokens from Alice to the attacker

## Impact

The users of the protocol suffer loss of their approved ERC20 tokens. The attacker gains all tokens they manage to transfer. Since many protocol operations require token approvals, a significant portion of the user base could be affected, potentially leading to loss of all approved tokens for multiple users.
## Recommended Mitigation:

Make the payWithERC20 function internal so it can only be called from within the contract itself

---
