# Core Contracts - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [[H-01] Interest Accrual Failure Due to Incorrect Scaling in RToken Implementation](#H-01)
    - ### [[H-02] Fee Collection System Undermines Voter Incentives Due to Supply/Power Mismatch in Reward Distribution](#H-02)
    - ### [[H-03] Critical Economic Design Flaw in ZENO Zero-Coupon Bond Implementation Leads to Guaranteed User Losses](#H-03)
    - ### [[H-04] StabilityPool Cannot Execute Liquidations Due to Missing crvUSD Transfer Mechanism](#H-04)
    - ### [[H-05] Users Can Overwrite Existing Locks in veRAACToken Resulting in Permanent Loss of Funds](#H-05)
    - ### [[H-06] Double Liquidity Index Scaling in RToken.sol Transfer Functions Leads to Incorrect Token Amounts](#H-06)
    - ### [[H-07] Double Usage Index Scaling in StabilityPool Liquidation Inflates Required CRVUSD Balance](#H-07)
    - ### [[H-08] Incorrect Collateralization Check in LendingPool Allows Excessive Borrowing](#H-08)
    - ### [[H-09] Fee rewards are permanently lost for users who claim early due to incorrect tracking of claimed rewards](#H-09)
- ## Medium Risk Findings
    - ### [[M-01] Interest Rate Model Uses Prime Rate Instead of Optimal Rate at Optimal Utilization](#M-01)
    - ### [[M-02] Incorrect Emission Rate Used for Past Block Rewards](#M-02)
    - ### [[M-03] Incorrect Vault Share Accounting Leads to Inaccurate Liquidity Management](#M-03)
    - ### [[M-04] RWAGauge Sequencer Downtime Period Misalignment](#M-04)
    - ### [[M-05] Liquidation Grace Period Can Be Unfairly Shortened By Protocol Pause](#M-05)
    - ### [[M-06] RToken.sol:: transferFrom() and transfer() use inconsistent liquidity index sources leading to incorrect token transfers"](#M-06)
    - ### [[M-07] ReservePool fails to implement updateLiquidityIndex function, permanently freezing RToken's transferFrom() scaling at initial value"](#M-07)
    - ### [[M-08] GaugeController.sol uses incorrect voting power metric leading to compromised voting weight](#M-08)
    - ### [[M-09] Emergency Revocation Results in Permanent Token Lock](#M-09)
    - ### [[M-10] RToken.sol::calculateDustAmount() unable to to execute due to incorrect implementation, resulting in dust stuck in contract indefinitely](#M-10)
    - ### [[M-11] Incorrect `RAACMinter.sol::calculateNewEmissionRate()` Due to Index vs Balance Mismatch](#M-11)
    - ### [[M-12] BaseGauge.sol Uses Raw Balance Instead of Voting Power for Direction Voting](#M-12)
- ## Low Risk Findings
    - ### [[L-01] Extended Recovery Period After Emergency Shutdown Due to Rate Adjustment Limitations](#L-01)
    - ### [[L-02] Incorrect Voting Power Reporting in `veRAACToken.sol::getLockPosition` Function](#L-02)
    - ### [[L-03] Storage Inconsistency in veRAACToken's getLockEndTime Function](#L-03)
    - ### [[L-04] Incorrect Fee Type Initialization Causes Fee Updates to Break and Charges 10x Intended Fee Rate](#L-04)
    - ### [[L-05] Storage Inconsistency in veRAACToken's getLockedBalance Function](#L-05)]

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Regnum Aurum Acquisition Corp

### Chain: EVM

### Dates: Feb 3rd, 2025 - Feb 24th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-02-raac)
# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 9
- Medium: 12
- Low: 5
# High Risk Findings

## <a id='H-01'></a>H-01 Interest Accrual Failure Due to Incorrect Scaling in RToken Implementation            

## Summary

The `RToken.sol` contract fails to properly implement the scaled balance mechanism for interest-bearing tokens, leading to a systemic failure in interest distribution to depositors. While the protocol correctly tracks interest accrual through the liquidity index, the core token scaling logic is implemented incorrectly, causing all interest payments to become trapped in the protocol rather than being distributed to depositors.

## Vulnerability Details

The current `RToken.sol` implementation incorrectly handles the scaling mechanism needed for interest-bearing tokens. In lending protocols, token balances need to be stored in a scaled form to track proportional ownership as interest accrues. This requires:

1. Scaling down deposit amounts by dividing by the current liquidity index when minting tokens
2. Storing these scaled balances internally
3. Scaling up by multiplying by the current liquidity index when calculating real balances or processing withdrawals

The RToken contract instead:

1. Stores unscaled balances during minting: \_mint(onBehalfOf, amount)
2. Burns the unscaled amount: \_burn(from, amount)
3. Transfers the unscaled amount: safeTransfer(receiverOfUnderlying, amount)

This fundamentally breaks the interest distribution mechanism as depositors always receive their exact deposit amount back, regardless of accrued interest.

## Impact

High

* Depositors never receive any interest on their deposits
* Interest payments from borrowers become permanently trapped in the protocol
* The error affects all depositors equally and consistently

## Likelihood

The likelihood is certain (high) as this is not a situational bug but rather a systematic failure in the core token mechanics that will affect every deposit and withdrawal operation.

## Recommendations

Modify RToken to implement proper balance scaling following the pattern used in `DebtToken.sol` in `RToken.sol::` `mint()` `burn()` 

## <a id='H-02'></a>H-02 Fee Collection System Undermines Voter Incentives Due to Supply/Power Mismatch in Reward Distribution            

## Summary

The `FeeCollector.sol` contract's reward distribution mechanism has a critical flaw in how it calculates user reward shares. While a user's voting power naturally decays over their lock period, this decaying power is compared against the non-decaying total supply of veRAAC tokens when determining reward shares. This mismatch causes users to receive disproportionately smaller rewards as their locks ages, effectively punishing long-term holders and resulting in protocol fees becoming permanently locked in the contract.

## Vulnerability Details

The vulnerability stems from a mismatch between how voting power decays and how rewards are calculated in the FeeCollector's distribution mechanism. Let's examine how fees flow through the system and where they get trapped:

The distribution process follows these steps:

1. Protocol fees accumulate in the CollectedFees struct
2. When distributeCollectedFees() is called, it:

* Calculates the total fees to distribute
* Allocates 80% to veRAACToken holders based on voting power
* Records these reward shares in the contract state
* Deletes all collected fees with delete collectedFees

The critical issue emerges from how rewards are calculated versus how they're tracked:

```solidity
share = (totalDistributed * userVotingPower) / totalVotingPower;
```

Where:

* userVotingPower = User's voting power that decays linearly over the lock period
* totalVotingPower = Total supply of veRAAC tokens which does not decay

Consider a user who locks 1000 RAAC tokens for 2 years:

1. At lock creation (t=0):

* Voting Power = 500 (scaled based on lock duration)
* veRAAC Total Supply = 1000
* Reward Share = 500/1000 = 50% of rewards

1. Near lock expiry (t=700 days):

* Voting Power ≈ 15.75 (due to linear decay)
* veRAAC Total Supply = still 1000 (no decay)
* Reward Share = 15.75/1000 = 1.575% of rewards

Because `distributeCollectedFees()` deletes all fee records after calculating these diminished rewards:

```solidity
delete collectedFees;
```

Each distribution cycle permanently locks away an increasing portion of protocol fees as locks age. This creates a compounding loss of value that should have been distributed to protocol participants.

## Impact

High:

* Severely penalizes long-term token holders by drastically reducing their rewards as their lock ages
* Results in protocol fees becoming permanently trapped in the contract
* Creates misaligned incentives that work against the protocol's goal of encouraging long-term holding

## Likelihood

High - This issue will affect every user who locks tokens and every fee distribution event. The impact becomes more severe as locks age and will be particularly pronounced for longer lock durations.

## Recommendations

Compare against total actual voting power rather than total supply:

