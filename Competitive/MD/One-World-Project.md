# One World Project- Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [[H-01] No contract-level verification of KYC requirements for `MembershipFactory.sol::createNewDAOMembership()` and `MembershipFactory.sol::joinDAO()` despite platform requirements; Any address can create or join a DAO.](#H-01)
# <a id='contest-summary'></a>Contest Summary

### Sponsor: One World

### Chain: EVM

### Dates: Nov 6th, 2024 - Nov 13th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-11-one-world)
# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 0
- Low: 0
# High Risk Findings

## <a id='H-01'></a>H-1 No contract-level verification of KYC requirements for `MembershipFactory.sol::createNewDAOMembership()` and `MembershipFactory.sol::joinDAO()` despite platform requirements; Any address can create or join a DAO.            

## Description:

The MembershipFactory contract lacks validation that:

1. DAO creators have completed required KYC/AML process
2. DAO joiners own a pseudo-KYC Identity NFT

From One World Project's [website](https://www.oneworldproject.io/dao):

> DAO Creation: Sign Up and Verify - Create an account and complete the KYC/AML process.\
> DAO Membership: Purchase an Identity NFT, which serves as your pseudo-KYC and digital business card. This NFT verifies your identity and grants you access to the platform's features.

However, examining MembershipFactory.sol shows no validation at the contract level for either requirement:

```solidity
// No KYC check for DAO creation:
function createNewDAOMembership(DAOInputConfig calldata daoConfig, TierConfig[] calldata tierConfigs) external returns (address) { require(currencyManager.isCurrencyWhitelisted(daoConfig.currency), "Currency not accepted."); require(daoConfig.noOfTiers == tierConfigs.length, "Invalid tier input."); 
require(daoConfig.noOfTiers > 0 && daoConfig.noOfTiers <= TIER_MAX, "Invalid tier count."); 
require(getENSAddress[daoConfig.ensname] == address(0), "DAO already exist."); 
// No check that msg.sender has completed KYC/AML ...

// No Identity NFT check for joining DAOs:
function joinDAO(address daoMembershipAddress, uint256 tierIndex) external { 
require(daos[daoMembershipAddress].noOfTiers > tierIndex, "Invalid tier.");
require(daos[daoMembershipAddress].tiers[tierIndex].amount > daos[daoMembershipAddress].tiers[tierIndex].minted, "Tier full.");
// No check that msg.sender owns an Identity NFT ...
```

## Impact:

* Complete bypass of platform's identity verification and compliance requirements
* Any address can create DAOs without KYC/AML verification
* Any address can join DAOs without Identity NFT verification
* Core platform functionality that relies on verified identities is compromised
* Reliance solely on frontend restrictions which can be circumvented by anyone who knows the contract address post-deployment.

## Explanation of severity:

This is considered a high severity bug as likelihood is High(very easy to circumvent) and the impact is also High(having members be KYC'd is core principle of protocol).

Security through obscurity is not a valid approach.

> If it was the intention of the protocol to  control entire business flow on the frontend, the functions should have access controls, thus the issue remains.

## Recommended Mitigation:

1. Add access controls if the intention was frontend control from the protocol.
2. Implement on-chain KYC tracking for DAO creation.
3. Add Identity NFT check to DAO joining:

```diff
function joinDAO(address daoMembershipAddress, uint256 tierIndex) external { 
+ require(identityNFT.balanceOf(msg.sender) > 0, "Must own Identity NFT to join DAO"); 
  require(daos[daoMembershipAddress].noOfTiers > tierIndex, "Invalid tier."); ... }
```