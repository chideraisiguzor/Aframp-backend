# Payment Monitoring Enhancement - Amount Verification

## Overview

This document provides the code enhancement to add amount verification to the payment monitoring system. This ensures that the incoming cNGN payment amount matches the expected transaction amount exactly.

---

## Enhancement Summary

**Current Behavior**: 
- Payments are matched to transactions by memo
- Payment asset type (cNGN) is verified
- **Missing**: Amount verification

**New Behavior**:
- All current checks pass
- Amount from Stellar operation is compared to `transaction.cngn_amount`
- If amounts don't match:
  - Payment is NOT processed
  - Amount mismatch event is logged
  - Transaction remains in `pending_payment` status
  - Alert is generated for manual review

---

## Code Changes Required

### 1. Modify `is_incoming_cngn_payment()` Method

**File**: `src/workers/transaction_monitor.rs`

**Current Signature** (around line 520):
```rust
async fn is_incoming_cngn_payment(
    &self,
    tx_hash: &str,
    system_wallet: &str,
) -> anyhow::Result<bool>
```

**New Signature**:
```rust
async fn is_incoming_cngn_payment(
    &self,
    tx_hash: &str,
    system_wallet: &str,
    expected_amount: Option<&str>,  // NEW: Expected cNGN amount
) -> anyhow::Result<PaymentVerificationResult>  // NEW: Return detailed result
```

### 2. Create Return Type

**New Struct** (add near the top of the file, after imports):

```rust
/// Result of payment verification
#[derive(Debug, Clone)]
pub struct PaymentVerificationResult {
    /// Payment is cNGN to system wallet
    pub is_cngn_payment: bool,
    /// Payment amount (if found)
    pub amount: Option<String>,
    /// Amount matches expected (if amount was provided)
    pub amount_matches: bool,
    /// Verification error message (if any)
    pub error: Option<String>,
}

impl PaymentVerificationResult {
    pub fn success() -> Self {
        Self {
            is_cngn_payment: true,
            amount: None,
            amount_matches: true,
            error: None,
        }
    }

    pub fn with_amount(amount: String) -> Self {
        Self {
            is_cngn_payment: true,
            amount: Some(amount),
            amount_matches: true,
            error: None,
        }
    }

    pub fn amount_mismatch(expected: &str, received: &str) -> Self {
        Self {
            is_cngn_payment: true,
            amount: Some(received.to_string()),
            amount_matches: false,
            error: Some(format!(
                "Amount mismatch: expected {}, received {}",
                expected, received
            )),
        }
    }

    pub fn not_cngn() -> Self {
        Self {
            is_cngn_payment: false,
            amount: None,
            amount_matches: false,
            error: Some("Not a cNGN payment".to_string()),
        }
    }
}
```

### 3. Enhanced Implementation

**New Implementation** (replace the existing `is_incoming_cngn_payment()` method):

```rust
async fn is_incoming_cngn_payment(
    &self,
    tx_hash: &str,
    system_wallet: &str,
    expected_amount: Option<&str>,
) -> anyhow::Result<PaymentVerificationResult> {
    let issuer = std::env::var("CNGN_ISSUER_TESTNET")
        .or_else(|_| std::env::var("CNGN_ISSUER_MAINNET"))
        .unwrap_or_default();
    
    let operations = self
        .stellar_client
        .get_transaction_operations(tx_hash)
        .await?;
    
    for op in operations {
        let op_type = op.get("type").and_then(|v| v.as_str()).unwrap_or("");
        if op_type != "payment" {
            continue;
        }

        let destination = op.get("to").and_then(|v| v.as_str()).unwrap_or("");
        let asset_code = op.get("asset_code").and_then(|v| v.as_str()).unwrap_or("");
        let asset_issuer = op
            .get("asset_issuer")
            .and_then(|v| v.as_str())
            .unwrap_or("");
        
        if destination == system_wallet && asset_code.eq_ignore_ascii_case("cngn") {
            if issuer.is_empty() || asset_issuer == issuer {
                // Found cNGN payment to system wallet
                let amount = op
                    .get("amount")
                    .and_then(|v| v.as_str())
                    .unwrap_or("0");

                // NEW: Check amount if expected_amount provided
                if let Some(expected) = expected_amount {
                    if amount != expected {
                        warn!(
                            tx_hash = %tx_hash,
                            expected_amount = %expected,
                            received_amount = %amount,
                            "cNGN payment amount mismatch"
                        );
                        return Ok(PaymentVerificationResult::amount_mismatch(expected, amount));
                    }
                }

                return Ok(PaymentVerificationResult::with_amount(amount.to_string()));
            }
        }
    }
    
    Ok(PaymentVerificationResult::not_cngn())
}
```

### 4. Update Call Site

**File**: `src/workers/transaction_monitor.rs` - `scan_incoming_transactions()` method

