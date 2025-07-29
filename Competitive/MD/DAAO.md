# DAAO - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
    
- ## High Risk Findings
	- ### [[H-01] Missing ETH to WETH Conversion Preventing Pool Creation, Therefore Unable to Initialise any Daao](#H-01) 
	- ### [[H-02] Missing Tick Range and Spacing Validation in finalizeFundraising Function](#H-02)
	- ### [[H-03] Catastrophic Token/LP Ratio Imbalance Enables Immediate Value Extraction](#H-03)
- ## Medium Risk Findings
	- ### [[M-01] Mandatory Tier Assignment Undermines Public Contribution Rounds](#M-01) 

- ## Low Risk Findings
    - ### [[L-01] Tier Manipulation Through Duplicate Address Inputs](#L-01)
	- ### [[L-02] Missing Validation in Daao.sol::extendFundraisingDeadline()](#L-02)
	- ### [[L-03] Missing ETH Balance Validation in execute() Function Can Lead to Failed Transactions](#L-03)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: DAAO

### Chain: EVM

### Dates: 27 Jan, 2025 - 31 Jan, 2025

[See more contest details here](https://cantina.xyz/competitions/bd43bdd1-bc7f-473b-96c0-d35d37f3db33)
# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 3
- Medium: 1 
- Low: 3

# High Risk Findings

## <a id='H-01'></a>H-01 Missing ETH to WETH Conversion Preventing Pool Creation, Therefore Unable to Initialise any Daao

## Summary

The `finalizeFundraising` function attempts to create a liquidity pool with WETH/MODE without first converting the contract's ETH balance to WETH, causing the pool creation to fail and potentially locking user funds.

## Description:

In the `Daao.sol` contract, the `finalizeFundraising` function attempts to create a liquidity pool pairing the DAO token with WETH (referenced as `MODE` at address `0x4200000000000000000000000000000000000006`). The contract calculates amounts for liquidity provision but fails to convert its ETH balance to WETH before attempting to create the pool.

The critical flaw occurs in this sequence:
```js
uint256 totalCollected = address(this).balance;
uint256 amountForLP = (totalCollected * LP_PERCENTAGE) / 100;
uint256 amountForTreasury = totalCollected - amountForLP;

// Contract attempts to use WETH without converting ETH
if (daoToken < MODE) {
    token0 = daoToken;
    token1 = MODE;  // WETH address
    amountToken0ForLP = 4 * amountForLP;
    amountToken1ForLP = amountForLP;
} else {
    token0 = MODE;  // WETH address
    token1 = daoToken;
    amountToken0ForLP = amountForLP;
    amountToken1ForLP = 4 * amountForLP;
}

// These approvals will fail as contract has no WETH balance
IERC20(token0).approve(address(POSITION_MANAGER), amountToken0ForLP);
IERC20(token1).approve(address(POSITION_MANAGER), amountToken1ForLP);


```

When the POSITION_MANAGER attempts to transfer WETH from the contract to create the pool, the transaction will revert because the contract only holds ETH, not WETH. This prevents the fundraising from being finalized. In addition to this, `refund()` can not be called as `goalReached` has been set to true.
## Impact:

This issue has High impact because:

- It prevents the core fundraising finalization process from completing
- It leads to user funds being locked in the contract as `goalReached` == true, thus `refund()` cannot be called; `emergencyEscape()` must be called.
- It affects all users who participate in the fundraising

## Likelihood:

The likelihood is High because:

- This issue will occur in every deployment where fundraising reaches its goal
- There is no alternative path or fallback mechanism
- The issue is not dependent on specific conditions or user behaviour

## Recommended Mitigation:

The contract should properly convert ETH to WETH before attempting pool creation. Additionally, it should account for gas costs in the calculations:
```js
function finalizeFundraising(int24 initialTick, int24 upperTick) external {
    // Previous checks remain the same
    
    uint256 totalCollected = address(this).balance;
    
    // Reserve some ETH for gas costs
    uint256 gasReserve = 0.1 ether; // Adjust based on expected gas costs
    require(totalCollected > gasReserve, "Insufficient balance for gas");
    totalCollected -= gasReserve;
    
    uint256 amountForLP = (totalCollected * LP_PERCENTAGE) / 100;
    uint256 amountForTreasury = totalCollected - amountForLP;
    
    // Transfer treasury amount
    (bool success, ) = owner().call{value: amountForTreasury}("");
    require(success, "Treasury transfer failed");
    
    // Convert ETH to WETH
    IWETH(MODE).deposit{value: amountForLP}();
    
    // Now the contract has WETH, proceed with pool creation

    
    // Rest of the function remains the same
}
```

## <a id='H-02'></a>H-02 Missing Tick Range and Spacing Validation in finalizeFundraising Function

## Summary

The `finalizeFundraising` function in `Daao.sol` accepts tick parameters for concentrated liquidity positions without validating their ordering or spacing requirements, which could lead to failed transactions or creation of unusable liquidity positions.

## Description:

The Daao protocol allows creation of concentrated liquidity positions through the `finalizeFundraising` function, which accepts `initialTick` and `upperTick` parameters. These parameters define the price range for the liquidity position. However, the contract fails to validate two critical requirements:
```js
int24 public constant TICKING_SPACE = 100;

function finalizeFundraising(int24 initialTick, int24 upperTick) external {
    // ... 
    INonfungiblePositionManager.MintParams memory params = INonfungiblePositionManager.MintParams(
        token0,
        token1,
        TICKING_SPACE,  // Spacing is specified here
        initialTick,    // But not validated
        upperTick,      // But not validated
        amountToken0ForLP,
        amountToken1ForLP,
        0,
        0,
        address(this),
        block.timestamp,
        sqrtPriceX96
    );

```
This lack of validation means the contract will attempt to create positions with invalid parameters, which will either:

Cause the transaction to revert at the Velodrome-Slipstream protocol level, wasting gas Create positions with invalid or unusable price ranges if the underlying protocol doesn't properly validate these parameters

1. The ticks must be multiples of the defined `TICKING_SPACE` (100)
2. `initialTick < upperTick`
## Impact:

The impact of this vulnerability is High because:

It affects a core protocol function (liquidity provision) which is essential for every DAO created through this system Failed transactions waste gas and could make the fundraising finalization impossible If positions with invalid parameters are created, the liquidity could become trapped in unusable ranges The issue affects all DAOs created through the protocol, not just individual instances

## Likelihood:

The likelihood is Low because:

1. The parameters are provided by `owner` without any validation.
2. Given this contract works like a factory contract, a new Owner may naturally input non-multiple of 100 values or incorrect ordering when trying to define price range

## Recommended Mitigation:

Implement proper validation checks before creating the liquidity position:
```js
function validateTicks(int24 initialTick, int24 upperTick) internal pure {
    // Validate tick spacing
    require(initialTick % TICKING_SPACE == 0, "Initial tick must be multiple of spacing");
    require(upperTick % TICKING_SPACE == 0, "Upper tick must be multiple of spacing");
    
    // Validate tick ordering
    require(initialTick < upperTick, "Upper tick must be greater than initial tick");
}

function finalizeFundraising(int24 initialTick, int24 upperTick) external {
    validateTicks(initialTick, upperTick);
    // ... rest of the function
}


```

## <a id='H-03'></a>H-03 Catastrophic Token/LP Ratio Imbalance Enables Immediate Value Extraction

## Summary:

The DAAO protocol has a critical vulnerability in its tokenomics design where the ratio between distributed tokens and LP pool tokens creates an extreme economic imbalance. This enables contributors to extract significantly more value than their initial investment by draining the liquidity pool immediately after launch.

## Description:

The vulnerability exists in Daao.sol where the LP token allocation is hardcoded to a 4:1 ratio with ETH:
```js
if (daoToken < MODE) {
    // 4:1 mapping (Token Ratio in pool)
    amountToken0ForLP = 4 * amountForLP;    // Only 4 tokens for LP /ETH
    amountToken1ForLP = amountForLP;        // ETH amount
}
```
Through testing, I've discovered a severe economic vulnerability in how tokens are distributed versus how they're valued in the liquidity pool. Here's the key issue:

Example Scenario: For a 100 ETH raise, 100 contributors, 1Eth limit:

- Total token supply: 1,000,000,000 tokens (This is hardcoded)
- Contributors each get proportional share (e.g. 100 contributors, each get 10_000_000 tokens)
- LP pool gets only 40 tokens paired with 10 ETH (10% of ETH raise reserved for LP, hardcoded 4 daoTokens : 1 ETH)

This creates a massive price discrepancy:

- LP pool implies a token price of 0.25 ETH per token (10 ETH / 40 tokens)
- At this price, a single user's 10M tokens are theoretically worth 2.5M ETH
- The LP pool only contains 10 ETH to defend this price
- Results in a 2.5 million times potential profit multiple

The hardcoded ratio is completely disconnected from the total token supply (1B tokens) distributed to contributors.
## Severity:

The severity is actually critical because:

- The vulnerability is immediately exploitable upon launch
- Any contributor can drain the LP pool for massive profit
- The protocol is fundamentally broken economically
- User funds in the LP are guaranteed to be lost
- No special conditions or complex setup required to exploit
## Recommended Mitigation:

1. Revise the LP token allocation to be proportional to total supply
2. Ensure LP token amount reflects a sustainable token price
3. Add validation to prevent extreme token/ETH ratios

---

# Medium Risk Findings

## <a id='M-01'></a>M-01 Mandatory Tier Assignment Undermines Public Contribution Rounds

## Summary

In public contribution rounds (maxWhitelistAmount = 0), the contract's mandatory tier assignment system unintentionally restricts user contributions to their tier limits, regardless of the higher maxPublicContributionAmount. This effectively makes the public contribution limit unreachable.

## Description:

`Daao.sol` enforces mandatory tier assignment for all contributors through the `addToWhitelist()`:
```js
function addToWhitelist(address[] calldata _addresses, WhitelistTier[] calldata _tiers) external {
    ...
@>  require(_tiers[i] != WhitelistTier.None, "Invalid tier");
    userTiers[_addresses[i]] = _tiers[i];
}
```
Even in "public" rounds (maxWhitelistAmount = 0), every contributor must be assigned a tier. The contribute function then enforces tier limits regardless of public contribution settings:
```js
function contribute() public payable nonReentrant {
    WhitelistTier userTier = userTiers[msg.sender];
    require(userTier != WhitelistTier.None, "Not whitelisted");
    
    uint256 userLimit = tierLimits[userTier];
    require(contributions[msg.sender] + msg.value <= userLimit, "Exceeding tier limit");

    if (maxWhitelistAmount > 0) {
        require(contributions[msg.sender] + msg.value <= maxWhitelistAmount, "Exceeding maxWhitelistAmount");
    } else if (maxPublicContributionAmount > 0) {
        require(contributions[msg.sender] + msg.value <= maxPublicContributionAmount, "Exceeding maxPublicContributionAmount");
    }
    ...
}
```
This creates a situation where even if maxPublicContributionAmount is set high (e.g. 1 ETH), and contributors were assigned lower tiers just to add them to the whitelist(e.g. Silver with 0.1 ETH limit), they cannot contribute more than their tier limit, effectively making the public contribution limit unreachable.
## Impact:

Impact is rated as medium because:

- Causes severe disruption to intended protocol functionality
- No funds are at risk
- The contract remains secure, just more restrictive than intended

## Likelihood:

Likelihood is rated as medium because:

- The condition occurs 100% of the time in public rounds
- The issue is triggered by basic contract functionality
## Recommended Mitigation:

There are a couple of different approaches to fix this:

1. Bypass tier limits in public rounds
2. Add a public tier with configurable limit:

```js
enum WhitelistTier {
    None,
    Public,    // New tier for public rounds
    Platinum,
    Gold,
    Silver
}

```

---

# Low Risk Findings

## <a id='L-01'></a>L-01 Tier Manipulation Through Duplicate Address Inputs

## Summary

The `addToWhitelist()` function in `Daao.sol` lacks validation for duplicate addresses in input arrays, which could lead to unintended tier assignments when bulk-adding users to the whitelist.

## Description:

The `addToWhitelist()` function accepts arrays of addresses and their corresponding whitelist tiers but does not validate whether an address appears multiple times in the input. During large-scale whitelist operations or batch updates, this oversight could result in accidental tier reassignments since the function processes the arrays sequentially and overwrites the userTiers mapping with each iteration.

For example:
```js
addToWhitelist(
    [userA, userB, userA],
    [WhitelistTier.Silver, WhitelistTier.Gold, WhitelistTier.Platinum]
)
```
In this scenario, if userA was meant to be assigned Silver tier but accidentally appears again in the array with Platinum tier (perhaps due to a copy-paste error or data processing mistake), they would end up with Platinum tier.
## Impact:

The impact is Medium because:

- It could lead to users receiving incorrect tier assignments and contribution limits
- Does not directly result in loss of funds but affects the integrity of the tiered access system

## Likelihood:

The likelihood is Medium because:

- Batch operations with large arrays are common in whitelist management
- Data preparation errors or CSV processing mistakes could easily introduce duplicates
- The issue exists in core whitelist management functionality

## Recommended Mitigation:

Add validation to prevent duplicate addresses in the input arrays.

## <a id='L-02'></a>L-02 Missing Validation in Daao.sol::extendFundraisingDeadline

## Summary

The `Daao.sol::extendFundraisingDeadline()` function lacks validation to ensure the new deadline remains less than `fundExpiry`, potentially violating a core invariant established in the constructor.

## Description:

In `Daao.sol`, the constructor explicitly requires that _fundExpiry > fundraisingDeadline:
```js
constructor(
    uint256 _fundraisingGoal,
    string memory _name,
    string memory _symbol,
    uint256 _fundraisingDeadline,
    uint256 _fundExpiry,
    address _daoManager,
    address _liquidityLockerFactory,
    uint256 _maxWhitelistAmount,
    address _protocolAdmin,
    uint256 _maxPublicContributionAmount
) {
    require(
        _fundExpiry > fundraisingDeadline,
        "_fundExpiry > fundraisingDeadline"
    );
    // ...
}
```
However, when extending the fundraising deadline via `extendFundraisingDeadline`, there is no check to maintain this invariant:
```js
function extendFundraisingDeadline(
    uint256 newFundraisingDeadline
) external {
    require(
        msg.sender == owner() || msg.sender == protocolAdmin,
        "Must be owner or protocolAdmin"
    );
    require(!goalReached, "Fundraising goal was reached");
    require(
        newFundraisingDeadline > fundraisingDeadline,
        "new fundraising deadline must be > old one"
    );
    fundraisingDeadline = newFundraisingDeadline;
}
```

This means an admin could extend the fundraising deadline beyond the fund expiry date, breaking the intended temporal sequence of the protocol where fundraising must complete before fund expiry.
## Impact:

The impact is rated as medium because:

- It violates the core assumption that fundraising should complete before fund expiry
- Could interfere with liquidity pool operations which rely on the fund expiry timing
- May cause unexpected behavior in time-dependent operations

## Likelihood:

The likelihood is medium because:

- No UI safeguards can fully prevent this as it's a contract-level issue
- Current implementation makes it easy to overlook this requirement

## Recommended Mitigation:

Add a validation check in the extendFundraisingDeadline function to maintain the required temporal ordering:
```js
function extendFundraisingDeadline(
    uint256 newFundraisingDeadline
) external {
    require(
        msg.sender == owner() || msg.sender == protocolAdmin,
        "Must be owner or protocolAdmin"
    );
    require(!goalReached, "Fundraising goal was reached");
    require(
        newFundraisingDeadline > fundraisingDeadline,
        "new fundraising deadline must be > old one"
    );
  @>require(
        newFundraisingDeadline < fundExpiry,
        "new fundraising deadline must be < fund expiry"
    );
    fundraisingDeadline = newFundraisingDeadline;
}
```

## <a id='L-03'></a>L-03 Missing ETH Balance Validation in execute() Function Can Lead to Failed Transactions

## Summary

The `execute()` function in `Daao.sol` lacks balance validation before attempting ETH transfers, which could lead to failed transactions and wasted gas if insufficient ETH is present in the contract.

## Description:

The execute() function allows the owner (AI treasury) to make arbitrary contract calls with ETH values, but does not verify if the contract has sufficient ETH balance before attempting these transfers:
```js
function execute(
    address[] calldata contracts,
    bytes[] calldata data,
    uint256[] calldata msgValues
) external onlyOwner {
    require(fundraisingFinalized);
    require(
        contracts.length == data.length && data.length == msgValues.length,
        "Array lengths mismatch"
    );

    for (uint256 i = 0; i < contracts.length; i++) {
        (bool success, ) = contracts[i].call{value: msgValues[i]}(data[i]);
        require(success, "Call failed");
    }
}
```

Since the contract transfers 90% of ETH to the AI treasury during `finalizeFundraising()`:

```js
// Transfer treasury amount to owner (We are left with LP amount)
	(bool success, ) = owner().call{value: amountForTreasury}("");
	require(success, "Treasury transfer failed");
```

and the remaining 10% goes to liquidity provision, the contract may have insufficient ETH to fulfill the 'msgValues` in execute calls.
## Impact:

Medium:

- Cause transaction failures and wasted gas
- Lead to failed treasury operations
- Create operational friction requiring additional transfers

## Likelihood

Medium - Will occur any time execute() is called with ETH values exceeding the contract's balance, which is likely given the treasury transfer mechanism.

## Recommended Mitigation:

Add balance validation before attempting transfers:
```js
function execute(
    address[] calldata contracts,
    bytes[] calldata data,
    uint256[] calldata msgValues
) external onlyOwner {
    require(fundraisingFinalized);
    require(
        contracts.length == data.length && data.length == msgValues.length,
        "Array lengths mismatch"
    );
    
@>  // Add balance validation
    uint256 totalETHRequired;
    for(uint256 i = 0; i < msgValues.length; i++) {
        totalETHRequired += msgValues[i];
    }
    require(
        address(this).balance >= totalETHRequired,
        "Insufficient ETH balance for execution"
    );
    
    for (uint256 i = 0; i < contracts.length; i++) {
        (bool success, ) = contracts[i].call{value: msgValues[i]}(data[i]);
        require(success, "Call failed");
    }
}
```

---
