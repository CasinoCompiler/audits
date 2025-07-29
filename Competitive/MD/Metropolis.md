# Metropolis - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)

- ## Medium Risk Findings
	 - ### [[M-01] Emergency Mode Can Be Blocked by Queued Withdrawals, Creating DOS Vector](#M-01) 
# <a id='contest-summary'></a>Contest Summary

### Sponsor: Metropolis

### Chain: EVM

### Dates: April 2nd, 2025 - April 20th, 2025

[See more contest details here](https://cantina.xyz/competitions/076935b1-2706-48c6-bf0a-b3656aa24194)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- Medium: 1

# Medium Risk Findings

## <a id='M-01'></a>M-01Emergency Mode Can Be Blocked by Queued Withdrawals, Creating DOS Vector

## Summary
The `setEmergencyMode()` activation can be blocked by pending queued withdrawals, creating a denial-of-service vulnerability where an attacker can continuously prevent emergency responses by strategically queuing minimal withdrawals.

## Description:
When `setEmergencyMode()` is called on `BaseVault.sol` via `VaultFactory.sol`, it executes the following sequence:

1. The vault calls `_strategy.withdrawAll()`
2. The strategy transfers assets back to the vault
3. The strategy then attempts to call `vault.executeQueuedWithdrawals()`
4. This call fails with `ReentrancyGuard: reentrant call` because the vault is already in a non-reentrant function (`setEmergencyMode()`)

Due to this design, emergency mode can only be activated if there are no pending queued withdrawals in the current round, or after those withdrawals have been processed via a strategy rebalance operation.

An attacker can exploit this limitation by continuously monitoring the mempool for emergency mode activation attempts and frontrunning them with minimal withdrawal queue transactions. Since there is no safeguards, the attacker can effectively create a permanent denial-of-service condition against the emergency mode functionality. The attack would work as follows:

1. Attacker deposits a minimal amount to obtain vault shares
2. Attacker monitors the mempool for `setEmergencyMode()` transactions
3. When detected, attacker frontruns with a small `queueWithdrawal()` transaction using higher gas fees
4. The emergency mode transaction will then fail due to the reentrancy protection
5. If operators try to process withdrawals first, the attacker can frontrun those operations as well
## Impact:
As per [Cantina Impact guidelines,](https://docs.cantina.xyz/cantina-docs/cantina-competitions/judging-process/finding-severity-criteria#:~:text=Breaks%20Core%20Functionality%3A%20Causes%20a%20failure%20in%20fundamental%20protocol%20operations.) this is a high impact scenario as important functionality can be completely denied.

## Likelihood:
 Although it can be argued that this is a Low likelihood scenario as it requires the protocol/vault owner wanting to shutdown the vault due to an emergency, I'd like to challenge this perspective and deem it as High.

The reason for this is the context in why this would be exploited.

1. Firstly, if a vault wanted to shutdown operations normally without an emergency situation, it would call `submitShutdown()`. This just flags the vault for shutdown and the operator would have to call `setEmergencyMode()` to then actually shutdown.
2. If an emergency situation were to occur for any reason e.g. draining, unexpected behaviour etc., it is imperative that the vault is able to shutdown immediately to prevent any further loss.

Either situation should not be considered unlikely, as per this [notice,](https://docs.google.com/document/d/14K8ySz_A0qTiELvPsCyMmLt9dvN5ZkH_u32CJ0bTrTU/edit?tab=t.0) the protocol has recently conducted these activities.

Now that this context is in frame, it can be seen why I am deeming this as High:

- Regular shutdowns could just be griefed
- Emergency shutdowns can be DoS'd and this can be done [trivially](https://docs.cantina.xyz/cantina-docs/cantina-competitions/judging-process/finding-severity-criteria#:~:text=Likelihood-,High,Issues%20that%20will%20generate%20outsized%20returns%20to%20the%20exploiter,-Medium) as an attacker can just monitor the mempool and frontrun with minimal deposits/withdrawals indefinetely.
- If there is additional incentive for the attacker e.g. they have exploited a vault, they can actively DOS emergency shutdowns so that they can fully drain the vault.

NOTE: it might be argued that `_depositsPaused` can be flagged so an attacker won't be able to deposit, therefore cannot queue withdrawal; but, there is no function which flags both `_depositsPaused` and `_flaggedForShutdown` together. Additionally, an attacker could just pre-deposit small amount of funds in every single vault in anticipation of vault shutdowns.

As such, with the above context in mind, the Severity should be deemed **critical** and following the [Cantina severity matrix,](https://docs.cantina.xyz/cantina-docs/cantina-competitions/judging-process/finding-severity-criteria) this should be High x High

## Recommended Mitigation:
The emergency mode needs reworking and thorough testing. It is easy to just suggest removing one of the reentrency guards, but this could introduce a reentrency attack.

Some solutions could be:

1. Create a separate emergency mode function that bypasses queued withdrawal processing
2. Implement a two-phase emergency response:

- Phase 1: Freeze operations using the forced emergency mode
- Phase 2: Process asset recovery and queued withdrawals as separate transactions

3. Consider adding a time-based rate limit on queued withdrawals to prevent rapid successive calls from the same address
4. In the case of normal shutdowns, `_flaggedForShutdown` and `_depositsPaused` should be set `true` in one function.

---
