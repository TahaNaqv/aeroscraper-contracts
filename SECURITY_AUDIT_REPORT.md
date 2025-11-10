# Aerospacer Protocol - Comprehensive Security Audit Report
**Date:** November 10, 2025  
**Auditor:** Replit Agent (Architect System)  
**Scope:** All 16 instructions in aerospacer-protocol program

---

## Executive Summary

A comprehensive security audit was conducted on all instructions in the aerospacer-protocol contract. Out of 16 instructions:

- ✅ **6 Production-Ready**: transfer_stablecoin, borrow_loan, repay_loan, open_trove, close_trove, stake, unstake, query_liquidatable_troves, withdraw_liquidation_gains, redeem
- ⚠️ **6 Need Fixes**: initialize, update_protocol_addresses, add_collateral, remove_collateral, liquidate_trove, liquidate_troves

---

## Critical Findings Summary

### 🔴 CRITICAL SEVERITY

1. **liquidate_trove**: Burns entire debt even when partially covered by stability pool, destroying unbacked tokens and breaking solvency
2. **liquidate_troves**: Missing token account validation allows collateral redirection attacks

### 🟡 MEDIUM SEVERITY

3. **initialize**: Missing `stable_coin_code_id` persistence and unchecked mint account
4. **update_protocol_addresses**: No validation of target addresses, can brick protocol
5. **add_collateral**: Neighbor hints not enforced, sorted list integrity compromised
6. **remove_collateral**: Missing owner validation and neighbor hint enforcement

---

## Detailed Findings by Instruction

### 1. initialize ⚠️ NOT PRODUCTION-READY

**Status:** FAIL - Critical state initialization issues

**Issues:**
1. **Missing State Persistence**: `stable_coin_code_id` from `InitializeParams` is never written to `StateAccount`
2. **Unchecked Mint Account**: `stable_coin_mint` is `UncheckedAccount` with no owner/type validation

**Impact:** State inconsistency, potential mint misconfiguration

**Required Fixes:**
```rust
// Add to StateAccount initialization
state.stable_coin_code_id = params.stable_coin_code_id;

// Change account type
pub stable_coin_mint: Account<'info, Mint>,
```

---

### 2. update_protocol_addresses ⚠️ NOT PRODUCTION-READY

**Status:** FAIL - Missing address validation

**Issues:**
1. **No PDA Verification**: Accepts arbitrary Pubkeys without validation
2. **No Program Ownership Checks**: Can set addresses to wrong programs
3. **No Default Key Protection**: Can set addresses to Pubkey::default()

**Impact:** Protocol bricking, fee theft, denial of service

**Required Fixes:**
```rust
// Add PDA derivation/verification for each address
// Add program ownership checks
// Reject default/duplicate addresses
require!(
    params.oracle_helper_addr != Pubkey::default(),
    AerospacerProtocolError::InvalidAddress
);
```

---

### 3. transfer_stablecoin ✅ PRODUCTION-READY

**Status:** PASS - All security requirements met

**Validated:**
- ✓ Token account validation (mint, owner)
- ✓ Authorization enforcement
- ✓ CPI security
- ✓ Amount handling

**Optional Enhancements:**
- Add zero-amount guard
- Emit structured events

---

### 4. open_trove ✅ PRODUCTION-READY

**Status:** PASS - All security requirements met

**Validated:**
- ✓ PDA verification (crate::ID ownership)
- ✓ L_snapshot initialization (prevents retroactive rewards)
- ✓ Neighbor hint validation
- ✓ Token account validation
- ✓ Collateral/debt initialization

**Optional Enhancements:**
- Consider enforcing neighbor hints in production

---

### 5. add_collateral ⚠️ NOT PRODUCTION-READY

**Status:** FAIL - Sorted list integrity compromised

**Issues:**
1. **Ignored Neighbor Hints**: `prev_node_id/next_node_id` not forwarded to TroveManager
2. **Bypassable Validation**: Can supply arbitrary PDAs to pass ICR checks
3. **Missing Owner Check**: Token account owner not validated

**Impact:** Sorted list corruption, incorrect liquidation ordering

