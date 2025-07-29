# Burve - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
    
- ## High Risk Findings
	- ### [[H-01] Phantom Shares Vulnerability in Fee-Charging Vaults Will Block All Swaps Once a Tipping Point is Reached](H-01)

- ## Medium Risk Findings
	 - ### [[M-01] Missing acceptOwnership() Function Selector Registration Prevents Ownership Transfers](#M-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Burve

### Chain: EVM

### Dates: April 19th, 2025 - May 7th, 2025 

[See more contest details here](https://audits.sherlock.xyz/contests/858)
# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 1
- Medium: 1

# High Risk Findings

## <a id='H-01'></a>H-01 Phantom Shares Vulnerability in Fee-Charging Vaults Will Block All Swaps Once a Tipping Point is Reached

## Summary:
The incorrect share calculation for deposits into `ReserveLib.sol` for fee-charging vaults will cause denial of service for all users as accumulated fees will create a growing discrepancy between tracked balances and actual balances, eventually preventing swaps.

## Root Cause:
The root cause is how the `shares` mapping tracks accounting in [ReserveLib.sol::deposit()](https://github.com/sherlock-audit/2025-04-burve/blob/main/Burve/src/multi/vertex/Reserve.sol#L32); in the case of a fee charging vault, the amount actually deposited into the vault will be less than the original amount, but the system increments the `shares` based on the raw amount.
```js
    function deposit(  
        VaultProxy memory vProxy,  
        VertexId vid,  
        uint256 amount  
    ) internal returns (uint256 shares) {  
        Reserve storage reserve = Store.reserve();  
        uint8 idx = vid.idx();  
        uint128 balance = vProxy.balance(RESERVEID, true);  
        vProxy.deposit(RESERVEID, amount);  
        shares = (balance == 0)  
            ? amount * SHARE_RESOLUTION  
  
        // Calculates shares based on system shares mapping and balance which do not increment by actual amount deposited  
-->         : (amount * reserve.shares[idx]) / balance; // No need for mulDiv.  
  
        // Increment by incorrect shares amount  
-->     reserve.shares[idx] += shares;  
    }  
```
This creates a phantom shares situation where the system tracks more shares than what actually exists in the vault. The vulnerability manifests through a chain of events:

1. Each time fees are earned and deposited into a fee-charging vault, the system records more shares than it should
2. These phantom shares accumulate over time with each operation
3. When the system attempts to withdraw tokens during a swap operation, it calculates withdrawal amounts based on these inflated share counts
4. Once the discrepancy reaches a critical threshold, the `Vertex.trimBalance` function detects that `targetReal > realBalance` and emits an InsufficientBalance event
5. The transaction continues but later fails when attempting to withdraw more tokens than actually exist in the vault

The tipping point occurs when the accumulated phantom shares cause the calculated withdrawal amount to exceed the actual available balance. This is influenced by:

1. The fee percentage charged by the vault
2. The number of operations performed
3. The initial liquidity in the vault as this liquidity provided by the protocol is never recorded for in the internal accounting; this is correct for the record but just highlighting that the number of operations to reach the tipping point is contingent on this initial liquidity provided aswell

In the test case, the system broke after 876 swaps when it believed it had approximately 8x more tokens (7.972e20) than it actually did (9.752e19).
### Internal Pre-conditions

1. The protocol needs to be using at least one fee-charging vault.
2. Users need to perform swaps that generate fees for that vertex.
3. Accumulated shares need to reach the point where tracked balances exceed actual balances, causing checks in `trimBalance` to revert the swap.
### Attack Path

1. A fee-charging vault is used for a vertex
2. Users perform normal operations (adding liquidity, swapping)
3. Each swap generates fees that are processed through `ReserveLib.deposit`
4. For each fee deposit, ReserveLib calculates shares based on the pre-fee amount but the vault only deposits the post-fee amount
5. This discrepancy accumulates over multiple operations
6. After enough operations (876 swaps in the test case), the system believes it has ~8x more tokens than it actually does
7. When attempting to perform operations that require withdrawing these tokens, the transaction reverts with `InsufficientBalance`
8. The vertex becomes unusable as all subsequent swaps involving the token will fail indefinetely.

### Impact

The vertex eventually becomes inaccessible as key functions stop working. Users are unable to swap tokens with this vertex as the counterparty due to the system trying to withdraw more tokens than actually exist in the vaults. This affects all users of the protocol simultaneously and the issue requires a complex migration process to fix once it manifests.

## Recommended Mitigation:

Modify the ReserveLib.deposit function to calculate shares based on the actual amount deposited after fees, rather than the pre-fee amount


---
# Medium Risk Findings

## <a id='M-01'></a>M-01 Missing acceptOwnership() Function Selector Registration Prevents Ownership Transfers

## Summary

Missing function selector registration will cause a permanent inability to complete ownership transfers for protocol administrators as new owners will be unable to call the acceptOwnership() function through the diamond contract.
## Root Cause

As per the README.md:

> We conform to ERC-2535.

This is the Diamond Proxy pattern and the project opts to use the `BaseAdminFacet` as the facet for admin functions. This base facet requires a two-step process for transferring ownership.

For [ERC-2535](https://eips.ethereum.org/EIPS/eip-2535#:~:text=When%20an%20external%20function%20is%20called%20on%20a%20diamond%20its%20fallback%20function%20is%20executed.%20The%20fallback%20function%20determines%20which%20facet%20to%20call%20based%20on%20the%20first%20four%20bytes%20of%20the%20call%20data%20\(known%20as%20the%20function%20selector\)%20and%20executes%20that%20function%20from%20the%20facet%20using%20delegatecall.)

> When an external function is called on a diamond its fallback function is executed. The fallback function determines which facet to call based on the first four bytes of the call data (known as the function selector) and executes that function from the facet using delegatecall.

When registering the selectors in [Diamond.sol::constructor()::lines:68-71,](https://github.com/sherlock-audit/2025-04-burve/blob/main/Burve/src/multi/Diamond.sol#L68) the adminSelectors array includes only three function selectors from `BaseAdminFacet` (`transferOwnership()`, `owner()`, and `adminRights()`), but omits the crucial `acceptOwnership()` selector. As a result, the ownership cannot be transferred.

## Internal Pre-conditions

1. Current owner needs to call `transferOwnership()` to initiate an ownership transfer
2. `pendingOwner` in the `AdminRegistry` storage is set to the new owner's address
3. New owner must call `acceptOwnership()` to complete the transfer process
## Attack Path

1. Current owner calls transferOwnership(newOwner) which succeeds and sets pendingOwner
2. New ownership remains in a pending state, requiring confirmation
3. New owner attempts to call acceptOwnership() to complete the transfer
4. The call reverts with "FunctionNotFound" error because the function selector was never registered in the diamond
5. The ownership transfer cannot be completed, leaving the old owner as the owner

## Impact

The protocol suffers from a permanent inability to complete ownership transfers.

High Impact x Low Likelihood = Medium Severity

## Recommended Mitigation:

Change the adminSelectors array to include all four function selectors:

```js
bytes4[] memory adminSelectors = new bytes4[](4);  
adminSelectors[0] = BaseAdminFacet.transferOwnership.selector;  
adminSelectors[1] = BaseAdminFacet.owner.selector;  
adminSelectors[2] = BaseAdminFacet.adminRights.selector;  
adminSelectors[3] = BaseAdminFacet.acceptOwnership.selector; 
```
---