```solidity
function _calculatePendingRewards(address user) internal view returns (uint256) {
    uint256 totalActualVotingPower = _calculateTotalActualVotingPower(); // <- needs implementing
    if (totalActualVotingPower == 0) return 0;
    
    uint256 userVotingPower = veRAACToken.getVotingPower(user);
    return (totalDistributed * userVotingPower) / totalActualVotingPower;
}
```

## <a id='H-03'></a>H-03 Critical Economic Design Flaw in ZENO Zero-Coupon Bond Implementation Leads to Guaranteed User Losses            

## Summary

The ZENO protocol's zero-coupon bond implementation has a fundamental economic design flaw where users pay a premium price for ZENO tokens but can only redeem them for a fraction of their purchase price, resulting in guaranteed significant losses for users. This completely inverts the intended mechanics of zero-coupon bonds, which should provide a profit through discount pricing.

## Vulnerability Details

The issue stems from a mismatch between the auction purchase price and redemption mechanics:

1. During the auction, users pay a premium price for ZENO tokens:

```solidity
// In Auction.sol
function buy(uint256 amount) external whenActive {
    uint256 price = getPrice();
@>  uint256 cost = price * amount;
    usdc.transferFrom(msg.sender, businessAddress, cost);
    zeno.mint(msg.sender, amount);
}
```

1. However, the redemption is fixed at 1:1 with USDC:

```solidity
// In ZENO.sol
function redeem(uint256 amount) external nonReentrant {
    _burn(msg.sender, amount);
@>  USDC.safeTransfer(msg.sender, amount); // 1 ZENO = 1 USDC
}
```

This is also the case for `ZENO.sol::redeemAll()`

Lets hypothetically think about an auction for 1000 ZENO tokens at a starting price of 900 USDC and a reserve price of 700 USDC.