**Required Fixes:**
```rust
// Forward neighbor hints to TroveManager
TroveManager::add_collateral(
    &mut trove_data,
    params.amount,
    Some(params.prev_node_id),
    Some(params.next_node_id),
)?;

// Add token account owner check
require!(
    ctx.accounts.user_collateral_account.owner == ctx.accounts.user.key(),
    AerospacerProtocolError::Unauthorized
);
```

---

### 6. remove_collateral ⚠️ NOT PRODUCTION-READY

**Status:** FAIL - Multiple validation gaps

**Issues:**
1. **Missing Owner Validation**: Token account owner not checked
2. **Bypassable Neighbor Hints**: Validation can be skipped
3. **Ineffective ICR Guard**: Relies on attacker-controlled neighbors

**Impact:** Undercollateralization, sorted list corruption, token theft

**Required Fixes:**
```rust
// Add owner check
require!(
    ctx.accounts.user_collateral_account.owner == ctx.accounts.user.key(),
    AerospacerProtocolError::Unauthorized
);

// Enforce neighbor hints
require!(
    !ctx.remaining_accounts.is_empty(),
    AerospacerProtocolError::InvalidList
);

// Add direct ICR check
require!(
    new_icr >= state.min_collateral_ratio,
    AerospacerProtocolError::InvalidCollateralRatio
);
```

---

### 7. borrow_loan ✅ PRODUCTION-READY

**Status:** PASS (Previously audited and fixed)

**Validated:**
- ✓ Debt accounting correct (gross amount)
- ✓ Neighbor hint validation
- ✓ PDA verification
- ✓ Fee handling

---

### 8. repay_loan ✅ PRODUCTION-READY

**Status:** PASS (Previously audited and fixed)

**Validated:**
- ✓ Debt repayment logic
- ✓ Neighbor hint validation
- ✓ State updates
- ✓ Token burning

---

### 9. close_trove ✅ PRODUCTION-READY

**Status:** PASS - All security requirements met

**Validated:**
- ✓ Full debt repayment enforced
- ✓ Redistribution rewards applied
- ✓ PDA constraints on all accounts
- ✓ State cleanup (liquidity threshold)
- ✓ Token account validation

**Optional Enhancements:**
- Add explicit mint constraint on stablecoin account

---

### 10. liquidate_trove 🔴 NOT PRODUCTION-READY - CRITICAL

**Status:** FAIL - Solvency-breaking bug

**Critical Issue:**
**Unconditional Debt Burning**: Burns entire `debt_amount` before branching into hybrid logic, destroying unbacked tokens when stability pool can't cover

**Code Location:**
```rust
// CURRENT (WRONG):
anchor_spl::token::burn(burn_ctx, debt_amount)?; // Burns ALL debt

// Then later:
if covered_by_pool < debt_amount {
    redistribute_debt_and_collateral(...)?; // Redistributes debt already burned!
}
```

**Impact:**
- Breaks protocol solvency
- Creates unbacked stablecoin supply
- System becomes insolvent after any partial/empty pool liquidation

**Required Fix:**
```rust
// Move burn inside stability pool coverage branch
if total_stake > 0 {
    let covered_debt = debt_amount.min(total_stake);
    anchor_spl::token::burn(burn_ctx, covered_debt)?; // Only burn covered amount
    
    if covered_debt < debt_amount {
        // Redistribute uncovered portion
        let uncovered_debt = debt_amount - covered_debt;
        redistribute_debt_and_collateral(uncovered_debt, uncovered_collateral)?;
    }
} else {
    // Pure redistribution - NO BURN
    redistribute_debt_and_collateral(debt_amount, collateral_amount)?;
}
```

---

### 11. liquidate_troves 🔴 NOT PRODUCTION-READY - CRITICAL

**Status:** FAIL - Collateral redirection vulnerability

**Critical Issues:**
1. **Token Account Validation Broken**: `validate_token_account` ignores `expected_user` parameter
2. **Cross-Denom Corruption**: No verification that trove accounts match `params.collateral_denom`