**Current Code** (around line 429):
```rust
let looks_like_incoming = self
    .is_incoming_cngn_payment(&tx.hash, system_wallet)
    .await
    .unwrap_or(false);
if !looks_like_incoming {
    continue;
}
```

**New Code**:
```rust
// First, extract memo to get transaction ID for lookup
let memo = match tx.memo.as_deref() {
    Some(m) if !m.trim().is_empty() => m,
    _ => continue,
};

let tx_id_str = if memo.starts_with("WD-") {
    &memo[3..]
} else {
    memo
};

// Look up transaction to get expected amount
let tx_repo = TransactionRepository::new(self.pool.clone());
let expected_amount = match tx_repo.find_by_id(tx_id_str).await {
    Ok(Some(db_tx)) => {
        // Convert cngn_amount to string with proper format
        db_tx.cngn_amount.to_string()
    }
    _ => {
        // If we can't find it, we still try to verify as cNGN payment
        // (in case it's a new transaction not yet in DB)
        String::new()
    }
};

// Verify payment with amount check
let verification = self
    .is_incoming_cngn_payment(
        &tx.hash,
        system_wallet,
        if !expected_amount.is_empty() {
            Some(&expected_amount)
        } else {
            None
        },
    )
    .await
    .unwrap_or(PaymentVerificationResult::not_cngn());

// Handle verification results
if !verification.is_cngn_payment {
    // Not a cNGN payment, skip
    continue;
}

if !verification.amount_matches && verification.error.is_some() {
    // Amount mismatch - log but don't process
    warn!(
        memo = %memo,
        error = %verification.error.as_ref().unwrap(),
        "incoming payment amount mismatch; transaction not updated"
    );
    self.log_webhook_event_for_mismatch(
        tx_id_str,
        &tx,
        verification.error.as_deref().unwrap_or("unknown error"),
    ).await;
    continue;
}

// Amount verified, process normally (existing code continues...)
```

### 5. Add Helper Method for Amount Mismatch Events

**New Method** (add to `TransactionMonitorWorker` impl block):

```rust
async fn log_webhook_event_for_mismatch(
    &self,
    transaction_id: &str,
    tx: &HorizonTransactionRecord,
    reason: &str,
) {
    let parsed_tx_id = Uuid::parse_str(transaction_id).ok();
    let repo = WebhookRepository::new(self.pool.clone());
    let event_id = format!("mismatch:{}", tx.hash);
    let payload = json!({
        "transaction_id": transaction_id,
        "reason": reason,
        "hash": tx.hash,
        "ledger": tx.ledger,
        "created_at": tx.created_at,
        "memo": tx.memo,
    });
    if let Err(e) = repo
        .log_event(
            &event_id,
            "stellar",
            "stellar.incoming.amount_mismatch",
            payload,
            None,
            parsed_tx_id,
        )
        .await
    {
        warn!(
            transaction_id = %transaction_id,
            error = %e,
            "failed to log amount mismatch event"
        );
    }
}
```

---

## Testing the Enhancement

### Unit Test

Add to the `tests` module at the bottom of `transaction_monitor.rs`:

```rust
#[test]
fn payment_verification_detects_amount_mismatch() {
    let result = PaymentVerificationResult::amount_mismatch("50000.0000000", "49500.0000000");
    assert!(!result.amount_matches);
    assert!(result.is_cngn_payment);
    assert!(result.error.is_some());
    assert!(result.error.unwrap().contains("Amount mismatch"));
}

#[test]
fn payment_verification_success() {
    let result = PaymentVerificationResult::with_amount("50000.0000000".to_string());
    assert!(result.is_cngn_payment);
    assert!(result.amount_matches);
    assert_eq!(result.amount, Some("50000.0000000".to_string()));
}

#[test]
fn payment_verification_not_cngn() {
    let result = PaymentVerificationResult::not_cngn();
    assert!(!result.is_cngn_payment);
    assert!(!result.amount_matches);
}
```

### Integration Test Scenario

**Manual Test** (20 minutes):

1. **Create withdrawal transaction**
   ```bash
   curl -X POST http://localhost:3000/api/offramp/initiate \
     -H "Content-Type: application/json" \
     -d '{
       "wallet_address": "G...",
       "quote_id": "quote-123",
       "bank_code": "058",
       "account_number": "0123456789",
       "cngn_amount": 50000.0
     }'
   ```
   Note: Returns `cngn_amount: 50000.0` and `payment_memo: WD-9F8E7D6C`

2. **Test Exact Amount Match** (Should succeed)
   ```
   Send to: {system_wallet}
   Asset: cNGN
   Amount: 50000.0
   Memo: WD-9F8E7D6C
   ```
   Expected: Status updates to `cngn_received`