NOTE: these numbers are similar to the numbers used by the [devs in their own testing](https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/test/unit/Zeno/Integration.test.js#L81)

* The user pays 700,000-900,000 USDC (depending on auction timing)
* At redemption, they receive only 1,000 USDC (1 USDC per ZENO)
* The user loses 699,000-899,000 USDC on their investment.

This scenario is demonstrated in the PoC

## Impact

Critical. The current implementation:

* Guarantees substantial losses  for all users
* Completely fails to function as a zero-coupon bond
* Makes the product economically non-viable

## Likelihood

High - This is not a conditional bug but rather a fundamental design flaw that affects every single transaction in the intended core functionality of the protocol.

## Recommendations

The problem seems to be a lack of correct scaling in the `redeem` and `redeemAll()` function.

It would be correct to scale the redemption amounts by the price at the time of auction and thereafter add the fixed interest they should've accrued.

## <a id='H-04'></a>H-04 StabilityPool Cannot Execute Liquidations Due to Missing crvUSD Transfer Mechanism      
## Summary

The `StabilityPool.sol` contract, which is responsible for liquidating unhealthy positions, lacks any mechanism to obtain or hold crvUSD tokens that are required to execute liquidations. While the StabilityPool can receive rTokens (which represent crvUSD deposits), it cannot access the actual crvUSD tokens needed for the liquidation process. This fundamental disconnect in token flow renders the liquidation mechanism non-functional.

## Vulnerability Details

The vulnerability stems from an architectural misalignment between the token flow design and the liquidation requirements. Let's examine the critical path that reveals this issue:
When a position requires liquidation, the LendingPool expects the StabilityPool to provide crvUSD tokens:

```solidity
// LendingPool.sol
function finalizeLiquidation(address userAddress) external nonReentrant onlyStabilityPool {
    uint256 userDebt = lendingPool.getUserDebt(userAddress);
    uint256 scaledUserDebt = WadRayMath.rayMul(userDebt, lendingPool.getNormalizedDebt());
    
    // StabilityPool is expected to provide crvUSD here
@>  IERC20(reserve.reserveAssetAddress).safeTransferFrom(msg.sender, reserve.reserveRTokenAddress, amountScaled);
}
```

However, the token flow in the protocol never provides the StabilityPool with crvUSD. Instead:

1. Users deposit crvUSD into the LendingPool, which transfers it to the rToken contract
2. Users receive rTokens representing their deposit
3. Users can deposit these rTokens into StabilityPool and receive deTokens
4. The crvUSD remains locked in the rToken contract with no mechanism for the StabilityPool to access it

The StabilityPool contract has no functions that would allow it to:

* Receive crvUSD directly
* Convert rTokens to crvUSD
* Access the crvUSD held by the rToken contract

## Impact

High - Unhealthy positions cannot be liquidated and the protocol will build up bad debt and eventually become insolvent

## Likelihood

High - Liquidations are an integral part of the system and this will occur every time.

## Recommendations

There are multiple different approaches that can be taken to fix this.

1. Modify the liquidation mechanism to work with rTokens instead of requiring crvUSD
2. Create a new mechanism for the StabilityPool to redeem rTokens for crvUSD specifically for liquidations
3. Restructure the protocol's token flow to ensure the StabilityPool can maintain its own crvUSD reserves for liquidations, separate from the rToken system

## <a id='H-05'></a>H-05 Users Can Overwrite Existing Locks in veRAACToken Resulting in Permanent Loss of Funds            

## Summary

The `veRAACToken.sol` contract allows users to lock RAAC tokens for a specified duration to receive veRAAC tokens (voting power). However, there is a critical vulnerability where users can call the `lock()` function multiple times, overwriting their existing lock position. When this happens, the tokens from the original lock remain in the contract but are disassociated from the user's lock position, resulting in permanent loss of user funds.

## Vulnerability Details

The vulnerability exists in the `lock()` function of the `veRAACToken.sol` contract, which calls into the LockManager library's `createLock()` function. When a user already has an active lock and calls lock() again with a new amount and duration, the function:

1. Takes the new RAAC tokens from the user
2. Completely overwrites the user's existing lock record
3. Keeps the original locked tokens in the contract
4. Does not return the original locked tokens to the user

The root cause is in the LockManager library's createLock() function, which simply overwrites the existing lock record without checking if a user already has tokens locked:

```solidity
function createLock(
    LockState storage state,
    address user,
    uint256 amount,
    uint256 duration
) internal returns (uint256 end) {
    // ...
    
    state.locks[user] = Lock({
        amount: amount,
        end: end,
        exists: true
    });

    state.totalLocked += amount;

    emit LockCreated(user, amount, end);
    return end;
}
```

There is no check in either the veRAACToken contract or the LockManager library to prevent overwriting an existing lock.

## Impact

High - Users can permanently lose all the RAAC tokens they initially locked if they  call `lock()` again instead of using the `increase()` function.
Although an `emergencyWithdraw` can be called, a protocol wide reset occurs which would not be deemed logical. Therefore, the tokens are essentially locked.

## Likelihood

Low - Requires user to interact with `lock()` again after having already locked tokens.

## Severity

High x Low = Medium overall
## Recommendations

implement a check in the lock() function to prevent overwriting existing active locks:

```solidity
function lock(uint256 amount, uint256 duration) external nonReentrant whenNotPaused {
    if (amount == 0) revert InvalidAmount();
    if (amount > MAX_LOCK_AMOUNT) revert AmountExceedsLimit();
    
    // Check if user already has an active lock
    LockManager.Lock memory userLock = _lockState.getLock(msg.sender);
    if (userLock.exists && userLock.end > block.timestamp) {
        revert ExistingActiveLock("Use increase() or extend() to modify your lock");
    }
    
    // Rest of the function...
}
```

## <a id='H-06'></a>H-06 Double Liquidity Index Scaling in RToken.sol Transfer Functions Leads to Incorrect Token Amounts            

## Summary

The `RToken.sol` contract's `transfer()` and `transferFrom()` functions incorrectly scale token amounts twice by the liquidity index when performing transfers. This occurs because both the transfer function and the internal `_update` function independently apply the same scaling operation, resulting in token amounts being divided by the liquidity index twice.

## Vulnerability Details

The issue arises from the interaction between the transfer functions and the `_update` function.

When a transfer is initiated:

$ScaledAmount = Amount / LiquidityIndex$ <= This is Correct

```solidity
function transfer(address recipient, uint256 amount) public override(ERC20, IERC20) returns (bool) {
    uint256 scaledAmount = amount.rayDiv(ILendingPool(_reservePool).getNormalizedIncome());
    return super.transfer(recipient, scaledAmount);
}

```

But then, this scaled amount is passed to `_update` and divided by the Liquidity Index again:

```solidity
function _update(address from, address to, uint256 amount) internal override {
    uint256 scaledAmount = amount.rayDiv(ILendingPool(_reservePool).getNormalizedIncome());
    super._update(from, to, scaledAmount);
}
```

As a result, the user would transfer:

$Amount / LiquidityIndex^2$

## Impact

Low - Transferring incorrect token amounts, significantly less than intended. However, user doesn't lose funds therefore low impact.

## Likelihood

High - This issue would be triggered by every transfer operation, affecting all users of the protocol who attempt to transfer tokens.

## Severity
Low x High = Medium overall

## Recommendations

Remove the scaling operation from the transfer and transferFrom functions, keeping it only in \_update.

## <a id='H-07'></a>H-07 Double Usage Index Scaling in StabilityPool Liquidation Inflates Required CRVUSD Balance            
## Summary

The `StabilityPool.sol::liquidateBorrower()` function incorrectly scales user debt by the usage index twice when checking against the pool's CRVUSD balance. This occurs because the function multiplies an already-normalized debt value by the normalization factor again, artificially inflating the required CRVUSD balance needed for liquidation.

## Vulnerability Details

The scaling error occurs in two steps:

1. Here, the `LendingPool.sol::getUserDebt()` function returns the user debt multiplied by the usage index:

```solidity
uint256 userDebt = lendingPool.getUserDebt(userAddress);
```

1. Thereafter, it is scaled again:

```solidity
uint256 scaledUserDebt = WadRayMath.rayMul(userDebt, lendingPool.getNormalizedDebt());
```

1. It is then used as a check to see if there is enough CRVUSD in the StabilityPool.sol contract to liquidate the user:

```solidity
// Uses double-scaled value for balance check
if (crvUSDBalance < scaledUserDebt) revert InsufficientBalance();
```

What it should be:
$scaledUserDebt = debt * usageIndex$

Current implementation:
$scaledUserDebt = (debt * usageIndex) * usageIndex = debt * usageIndex^2$

## Impact

High - Considering usage index is designed to only go up, liquidating positions is made significantly harder as time passes and eventually, impossible as the funds needed would far surpass the total protocol liquidity.

## Likelihood

High - This will occur for every liquidation attempt through the Stability Pool, making it a systematic issue rather than an edge case.

## Recommendations

* create consistency in usage of scaling, whether you scale through `getUserDebt()` or within `StabilityPool.sol::liquidateBorrower()`.

## <a id='H-08'></a>H-08 Incorrect Collateralization Check in LendingPool Allows Excessive Borrowing            

## Summary

The LendingPool contract contains a critical flaw in its collateral validation logic that allows users to borrow significantly more than their collateral ratio should permit. Due to a reversed comparison in the collateralization check, borrowers can repeatedly borrow amounts that would typically exceed their maximum borrowing capacity, potentially leading to undercollateralized positions and protocol insolvency.

## Vulnerability Details

The issue exists in the `LendingPool.sol::borrow()` function where the collateralization check is implemented incorrectly:

```solidity
if (collateralValue < userTotalDebt.percentMul(liquidationThreshold)) {
    revert NotEnoughCollateralToBorrow();
}

```

This check compares whether the collateral value is less than the scaled debt value (userTotalDebt \* liquidationThreshold), which is the inverse of the correct comparison. The proper validation should ensure that the new total debt after borrowing remains below the maximum allowed by the collateral value.

For example, with a liquidation threshold of 80%, a user with 100,000 in collateral should only be able to borrow up to 80,000;  for simplicity, lets assume usageIndex = 1.0.

However, the current implementation allows users to borrow beyond this limit because the check is reversed, essentially comparing 100,000 < (90,000 \* 0.8) which evaluates to 100,000 < 72,000, allowing the transaction to proceed when it should revert.

Additionally, because a user must be flagged for liquidation through `LendingPool.sol::initiateLiquidation()` function call, the user in one txn can call borrow infinitesimally until the check reaches 100,000, by which point they would have far exceeded their collateral value.

Building from the last example, a user then calls `LendingPool.sol::borrow()` with amount = 15,000.

```solidity
if (isUnderLiquidation[msg.sender]) revert CannotBorrowUnderLiquidation();
```

This check will pass because the user hasn't been flagged for liquidation yet.

```solidity
uint256 userTotalDebt = user.scaledDebtBalance.rayMul(reserve.usageIndex) + amount;
```

`userTotalDebt` == 105,000.

Thereafter, the check will evaluate to:
`100000 < 105000 * 0.8 = 84000`. At this point, they've already taken out a a loan greater than the principle but are able to continue until the check fails.

## Likelihood

High

The vulnerability is easily exploitable as it requires no special conditions or complex setup. Any user with deposited collateral can execute this attack by making multiple borrow calls in the same transaction before liquidation checks occur. The bug exists in a core function that will be frequently used, making it highly likely to be discovered and exploited.

## Impact

High

This vulnerability allows users to:

1. Borrow significantly more than their collateral should permit
2. Potentially drain the protocol's lending pools
3. Lead to bad debt accumulation and protocol insolvency

## Recommendations

The collateralization check should be modified to ensure the new total debt remains under the maximum allowed borrowing power

## <a id='H-09'></a>H-09 Fee rewards are permanently lost for users who claim early due to incorrect tracking of claimed rewards            

## Summary

In the `FeeCollector.sol` contract, users' claimed rewards are incorrectly tracked by setting `userRewards[user]` to the total amount distributed to all users (`totalDistributed`) rather than incrementing it by the amount actually claimed. This causes users who claim early to lose access to future rewards permanently, while late claimers receive disproportionately higher rewards.

## Vulnerability Details

The issue occurs in the `claimRewards` function where `userRewards[user]` is set to `totalDistributed` after a claim:

```solidity
function claimRewards(address user) external override nonReentrant whenNotPaused returns (uint256) {
    uint256 pendingReward = _calculatePendingRewards(user);
    // Reset user rewards before transfer
@>  userRewards[user] = totalDistributed; // @audit-issue Should increment by claimed amount
    raacToken.safeTransfer(user, pendingReward);
    emit RewardClaimed(user, pendingReward);
}
```

When calculating pending rewards:

```solidity
function _calculatePendingRewards(address user) internal view returns (uint256) {
    uint256 userVotingPower = veRAACToken.getVotingPower(user);
    if (userVotingPower == 0) return 0;

    uint256 totalVotingPower = veRAACToken.getTotalVotingPower();
    if (totalVotingPower == 0) return 0;
    
    uint256 share = (totalDistributed * userVotingPower) / totalVotingPower;
@>  return share > userRewards[user] ? share - userRewards[user] : 0;
}
```

Consider two users with equal voting power (250e18 each) when 800 RAAC tokens are distributed:

1. User A claims early:

   * Gets 400 RAAC (their 50% share)
   * `userRewards[A]` is set to 800 (total distributed)

2. Another 800 RAAC distribution occurs:

   * `totalDistributed` is now 1600
   * User A's share calculation: `(1600 * 250/500) = 800`
   * Pending rewards: `800 - 800 = 0` (no rewards despite being entitled to them)

3. User B claims for the first time:

   * Share calculation: `(1600 * 250/500) = 800`
   * Pending rewards: `800 - 0 = 800` (gets full amount because `userRewards[B]` was 0)

## Impact

High - Directly causes loss of rewards that users are entitled to and creates systemic unfairness in reward distribution.

* Users who claim rewards early permanently lose access to future rewards
* Late claimers receive disproportionately higher rewards
* Creates a "last to claim" game theory that undermines the protocol's intended reward distribution

## Likelihood

High - Comes into fruition after first round of rewards.

## Recommendations

* Update the reward tracking to increment by claimed amount.

    
# Medium Risk Findings

## <a id='M-01'></a>M-01 Interest Rate Model Uses Prime Rate Instead of Optimal Rate at Optimal Utilization            

## Summary

The interest rate model in `ReserveLibrary::calculateBorrowRate()` calculates borrowing rates using the prime rate instead of the configured optimal rate,
causing borrowers to pay double the intended interest rate when utilization reaches the optimal point.

## Severity

High

## Vulnerability Details

The `ReserveLibrary` implements a two-slope interest rate model where rates should scale from base rate to optimal rate in the first slope (0-80% utilization), then from optimal rate to max rate in the second slope (80-100% utilization). However, the implementation incorrectly uses the prime rate instead of optimal rate when calculating the first slope.
The optimal rate is explicitly set to 50% of the prime rate during initialization, indicating an intended relationship between these rates. However, in `calculateBorrowRate`, the function uses primeRate in its calculations despite taking optimalRate as a parameter:

```solidity
if (utilizationRate <= optimalUtilizationRate) {
    uint256 rateSlope = primeRate - baseRate;  // Uses primeRate instead of optimalRate
    uint256 rateIncrease = utilizationRate.rayMul(rateSlope).rayDiv(optimalUtilizationRate);
    rate = baseRate + rateIncrease;
}

```

This causes the first slope to target the prime rate instead of the optimal rate at the optimal utilization point, creating a steeper rate curve than intended in normal market conditions.

This is proven to be unintentional behaviour from the [documentation](https://github.com/Cyfrin/2025-02-raac/blob/main/docs/core/libraries/pools/ReserveLibrary.md).

> \| optimalRate |  uint256     | Rate at optimal utilization in RAY |

And also this [blogpost](https://blog.raac.io/posts/raac-introduces-a-new-era-of-real-estate-in-defi)

> The protocol offers predictable borrowing rates, with interest rates soft-pegged to half of US Prime Rate

## Impact

High impact:

1. Protocol Design Impact:

* Breaks core interest rate model assumptions
* Nullifies the purpose of having an optimal rate parameter
* Documentation and implementation mismatch on fundamental mechanism

1. Economic Impact:

* Borrowers consistently pay double the intended interest rate
* Distorts the intended economic incentives of the two-slope model

## Likelihood

High likelihood:

* Triggered on every borrow transaction when utilization approaches optimal level
* Built into the core interest rate calculation
* Affects all markets and all users equally
## Recommendations

Modify the rate calculation in ReserveLibrary.sol to use optimalRate instead of primeRate:

```diff
function calculateBorrowRate(
    uint256 primeRate,
    uint256 baseRate,
    uint256 optimalRate,
    uint256 maxRate,
    uint256 optimalUtilizationRate,
    uint256 utilizationRate
) internal pure returns (uint256) {
    if (utilizationRate <= optimalUtilizationRate) {
-       uint256 rateSlope = primeRate - baseRate;
+       uint256 rateSlope = optimalRate - baseRate;
        uint256 rateIncrease = utilizationRate.rayMul(rateSlope).rayDiv(optimalUtilizationRate);
        rate = baseRate + rateIncrease;
    } else {
        uint256 excessUtilization = utilizationRate - optimalUtilizationRate;
        uint256 maxExcessUtilization = WadRayMath.RAY - optimalUtilizationRate;
-       uint256 rateSlope = maxRate - primeRate;
+       uint256 rateSlope = maxRate - optimalRate;
        uint256 rateIncrease = excessUtilization.rayMul(rateSlope).rayDiv(maxExcessUtilization);
        rate = primeRate + rateIncrease;
    }
return rate;
}
```

## <a id='M-02'></a>M-02 Incorrect Emission Rate Used for Past Block Rewards            

## Summary

The `RAACMinter.sol::tick()` function updates the emission rate before calculating and minting tokens for past blocks. This causes tokens to be minted using the new emission rate for blocks that should have used the previous rate, leading to incorrect token emissions.

## Vulnerability Details

In RAACMinter.sol, the tick() function performs two main operations:

1. Updates the emission rate if the update interval has passed
2. Mints tokens for blocks that have passed since the last update

The issue is that after updating the emission rate, it uses this new rate to calculate emissions for blocks that occurred during the previous period. These past blocks should be calculated using the rate that was in effect during their period.

```solidity
function tick() external nonReentrant whenNotPaused {
    // First updates emission rate
    if (emissionUpdateInterval == 0 || block.timestamp >= lastEmissionUpdateTimestamp + emissionUpdateInterval) {
@>      updateEmissionRate();
    }
    
    // Then uses new rate to calculate past emissions
    uint256 currentBlock = block.number;
    uint256 blocksSinceLastUpdate = currentBlock - lastUpdateBlock;
    if (blocksSinceLastUpdate > 0) {
@>      uint256 amountToMint = emissionRate * blocksSinceLastUpdate;
        // minting logic...
    } 
}

```

For example:

* Day 1: Rate is 1000 tokens/block
* Day 2: Rate changes to 1100 tokens/block
* When tick() is called on Day 2, it will mint (1100 \* blocks\_in\_day1) tokens for Day 1's blocks, instead of (1000 \* blocks\_in\_day1)

## Impact

* If emission rate increases: Past blocks receive more tokens than intended
* If emission rate decreases: Past blocks receive fewer tokens than intended

## Likelihood

High - This will occur every time the emission rate changes and tick() is called, which is a core function of the protocol.

## Recommendations

Modify the tick() function to:

1. Calculate and mint rewards for past blocks using the old emission rate
2. Then update the emission rate for future blocks

Thereafter, it is important to consider that emergencyShutdown() resets emissionRate to 0, therefore a check for after emrgencyShutdown() being called should be placed to update() the emissionRate so no rewards are missed on day 1 after the protocol restart.

## <a id='M-03'></a>M-03 Incorrect Vault Share Accounting Leads to Inaccurate Liquidity Management            

## Summary

The `LendingPool.sol` contract incorrectly tracks deposits in the Curve crvUSD vault by recording the deposited crvUSD amount rather than the received vault shares. Since vault shares appreciate in value as the vault generates yield, this creates a growing discrepancy between the tracked and actual value of vault deposits, leading to potential issues with liquidity management and withdrawal handling.

## Vulnerability Details

The `LendingPool.sol` implements a liquidity management system where excess crvUSD is deposited into a Curve vault, tracked through the totalVaultDeposits variable. However, when interacting with the vault, the contract fails to account for the share-based mechanics of vault tokens:

```solidity
function _depositIntoVault(uint256 amount) internal {
    IERC20(reserve.reserveAssetAddress).approve(address(curveVault), amount);
    curveVault.deposit(amount, address(this));
    totalVaultDeposits += amount;  // Tracks crvUSD amount instead of shares
}
```

The vault converts deposits to shares using this [formula](https://github.com/curvefi/scrvusd/blob/95a120847c7a2901cea5256ba081494e18ea5315/contracts/yearn/VaultV3.vy#L479):

`shares = assets * (total_supply / total_assets)`

As the vault generates yield, `total_assets` increases while `total_supply` remains constant, meaning each share becomes worth more crvUSD over time. The contract's tracking of crvUSD amounts rather than shares fails to account for this appreciation.

This issue is particularly evident in the withdrawal function:

```solidity
function _withdrawFromVault(uint256 amount) internal {
    curveVault.withdraw(amount, address(this), msg.sender, 0, new address[](0));
    totalVaultDeposits -= amount;  // Incorrectly deducts crvUSD amount instead of shares
}
```

When withdrawing, the contract deducts the crvUSD amount from totalVaultDeposits, but this amount no longer accurately represents the shares being consumed due to yield appreciation.

Additionally, Protocol accounting and financial reporting will be inaccurate and this will lead to issues; the extra generated yield should be redistributed into the protocol to various different categories (RToken, governance, team etc) but the shares are never recorded virtually.

## Impact

High:

1. The liquidity buffer ratio becomes inaccurate over time as the true value of vault deposits grows higher than tracked
2. Rebalancing operations may make incorrect decisions based on understated vault value
3. The contract might fail to maintain adequate liquidity buffers, potentially leading to failed withdrawals
4. Protocol accounting and financial reporting inaccuracies.

## Likelihood

Medium - The issue manifests gradually as yield accrues in the vault and the impact depends on yield rates and deposit duration. The problem compounds over time, becoming more severe as the vault generates returns.

## Recommendations

Use a virtual balance to track shares. Many subsequent changes need to take place for system accounting but this is imperative since the protocol is integrating heavily with crvUSD vaults and seemlessly depositing and withdrawing from it.

## <a id='M-04'></a>M-04 RWAGauge Sequencer Downtime Period Misalignment            

## Summary

The `RWAGauge.sol` contract calculates new period start times using absolute timestamp alignment, which can cause period misalignment with real-world asset payment cycles when L2 sequencer downtime occurs. This misalignment disrupts the synchronization between on-chain reward distributions and real-world rent collection cycles.

## Vulnerability Details

The RWAGauge contract handles 30-day periods for real-world asset yield distribution. The period transition mechanism in BaseGauge.sol calculates the next period start time using:

```solidity
uint256 nextPeriodStart = ((currentTime / periodDuration) + 2) * periodDuration;
```

When sequencer downtime occurs near a period transition, the contract calculates the new period start by rounding to the next absolute 30-day boundary from timestamp 0. This creates a permanent misalignment between protocol periods and real-world payment cycles.

For example:

1. Period 1 starts January 1st
2. Period should transition on January 31st
3. Sequencer goes down on January 30th for 5 days
4. When sequencer resumes on February 4th, next period starts February 5th
5. All subsequent periods now start on the 5th instead of the 1st

The root cause is that the period calculation uses absolute timestamp alignment rather than maintaining relative 30-day increments from the previous period.

## Impact

Medium - Protocol reward distributions become permanently misaligned with real-world rent collection cycles

## Likelihood

Low - requires L2 sequencer to go down and timings to align,

## Recommendations

Consider implementing these changes:

1. Calculate next period based on previous period rather than absolute alignment:

```solidity
uint256 nextPeriodStart = periodState.periodStartTime + getPeriodDuration();
```

1. Add emergency realignment capability:

```solidity
function realignPeriod(uint256 targetTimestamp) external onlyRole(EMERGENCY_ADMIN) {
    require(targetTimestamp > block.timestamp, "Invalid target");
    periodState.periodStartTime = targetTimestamp;
}
```

## <a id='M-05'></a>M-05 Liquidation Grace Period Can Be Unfairly Shortened By Protocol Pause            

## Summary

The `LendingPool.sol` contract uses a grace period to allow users to repay their debt after liquidation is initiated. However, if the protocol is paused during this grace period, users lose their opportunity to repay since all functions including liquidation-related ones are blocked by the `whenNotPaused` modifier. This can lead to unfair liquidations as users are prevented from saving their positions during the pause duration.

## Vulnerability Details

The `LendingPool.sol` contract implements a liquidation system with a grace period where users can repay their debt to avoid liquidation. However, all functions including `repay()` and `closeLiquidation()` have the `whenNotPaused` modifier. If the protocol is paused during a user's grace period and remains paused for a duration that exceeds the remaining grace period, the user loses their chance to repay their debt and save their position.


So a user can be liquidated during this down period.

## Impact

High - User is forced into liquidation resulting in loss of funds.

## Likelihood

Low - Deemed low because it would require the protocol to be paused for a time period greater than the grace period and a user to be initiated into liquidation  before protocol being paused.

## Severity

High x Low = Medium

## Proof of Concept

1. User position becomes eligible for liquidation
2. Liquidation is initiated with 3-day grace period starting
3. After 1 day, protocol is paused for emergency
4. Pause lasts for 3 days
5. When protocol unpauses, user's grace period has expired
6. User can no longer repay and save their position
7. Position becomes forcefully liquidated

## Recommendations

Few approaches:

1. Add pause duration compensation
2. Create separate emergency pause levels that don't affect liquidation functions

## <a id='M-06'></a>M-06 RToken.sol:: transferFrom() and transfer() use inconsistent liquidity index sources leading to incorrect token transfers"            

## Summary

The `RToken.sol` contract uses inconsistent methods to obtain the liquidity index between its `transfer()` and `transferFrom()` functions. While `transfer()` uses the current liquidity index from the LendingPool, `transferFrom()` uses a stored `_liquidityIndex` value that must be manually updated. This inconsistency can lead to incorrect token amounts being transferred if the stored index becomes stale.

## Vulnerability Details

The RToken contract implements an interest-bearing token where token balances increase over time based on accrued interest, represented by a liquidity index. When transferring tokens, the actual transfer amount needs to be scaled by this index. However, the contract uses two different approaches to obtain this scaling factor:

1. In `transfer()`:

```solidity
function transfer(address recipient, uint256 amount) public override(ERC20, IERC20) returns (bool) {
@>  uint256 scaledAmount = amount.rayDiv(ILendingPool(_reservePool).getNormalizedIncome());
    return super.transfer(recipient, scaledAmount);
}
```

1. in `transferFrom()`:

```solidity
function transferFrom(address sender, address recipient, uint256 amount) public override(ERC20, IERC20) returns (bool) {
@>  uint256 scaledAmount = amount.rayDiv(_liquidityIndex);
    return super.transferFrom(sender, recipient, scaledAmount);
}
```

The `_liquidityIndex` is only updated when the LendingPool calls `updateLiquidityIndex()`. If this update is not called frequently enough, the stored index will become stale compared to the actual current index from `getNormalizedIncome()`.

## Impact

High

* Users transferring the same amount of tokens through transfer() vs transferFrom() could end up transferring different underlying amounts due to the index discrepancy
* If `_liquidityIndex` becomes stale and lower than the current index, transfer() will transfer less tokens than intended

## Likelihood

Medium

* The likelihood depends on how frequently updateLiquidityIndex() is called by the LendingPool
* The issue exists in the code and requires no special conditions to manifest
* Any period of time where the indices diverge creates an exploitable condition

## Recommendations

Always use the current liquidity index from LendingPool for both functions:

## <a id='M-07'></a>M-07 ReservePool fails to implement updateLiquidityIndex function, permanently freezing RToken's transferFrom() scaling at initial value"            

## Summary

The `ReservePool.sol` contract fails to implement the `updateLiquidityIndex` function that is required to update the `_liquidityIndex` parameter in `RToken.sol`. This causes `RToken::transferFrom()` function to permanently use the initial liquidity index value set at deployment, preventing proper interest accrual scaling for these transfers.

## Vulnerability Details

The RToken contract relies on the ReservePool to update its liquidity index through the updateLiquidityIndex function:

```solidity
// RToken.sol
function updateLiquidityIndex(uint256 newLiquidityIndex) external override onlyReservePool {
    if (newLiquidityIndex < _liquidityIndex) revert InvalidAmount();
    _liquidityIndex = newLiquidityIndex;
    emit LiquidityIndexUpdated(newLiquidityIndex);
}
```

However, the ReservePool contract does not implement this function at all. This means:

The \_liquidityIndex in RToken remains at its initial value forever
All transferFrom() calls use this stale initial value for scaling:

```solidity
function transferFrom(address sender, address recipient, uint256 amount) public override(ERC20, IERC20) returns (bool) {
    uint256 scaledAmount = amount.rayDiv(_liquidityIndex); // _liquidityIndex never updates!
    return super.transferFrom(sender, recipient, scaledAmount);
}
```

## Impact

High

* The transferFrom() function will ALWAYS use the initial liquidity index value
* No interest accrual will be reflected in transferFrom() operations
* Users will receive incorrect token amounts when using transferFrom()

## Likelihood

High

* The issue is present from deployment
* Affects all transferFrom() operations
* No special conditions needed to trigger
* Will definitely lead to incorrect transfers as soon as any interest accrues

## Recommendations

1. Implement `updateLiquidityIndex()` in `ReservePool.sol`:
2. Ensure the function is called whenever interest accrues in the ReservePool
3. Alternatively, use this functionality instead: `ILendingPool(_reservePool).getNormalizedIncome()`

## <a id='M-08'></a>M-08 GaugeController.sol uses incorrect voting power metric leading to compromised voting weight            

## Summary

The `GaugeController.sol::vote()` function uses `veRAACToken.balanceOf()` to determine a user's voting power instead of `veRAACToken.getVotingPower()`. This bypasses the core vote-escrow mechanism where voting power should be weighted by both the amount of tokens locked and the lock duration. Additionally, user whose locks have expired can continue to vote on key protocol decision despite having zero voting power.

## Vulnerability Details

In the GaugeController contract, the vote() function determines voting influence using:

```solidity
    function vote(address gauge, uint256 weight) external override whenNotPaused {
        if (!isGauge(gauge)) revert GaugeNotFound();
        if (weight > WEIGHT_PRECISION) revert InvalidWeight();
        
->      uint256 votingPower = veRAACToken.balanceOf(msg.sender);
        if (votingPower == 0) revert NoVotingPower();

        uint256 oldWeight = userGaugeVotes[msg.sender][gauge];
        userGaugeVotes[msg.sender][gauge] = weight;
        
        _updateGaugeWeight(gauge, oldWeight, weight, votingPower);
        
        emit WeightUpdated(gauge, oldWeight, weight);
    }
```

However, this only retrieves the raw balance of locked tokens without considering the lock duration. The protocol implements a vote-escrow model similar to Curve's veCRV where voting power should be a function of both:

* Amount of tokens locked
* Duration of the lock (longer locks = more voting power)

The correct voting power should be obtained through veRAACToken.getVotingPower() which properly calculates the time-weighted voting power using the VotingPowerLib:

```solidity
function getVotingPower(address account) public view returns (uint256) {
    return _votingState.getCurrentPower(account, block.timestamp);
}
```

Additionally, users can keep voting once their locks have expired as the balance indicates they have voting power when they should have zero.

```solidity
        uint256 votingPower = veRAACToken.balanceOf(msg.sender);
->      if (votingPower == 0) revert NoVotingPower();
```

## Impact

High:

* Users receive voting weight solely based on their locked token amount, ignoring lock duration
* This breaks the vote-escrow incentive mechanism where longer lock commitments should grant greater voting influence
* Users who have zero voting power can keep impacting the protocol once their locks have expired

## Likelihood

High - Happens on every vote, not preconditions required.

## Recommendations

Replace `veRAACToken.balanceOf(msg.sender)` with `veRAACToken.getVotingPower(msg.sender)` in the `vote()` function:

```diff
function vote(address gauge, uint256 weight) external override whenNotPaused {
    if (!isGauge(gauge)) revert GaugeNotFound();
    if (weight > WEIGHT_PRECISION) revert InvalidWeight();
    
-   uint256 votingPower = veRAACToken.balanceOf(msg.sender);
+   uint256 votingPower = veRAACToken.getVotingPower(msg.sender);
    if (votingPower == 0) revert NoVotingPower();
    
    uint256 oldWeight = userGaugeVotes[msg.sender][gauge];
    userGaugeVotes[msg.sender][gauge] = weight;
    
    _updateGaugeWeight(gauge, oldWeight, weight, votingPower);
    
    emit WeightUpdated(gauge, oldWeight, weight);
}
```

## <a id='M-09'></a>M-09 Emergency Revocation Results in Permanent Token Lock            

## Summary

The `RAACReleaseOrchestrator::emergencyRevoke()` function, designed for emergency revocation of vested tokens, mistakenly transfers the tokens to itself (address(this)) without any mechanism to recover or redistribute these tokens, resulting in a permanent reduction of the token supply.

## Vulnerability Details

The issue exists in the `emergencyRevoke` function:

```solidity
function emergencyRevoke(address beneficiary) external onlyRole(EMERGENCY_ROLE) {
    VestingSchedule storage schedule = vestingSchedules[beneficiary];
    if (!schedule.initialized) revert NoVestingSchedule();
    
    uint256 unreleasedAmount = schedule.totalAmount - schedule.releasedAmount;
    delete vestingSchedules[beneficiary];
    
    if (unreleasedAmount > 0) {
@>      raacToken.transfer(address(this), unreleasedAmount);
        emit EmergencyWithdraw(beneficiary, unreleasedAmount);
    }
    
    emit VestingScheduleRevoked(beneficiary);
}
```

The contract has predefined token allocations for different categories:

```Solidity
categoryAllocations[TEAM_CATEGORY] = 18_000_000 ether;        // 18%
categoryAllocations[ADVISOR_CATEGORY] = 10_300_000 ether;     // 10.3%
categoryAllocations[TREASURY_CATEGORY] = 5_000_000 ether;     // 5%
categoryAllocations[PRIVATE_SALE_CATEGORY] = 10_000_000 ether;// 10%
categoryAllocations[PUBLIC_SALE_CATEGORY] = 15_000_000 ether; // 15%
categoryAllocations[LIQUIDITY_CATEGORY] = 6_800_000 ether;    // 6.8%
```

When `emergencyRevoke` is called, it calculates the unreleased tokens and transfers them to the contract itself. However, the contract lacks any functionality to transfer these tokens out or redistribute them, effectively locking them permanently.

## Impact

High:

* Tokens are permanently locked, reducing the total circulating supply
* The intended token distribution percentages are disrupted

## Likelihood

Low - Emergency revokes do not happen regularly.

## Recommendations

Implement a recovery mechanism for  locked tokens

## <a id='M-10'></a>M-10 RToken.sol::calculateDustAmount() unable to to execute due to incorrect implementation, resulting in dust stuck in contract indefinitely            

## Summary

The `calculateDustAmount()` function in the RToken contract performs inconsistent scaling operations when comparing CRVUSD balances to RToken supply, leading to incorrect dust calculations. The function scales down the actual CRVUSD balance while simultaneously double-scaling the RToken supply due to an existing scaling in `totalSupply()`. As a result, The dust accumulated in the contract can never be retrieved.

## Vulnerability Details

The dust calculation attempts to compare the contract's CRVUSD balance with the total RToken supply to determine excess funds. However, the implementation contains two critical scaling issues:

```solidity
// Incorrectly scales down actual CRVUSD balance
uint256 contractBalance = IERC20(_assetAddress).balanceOf(address(this)).rayDiv(ILendingPool(_reservePool).getNormalizedIncome());

// Gets already-scaled total supply
uint256 currentTotalSupply = totalSupply();

// Applies another scaling, resulting in double scaling
uint256 totalRealBalance = currentTotalSupply.rayMul(ILendingPool(_reservePool).getNormalizedIncome());
```

1. It compares the scaled down CRVUSD balance to the scaled up RToken balance.
2. It double scales the RToken balance as the implementation `totalSupply()` scales the totalSupply by the liquidityIndex and then the function itself scales this value by the liquidityIndex again

The function checks: `contractBalance <= totalRealBalance ? 0 : contractBalance - totalRealBalance`

but the current implementation always evaluates this to:

$CRVUSD/LiquidityIndex \le RToken_{total} * LiquidityIndex^2$

Which will always return true, thus the function `return 0`.

`transferAccruedDust()` will always revert with `noDust()`:

```solidity
        uint256 poolDustBalance = calculateDustAmount();
@>      if(poolDustBalance == 0) revert NoDust();
```

Additionally, `rescueToken()` cannot be used to collect the dust due to this check as CRVUSD is the main asset:

```solidity
    function rescueToken(address tokenAddress, address recipient, uint256 amount) external onlyReservePool {
        if (recipient == address(0)) revert InvalidAddress();
@>      if (tokenAddress == _assetAddress) revert CannotRescueMainAsset();
```

## Impact

High - Dust accumulation is completely irretrievable.

## Likelihood

High - Current implementation means it will always happen.

## Recommendations

1. Ensure either the raw CRVUSD is being compared to the `RToken * Liquidity` supply or the inverse.
2. Fix the double scaling

## <a id='M-11'></a>M-11 Incorrect `RAACMinter.sol::calculateNewEmissionRate()` Due to Index vs Balance Mismatch            

## Summary

The RAACMinter contract's utilization rate calculation incorrectly compares a RAY-scaled (1e27) usage index against WAD-scaled (1e18) token balances, resulting in massively inflated utilization rates that can reach millions of percent that affect emission calculations and overall protocol economics.

## Vulnerability Details

The issue lies in the getUtilizationRate() function in RAACMinter.sol where the calculation uses mismatched values:

```solidity
function getUtilizationRate() internal view returns (uint256) {
@>  uint256 totalBorrowed = lendingPool.getNormalizedDebt();
@>  uint256 totalDeposits = stabilityPool.getTotalDeposits();
    if (totalDeposits == 0) return 0;
    return (totalBorrowed * 100) / totalDeposits;
}
```

The flow of the issue:

1. getNormalizedDebt() returns the usage index which:

* Starts at 1e27 (RAY precision)
* Grows with interest accrual

1. getTotalDeposits() returns actual token amounts in 1e18 precision
2. The calculation multiplies by 100 before division, further amplifying the precision error
3. Results in utilization rates in the millions of percent

This utilization rate is then used in `calculateNewEmissionRate()` as an important check:

```solidity
function calculateNewEmissionRate() internal view returns (uint256) {
@>  uint256 utilizationRate = getUtilizationRate();
    uint256 adjustment = (emissionRate * adjustmentFactor) / 100;

@>  if (utilizationRate > utilizationTarget) {
```

Output from test:

```Solidity
Logs:
  Initial Usage Index: 1000000000000000000000000000
  Final Usage Index: 1410226029899202994928355252
  Total borrowed:  1410226029899202994928355252
  Total deposited:  51718750000000000000000
  StabilityPool.sol::calculateUtilizationRate result:  2726721
```

## Impact

High

* Utilization rates consistently report >1000x the actual utilization
* Will force emission rates to maximum values due to perceived over-utilization
* Economic model becomes dysfunctional as emissions cannot properly respond to true utilization

## Likelihood

High - The issue will consistently occur as it's a fundamental calculation error in the core economic logic.
## Recommendations

The issue stems from the way the utiliazation rate is calculated; even if the precision was 'fixed' in the current implementation, you would then get a small numerator / total deposits so the inverse problem would occur.

Current implementation divides a scalar index by the total number of deposits:

$Usage Index / RToken_{total supply} * 100$

The utilization rate should be calculated by:

$RToken_{stability pool} / RToken_{total supply} * 100$

## <a id='M-12'></a>M-12. BaseGauge.sol Uses Raw Balance Instead of Voting Power for Direction Voting            

## Summary

the `BaseGauge.sol::voteDirection()` function incorrectly uses raw veToken balance instead of actual voting power for weighting user votes, undermining the vote-escrow tokenomics for directional voting. Additionally, users whose locks have expired can continue to keep voting on key protocol decisions despite having zero voting power.

## Vulnerability Details

In `BaseGauge.sol::voteDirection()` function determines voting influence using:

```solidity
function voteDirection(uint256 direction) public whenNotPaused updateReward(msg.sender) {
    if (direction > 10000) revert InvalidWeight();
    
->  uint256 votingPower = IERC20(IGaugeController(controller).veRAACToken()).balanceOf(msg.sender);
    if (votingPower == 0) revert NoVotingPower(); Should revert if the user has no voting power
    
    totalVotes = processVote(
        userVotes[msg.sender],
        direction,
        votingPower,
        totalVotes
    );
    emit DirectionVoted(msg.sender, direction, votingPower);
}
```

The function uses the raw `balanceOf()` value from the `veRAACToken.sol` contract instead of getting the proper time-weighted voting power.

Additionally, if a user's lock has expired (therefore they should have 0 voting power), but they do not withdraw their veRaacTokens, they can still vote in the protocol.

```solidity
  uint256 votingPower = IERC20(IGaugeController(controller).veRAACToken()).balanceOf(msg.sender);
->  if (votingPower == 0) revert NoVotingPower(); Should revert if the user has no voting power
```

Although this is identical to the bug found in `GaugeController.sol`, This is a separate issue as it affects directional voting rather than gauge weight voting.

## Impact

High-

* Direction voting weights ignore lock duration, breaking vote-escrow incentives
* Users with short-term locks have disproportionate influence over directional votes
* Users can continue to impact the protocol after their voting power has ended

## Likelihood

High - Affects every directional vote cast through BaseGauge implementations.

## Recommendations

Use the `getVotingPower()` function instead:

```diff
function voteDirection(uint256 direction) public whenNotPaused updateReward(msg.sender) {
    if (direction > 10000) revert InvalidWeight();
    
-  uint256 votingPower = IERC20(IGaugeController(controller).veRAACToken()).balanceOf(msg.sender);
+  uint256 votingPower = IERC20(IGaugeController(controller).veRAACToken()).getVotingPower(msg.sender);
    if (votingPower == 0) revert NoVotingPower(); Should revert if the user has no voting power
    
    totalVotes = processVote(
        userVotes[msg.sender],
        direction,
        votingPower,
        totalVotes
    );
    emit DirectionVoted(msg.sender, direction, votingPower);
}
```


# Low Risk Findings

## <a id='L-01'></a>L-01 Extended Recovery Period After Emergency Shutdown Due to Rate Adjustment Limitations            
## Summary

The RAACMinter contract's emergency shutdown mechanism, while successfully halting emissions, leads to an extended period of economic distortion during recovery. Due to the interaction between the benchmark rate and the maximum daily adjustment factor, it can take up to 48 days for emission rates to normalize after an emergency shutdown, resulting in significant deviations from intended token distribution.

## Vulnerability Details

The emergencyShutdown function resets the emission rate to 0:

```solidity
function emergencyShutdown(bool updateLastBlock, uint256 newLastUpdateBlock) external onlyRole(DEFAULT_ADMIN_ROLE) {
@>  emissionRate = 0;
    ...
```

When operations resume, the emission rate gets set to the benchmark rate through `calculateNewEmissionRate()`:

```solidity
function calculateNewEmissionRate() internal view returns (uint256) {
    uint256 utilizationRate = getUtilizationRate();
    uint256 adjustment = (emissionRate * adjustmentFactor) / 100; // 5% adjustment limit

        if (utilizationRate > utilizationTarget) {
            uint256 increasedRate = emissionRate + adjustment;
@>          uint256 maxRate = increasedRate > benchmarkRate ? increasedRate : benchmarkRate;
            return maxRate < maxEmissionRate ? maxRate : maxEmissionRate;
        } else if (utilizationRate < utilizationTarget) {
            uint256 decreasedRate = emissionRate > adjustment ? emissionRate - adjustment : 0;
@>          uint256 minRate = decreasedRate < benchmarkRate ? decreasedRate : benchmarkRate;
            return minRate > minEmissionRate ? minRate : minEmissionRate;
        }
        return emissionRate;
}
```

The issue manifests in two scenarios:

Low-to-High Recovery:

* Initial rate: 100 RAAC/day (minimum)
* After shutdown and restart: 1000 RAAC/day (benchmark)
* With 5% maximum daily adjustment: Takes 48 days to return to minimum rate
* Result: 48 days of excess emissions

High-to-Low Recovery:

* Initial rate: 2000 RAAC/day (example)
* After shutdown and restart: 1000 RAAC/day (benchmark)
* With 5% maximum daily adjustment: Takes 15 days to return to original rate
* Result: 15 days of reduced emissions

## Impact

High - Extended periods of incorrect emissions (15-48 days depending on scenario)

In the low-to-high scenario:

Over 48 days:
$Excess\ Emissions = Σ(1000 - (100 + Daily\ Adjustment))$

Approximately 21,600 excess RAAC tokens emitted

In the high-to-low scenario:

Over 15 days:
$Missing\ Emissions = Σ(2000 - (1000 + Daily\ Adjustment))$

Approximately 7,500 RAAC tokens under-emitted

## Likelihood

Low - Emergency shutdowns are rare and severity depends on $Δ(Benchmark - Emission\ Rate)$.

## Recommendations

* Add a recovery mode that bypasses the adjustment factor limit
* Alternatively, store the pre-shutdown emission rate and allow it to be restored
* At minimum, document this behavior in the protocol specification and consider if the extended recovery period aligns with emergency response requirements.

## <a id='L-02'></a>L-02 Incorrect Voting Power Reporting in `veRAACToken.sol::getLockPosition` Function            
## Summary

The `veRAACToken.sol::getLockPosition()` function  incorrectly reports a user's voting power by returning their raw veRAAC token balance instead of calculating the time-decayed voting power. This creates a discrepancy between the reported voting power and the actual voting power used in governance operations.

## Vulnerability Details

The veRAAC token system implements a Curve-style voting mechanism where voting power decays linearly over time until the lock expiry. The `getLockPosition()` function returns balanceOf(account) as the power value, which doesn't account for the time-decay calculation that governs actual voting influence in the system.

```solidity
function getLockPosition(address account) external view override returns (LockPosition memory) {
    LockManager.Lock memory userLock = _lockState.getLock(account);
    return LockPosition({
        amount: userLock.amount,
        end: userLock.end,
@>      power: balanceOf(account) // Incorrect: returns raw token balance instead of time-decayed power
    });
}
```

## Impact

Low - This inconsistency primarily affects UI/frontend applications and users monitoring their voting power through this function. It could cause users to see incorrect information about their current voting influence, but doesn't affect the actual governance mechanisms since those use the proper time-decayed calculations directly.

## Recommendations

For consistency and clarity, modify the getLockPosition function to correctly return the time-decayed voting power:

```solidity
function getLockPosition(address account) external view override returns (LockPosition memory) {
    LockManager.Lock memory userLock = _lockState.getLock(account);
    return LockPosition({
        amount: userLock.amount,
        end: userLock.end,
@>      power: _votingState.getCurrentPower(account, block.timestamp)
    });
}
```

## <a id='L-03'></a>L-03 Storage Inconsistency in veRAACToken's getLockEndTime Function            

## Summary

The `veRAACToken.sol` contract contains a storage inconsistency between two separate state variables that track lock information. The contract uses `_lockState` (with the LockManager library) to manage and update lock data, but the `getLockEndTime()` function incorrectly reads from a different, non-updated mapping (`locks`), causing it to return incorrect values (specifically, returning `0` instead of the actual lock end time).

## Vulnerability Details

The contract maintains two separate data structures for tracking lock information:

A direct mapping:

```solidity
mapping(address => Lock) public locks;
```

A structured storage variable with the LockManager library:

```solidity
LockManager.LockState private _lockState;
```

When a user creates a lock via the lock() function, the contract correctly updates the \_lockState variable:

```solidity
_lockState.createLock(msg.sender, amount, duration);
```

However, the direct locks mapping is never updated. This becomes problematic when the getLockEndTime() function is called:

```solidity
function getLockEndTime(address account) external view returns (uint256) {
    return locks[account].end;
}
```

This function reads from the locks mapping rather than from \_lockState, resulting in it always returning 0 (the default value) instead of the actual lock end time.

## Impact

This inconsistency leads to incorrect data being returned from the getLockEndTime() function, which could mislead users or integrated protocols that rely on this function to determine when locks expire. However, since the `getLockPosition().end` works correctly and provides the same information, and the getLockEndTime() function does not appear to be used in critical protocol logic, the practical impact is minimal. This is a low severity issue that represents a correctness bug rather than a security vulnerability.

## Recommendations

Modify the getLockEndTime function to read from \_lockState:

```solidity
function getLockEndTime(address account) external view returns (uint256) {
    return _lockState.locks[account].end;
}
```

## <a id='L-04'></a>L-04 Incorrect Fee Type Initialization Causes Fee Updates to Break and Charges 10x Intended Fee Rate            

## Summary

The `FeeCollector.sol` contract initializes two fee types (Swap Tax and NFT Royalty) with incorrect basis points values that sum to 20% instead of the intended 2%. Additionally, the `updateFeeType` function enforces that all fee type parameters must sum to 100% (10000 basis points), making it impossible to update these specific fee types after initialization.

## Vulnerability Details

In `_initializeFeeTypes`, two fee types are initialized with incorrect values:

```solidity
// Buy/Sell Swap Tax (intended 2% total but actually 20%)
feeTypes[6] = FeeType({
    veRAACShare: 500,       // 500/10000 = 5%
    burnShare: 500,         // 500/10000 = 5%
    repairShare: 1000,      // 1000/10000 = 10%
    treasuryShare: 0
});

// NFT Royalty Fees (intended 2% total but actually 20%)
feeTypes[7] = FeeType({
    veRAACShare: 500,       // 500/10000 = 5%
    burnShare: 0,
    repairShare: 1000,      // 1000/10000 = 10%
    treasuryShare: 500      // 500/10000 = 5%
});
```

However, `updateFeeType` enforces that all parameters must sum to BASIS\_POINTS (10000):

```solidity
if (newFee.veRAACShare + newFee.burnShare + newFee.repairShare + newFee.treasuryShare != BASIS_POINTS) {
    revert InvalidDistributionParams();
}
```

This creates two issues:

1. The fees are charging 20% instead of the documented 2%
2. These fee types cannot be updated through `updateFeeType` since they're intended to sum to 200 basis points (2%) but the function requires 10000 basis points (100%)

## Impact

High:

* Users are charged 10x the intended fee rate (20% vs 2%)
* Protocol operators cannot update these fee types through normal governance mechanisms

## Likelihood

High:

* Every swap transaction (fee type 6)
* Every NFT royalty collection (fee type 7)
* Any attempt to update these fee types

## Recommendations

1. Fix `_initializeFeeTypes` to the intended fee amount:

```solidity
// Correct the initialization values for 2% total fee:
feeTypes[6] = FeeType({
    veRAACShare: 50,      // 0.5%
    burnShare: 50,        // 0.5%
    repairShare: 100,     // 1.0%
    treasuryShare: 0
});

feeTypes[7] = FeeType({
    veRAACShare: 50,      // 0.5%
    burnShare: 0,
    repairShare: 100,     // 1.0%
    treasuryShare: 50     // 0.5%
});
```

1. Modify `updateFeeType` to accept a target total for each fee type rather than enforcing 100%; this will allow more flexible changes and also allow updates to the swap fee and NFT royalty distributions:

```solidity
function updateFeeType(
    uint8 feeType, 
    FeeType calldata newFee,
    uint256 targetTotal
) external {
    if (newFee.veRAACShare + newFee.burnShare + newFee.repairShare + newFee.treasuryShare != targetTotal) {
        revert InvalidDistributionParams();
    }
    // rest of function...
}
```

## <a id='L-05'></a>L-05. Storage Inconsistency in veRAACToken's getLockedBalance Function            

## Summary

The `veRAACToken.sol` contract exhibits a storage inconsistency in the `getLockedBalance()` function. This function incorrectly reads lock data from a mapping that is never updated during lock operations, causing it to return `0` instead of the actual locked token amount. This creates a discrepancy where `getLockPosition()` returns the correct locked amount while `getLockedBalance()` always returns `0`.

## Vulnerability Details

The `veRAACToken.sol` contract maintains two separate storage structures for tracking lock information:

1. A direct mapping defined in the contract:

```solidity
mapping(address => Lock) public locks;
```

1. A structured storage variable that uses the LockManager library:

```solidity
LockManager.LockState private _lockState;
```

The issue is that when a user creates a lock through the lock() function, only the \_lockState variable is updated:

```solidity
_lockState.createLock(msg.sender, amount, duration);
```

The getLockedBalance() function incorrectly reads from the direct locks mapping:

```solidity
function getLockedBalance(address account) external view returns (uint256) {
    return locks[account].amount;
}
```

Since this mapping is never updated during lock creation, the function always returns 0 regardless of the actual locked amount.

## Impact

This inconsistency means that any user or protocol relying on the getLockedBalance() function will receive incorrect information (0) about locked token amounts. Since getLockPosition() provides correct information, this is unlikely to cause financial loss, but it could lead to confusion, inaccurate display of locked token amounts in UIs, or integration issues with protocols that rely on getLockedBalance(). As the function does not appear to be used in critical protocol logic, this is assessed as a low severity issue representing a correctness bug rather than a security vulnerability.
## Recommendations

Modify the getLockedBalance function to read from \_lockState:

```solidity
function getLockedBalance(address account) external view returns (uint256) {
    return _lockState.getLock(account).amount;
}
```