**Impact:**
- Malicious liquidator can redirect seized collateral
- Cross-denomination accounting corruption
- Vault balance manipulation

**Required Fixes:**
```rust
// Fix validate_token_account
fn validate_token_account(
    account: &AccountInfo,
    expected_user: &Pubkey,
    expected_mint: &Pubkey,
) -> Result<()> {
    let token_account = TokenAccount::try_deserialize(&mut &account.data.borrow()[..])?;
    require!(
        token_account.owner == *expected_user,
        AerospacerProtocolError::Unauthorized
    );
    require!(
        token_account.mint == *expected_mint,
        AerospacerProtocolError::InvalidMint
    );
    Ok(())
}

// Add denom validation for each trove
for trove in troves {
    // Verify PDA seeds match params.collateral_denom
    verify_collateral_denom_match(trove.collateral_account, &params.collateral_denom)?;
}
```

---

### 12. stake ✅ PRODUCTION-READY

**Status:** PASS (Previously audited and fixed)

**Validated:**
- ✓ Snapshot management
- ✓ State validation
- ✓ Token transfers

---

### 13. unstake ✅ PRODUCTION-READY

**Status:** PASS (Previously audited and fixed)

**Validated:**
- ✓ Snapshot updates
- ✓ Minimum amount handling
- ✓ Fund trapping prevention

---

### 14. query_liquidatable_troves ✅ PRODUCTION-READY

**Status:** PASS (Previously audited and fixed)

**Validated:**
- ✓ PDA verification
- ✓ Program ownership checks
- ✓ CPI compatibility

---

### 15. withdraw_liquidation_gains ✅ PRODUCTION-READY

**Status:** PASS (Recently fixed)

**Validated:**
- ✓ Token account validation
- ✓ PDA authenticity verification
- ✓ Snapshot protection
- ✓ Vault balance checks
- ✓ State persistence

---

### 16. redeem ✅ PRODUCTION-READY

**Status:** PASS (Recently fixed)

**Validated:**
- ✓ PDA verification
- ✓ Deterministic integer math
- ✓ Token account validation
- ✓ ICR ordering validation
- ✓ Redistribution rewards
- ✓ Zero-collateral protection

---

## Production Readiness Summary

### ✅ Ready for Production (10 instructions)
1. transfer_stablecoin
2. open_trove
3. borrow_loan
4. repay_loan
5. close_trove
6. stake
7. unstake
8. query_liquidatable_troves
9. withdraw_liquidation_gains
10. redeem

### 🔴 Requires Critical Fixes (2 instructions)
1. **liquidate_trove** - Solvency-breaking debt burn bug
2. **liquidate_troves** - Collateral redirection vulnerability

### ⚠️ Requires Important Fixes (4 instructions)
1. **initialize** - State initialization gaps
2. **update_protocol_addresses** - Missing validation
3. **add_collateral** - Sorted list integrity
4. **remove_collateral** - Validation gaps

---

## Recommendations

### Immediate Actions (Before Production)
1. **Fix liquidate_trove debt burning logic** (CRITICAL)
2. **Fix liquidate_troves token account validation** (CRITICAL)
3. Fix initialize state persistence
4. Add validation to update_protocol_addresses
5. Enforce neighbor hints in add_collateral and remove_collateral

### Testing Requirements
1. Add regression tests for all liquidation paths (full pool, partial, empty)
2. Test cross-denomination scenarios in liquidate_troves
3. Test sorted list integrity under adversarial conditions
4. Test state initialization completeness

### Future Enhancements
1. Consider enforcing neighbor hints globally for sorted list guarantees
2. Add structured events for better indexing
3. Add zero-amount guards where appropriate
4. Extend documentation on fee flows and authority models

---

## Audit Methodology

This audit used the Architect agent system to:
1. Review each instruction's account validation
2. Verify PDA authenticity and program ownership
3. Check token account validation (owner, mint)
4. Validate state update correctness
5. Verify CPI security (invoke_signed usage)
6. Check economic logic correctness
7. Identify edge cases and attack vectors

Each instruction was evaluated against production readiness criteria including security, correctness, and completeness.