3. **Test Amount Mismatch** (Should fail to process)
   - Create another transaction (same process)
   - Send WRONG amount (e.g., 49500.0 instead of 50000.0)
   - Memo: WD-{new_memo}
   - Wait for monitor
   - Expected: Status stays `pending_payment`, mismatch event logged

4. **Verify Logs**
   ```bash
   grep "incoming payment amount mismatch" logs/app.log
   # Should see warning with details
   ```

5. **Check Webhook Events**
   ```bash
   psql -c "SELECT payload FROM webhook_events 
           WHERE event_type = 'stellar.incoming.amount_mismatch' 
           ORDER BY created_at DESC LIMIT 1;"
   ```

---

## Database Considerations

### Schema Updates (if needed)

Currently, the `metadata` JSONB field stores:
- `incoming_amount` - Amount received from Stellar

**No schema changes needed** - the enhancement uses existing fields.

### Metadata with Amount Verification

```json
{
    "incoming_hash": "abc123...",
    "incoming_ledger": 12345,
    "incoming_confirmed_at": "2026-02-24T12:00:00Z",
    "incoming_amount": "50000.0000000",
    "amount_verified": true,
    "amount_match_status": "exact_match",  // NEW
    "expected_cngn_amount": 50000.0,       // NEW (for audit)
    "matched_at": "2026-02-24T12:00:01Z"
}
```

### Monitoring Mismatches

Query for amount mismatches:
```sql
SELECT transaction_id, metadata, created_at 
FROM webhook_events 
WHERE event_type = 'stellar.incoming.amount_mismatch'
ORDER BY created_at DESC;
```

---

## Error Handling

### Network Issues During Verification

If `get_transaction_operations()` fails:
- Error is caught and returned to caller
- Caller logs it and continues
- Transaction remains `pending_payment`
- Will retry in next monitor cycle

### Amount Parsing Issues

If amount cannot be parsed from Stellar operation:
```rust
// Treated as "not found" - no payment detected
return Ok(PaymentVerificationResult::not_cngn());
```

### Edge Cases

**Dust Amounts**: 
- Stellar uses 7 decimal places
- "1.0000000" == "1" == "1.0" when compared as strings
- Use `BigDecimal` for exact comparison

**Very Large Amounts**:
- Already handled by `BigDecimal` type in DB
- Stellar can handle up to 922,337,203,685.4775807 stroops

---

## Performance Impact

### Overhead

- **Additional API call**: Already happening (operations fetch)
- **Additional DB lookup**: Early lookup to get expected amount
- **String comparison**: O(1) operation
- **Estimated latency**: +1-2ms per payment

### Optimization Options (Future)

```rust
// Cache expected amounts in memory (if needed)
pub struct PaymentVerificationCache {
    // Map from memo → (expected_amount, expires_at)
}

// Or use transaction lookup only on first mismatch attempt
// (some payments might arrive quickly, before DB has synced)
```

---

## Deployment Notes

### Backwards Compatibility

✅ **Fully backwards compatible**
- Existing transactions without amount will bypass check
- Existing code paths remain unchanged
- Only new payments with expected_amount are verified

### Migration

**No migration needed**:
1. Deploy code enhancement
2. Restart transaction monitor worker
3. New deployments automatically verify amounts
4. Existing pending transactions unaffected

### Rollback

If issues arise:
1. Revert the code changes
2. Restart worker
3. Remove new webhook events (optional)

---

## Monitoring & Alerts

### Logs to Watch

```
"cNGN payment amount mismatch"
  tx_hash=...
  expected_amount=50000.0000000
  received_amount=49500.0000000
```

### Alert Thresholds

Consider setting alerts for:
- **Amount mismatches** > 5 per hour (indicates user confusion)
- **Verification errors** > 10% of payments (indicates network issues)

### Audit Trail

All amount verification results are logged:
```
webhook_events.event_type = 'stellar.incoming.amount_mismatch'
webhook_events.payload.reason = Full error details
webhook_events.created_at = Timestamp
```

---

## Future Enhancements

1. **Tolerance Range**: Allow ±0.01% tolerance for rounding
2. **Multi-currency Support**: If supporting multiple stablecoins
3. **Partial Payments**: Handle payments in installments
4. **Exchange Rate Updates**: Auto-adjust expected amount if rate changed

---

## References

**Implementation Location**: `src/workers/transaction_monitor.rs`

**Related Methods**:
- `scan_incoming_transactions()` - Main entry point
- `is_incoming_cngn_payment()` - Enhanced method
- `get_transaction_operations()` - Stellar client method

**Related Issues**:
- Issue #62: Withdrawal transaction creation
- Issue #34: Withdrawal processor
- Issue #12: Payment monitoring

---

**Status**: ✅ READY FOR IMPLEMENTATION  
**Estimated Time to Implement**: 30-45 minutes  
**Testing Time**: 20 minutes  
**Risk Level**: LOW (backwards compatible, non-breaking change)
