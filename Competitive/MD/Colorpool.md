# Colorpool- Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
    
- ## High Risk Findings
	 - ### [[H-01] Credit System Bypass via Account Cycling](#H-01) 

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Colorpool

### Chain: Chromia

### Dates: March 3rd, 2025 - March 17th, 2025

[See more contest details here](https://cantina.xyz/competitions/7db75599-9dad-40aa-9fc7-e879803eea2b)
# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 1

# High Risk Findings

## <a id='H-01'></a>H-01 Credit System Bypass via Account Cycling

## Summary

The DEX platform's credit system can be bypassed by users creating new accounts and transferring assets between them, effectively gaining unlimited operations despite daily credit limits.
## Description:

The platform implements a credit system where each account receives 1000 daily free credits, with operations costing varying amounts (e.g., -5 for swaps, -200 for account registration). This system is designed to prevent spam and abuse by limiting the number of operations users can perform daily and also providing a source of revenue since it is the developers/node operators who are paying for transactions on the Chromia blockchain. However, the account creation implementation presents insufficient barriers to prevent determined users from bypassing these limits. According to the platform's configuration, accounts can be created through two pathways with minimal economic barriers:

```toml
lib.ft4.core.accounts.strategies.transfer:
  rules:
    - sender_blockchain: 
      - x"090bcd47149fbb66f02489372e88a454e7a5645adde82125d40df1ef0c76f874" # Economy chain
      - x"7E57DB1DF38BE1A8503B9531A08B982B1D5F30F5BFE3AD9ECEBDC9C553717AAC" # Colorpool bridge chain
      sender: "*"
      recipient: "*"
      asset:
        - issuing_blockchain_rid: x"090bcd47149fbb66f02489372e88a454e7a5645adde82125d40df1ef0c76f874"
          name: "Chromia Test"
          min_amount: 10000000L # 10 CHR
        - issuing_blockchain_rid: x"7E57DB1DF38BE1A8503B9531A08B982B1D5F30F5BFE3AD9ECEBDC9C553717AAC"
          name: "BUSDC"
          min_amount: 2000000000000000000L # 2 BUSDC
      timeout_days: 15
      strategy: "open" # free if transfer from economy chain or colorpool bridge chain
    - sender_blockchain: "*"
      sender: "*"
      recipient: "*"
      asset:
        - name: "BUSDC"
          min_amount: 2000000000000000000L
        - name: "Chromia Test"
          min_amount: 10000000L
        - name: "COLOR"
          min_amount: 1000000000000000000000L # 10_000 COLOR
      timeout_days: 15
      strategy: "fee"lib.ft4.core.accounts.strategies.transfer:
  rules:
    - sender_blockchain: 
      - x"090bcd47149fbb66f02489372e88a454e7a5645adde82125d40df1ef0c76f874" # Economy chain
      - x"7E57DB1DF38BE1A8503B9531A08B982B1D5F30F5BFE3AD9ECEBDC9C553717AAC" # Colorpool bridge chain
      sender: "*"
      recipient: "*"
      asset:
        - issuing_blockchain_rid: x"090bcd47149fbb66f02489372e88a454e7a5645adde82125d40df1ef0c76f874"
          name: "Chromia Test"
          min_amount: 10000000L # 10 CHR
        - issuing_blockchain_rid: x"7E57DB1DF38BE1A8503B9531A08B982B1D5F30F5BFE3AD9ECEBDC9C553717AAC"
          name: "BUSDC"
          min_amount: 2000000000000000000L # 2 BUSDC
      timeout_days: 15
      strategy: "open" # free if transfer from economy chain or colorpool bridge chain
    - sender_blockchain: "*"
      sender: "*"
      recipient: "*"
      asset:
        - name: "BUSDC"
          min_amount: 2000000000000000000L
        - name: "Chromia Test"
          min_amount: 10000000L
        - name: "COLOR"
          min_amount: 1000000000000000000000L # 10_000 COLOR
      timeout_days: 15
      strategy: "fee"

```

The configuration reveals two critical issues:

1. Users from the Economy or Colorpool bridge chains can create accounts using the "open" strategy, requiring only a transfer of 10 CHR or 2 BUSDC without any actual fee payment.
2. Users from any other blockchain can create accounts using the "fee" strategy with minimal costs:
3. 
```toml
lib.ft4.core.accounts.strategies.transfer.fee:
  asset:
    - name: "BUSDC"
      amount: 2000000000000000000L  # 2 BUSDC
    - name: "Chromia Test"
      amount: 10000000L  # 10 CHR
    - name: "COLOR"
      amount: 1000000000000000000000L # 10_000 COLOR
```

Importantly, the system lacks admin-only controls on account creation. The configuration shows a regular transfer-based registration system that's open to any user meeting the minimal requirements:

```toml
lib.ft4.core.accounts:
  auth_flags:
    mandatory: "T"
  auth_descriptor:
    max_number_per_account: 100
    max_rules: 4
```

The attack flow works as follows:

1. Attacker creates an account by transferring 2 BUSDC (using the fee strategy) or by transferring 10 CHR/2 BUSDC from the Economy/Colorpool chain (using the open strategy)
2. The account receives 1000 daily credits (minus 200 for account creation)
3. Attacker uses the 800 credits for operations until depleted
4. Attacker transfers their assets to a new account
5. Attacker creates another account through the same process
6. The cycle repeats, effectively bypassing the daily credit limit

The credit system implementation in `credits/function.rell` is account-based, with no mechanism to track or limit users who create multiple accounts:

```
function _resolver_credit(account_id: byte_array, op_name: text): byte_array? {
->  // ... credit deduction happens per account with no cross-account tracking ...
    
->  // Get daily free amount
    _daily_free_credit(account_id, get_time_begin_day(op_context.last_block_time));
    
    return _create_or_update_credit_override_balance(
        get_account_by_pubkey(account_id),
        0L,
        fee,
        type,
        get_account_by_pubkey(account_id),
        false
    );
}
```

This combination of easy account creation and account-based credit limits creates a significant vulnerability, allowing users to treat the intended "daily" limit as merely a "per account" limit.
## Impact:

Medium:

1. Unfair advantages for users willing to cycle accounts versus those respecting daily limits
2. unknowing users would be paying for credits whilst other bypass
3. At the cost of the developers/node operators as they are essentially paying for transactions on Chromia.

## Likelihood:

High:

1. The implementation barrier is low - users only need to automate account creation and asset transfers
2. The economic barrier is minimal - 2 BUSDC is insignificant for traders moving substantial volumes
3. The potential gain from bypassing rate limits can be substantial for high-frequency traders and arbitrageurs
4. The pattern is discoverable through normal usage as users exhaust their daily credits

For sophisticated users or traders, this pattern would be both obvious and economically rational to exploit, especially during periods of high market volatility where additional operations could yield significant profits.
## Recommended Mitigation:

Several approaches could address this vulnerability:

1. There should be an associated credit cost with transferring tokens once a user has used up daily free credit
2. Increase the account creation cost significantly to make cycling economically unviable
3. Introduce cross-account tracking using onchain analysis to detect and penalize cycling patterns
4. Have admin control account creation
5. Implement a time-delay on asset transfers for new accounts to make cycling less time-efficient

---
