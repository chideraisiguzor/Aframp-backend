# Payment Monitoring Quick Reference Card

## One-Page Cheat Sheet

### The Problem
Aframp needs to monitor incoming cNGN payments to the system wallet and match them to withdrawal transactions so the withdrawal processor can complete withdrawals.

### The Solution
Transaction Monitor Worker watches Stellar blockchain every 7 seconds, detects cNGN payments, matches them to transactions via memo (WD-*), and updates status to `cngn_received`.

---

## Status Flow (Simple)

```
pending_payment ──(user sends cNGN)──> cngn_received ──(processor)──> completed
```

---

## Key Configuration (3 Required)

```bash
export STELLAR_NETWORK=TESTNET
export SYSTEM_WALLET_ADDRESS=G...
export CNGN_ISSUER_TESTNET=G...
```

---

## What Monitor Does

| When | What | Where |
|------|------|-------|
| Every 7 sec | Query system wallet | Stellar Horizon |
| On payment | Extract memo (WD-*) | Transaction data |
| On match | Verify amount ⭐ NEW | Payment operation |
| On success | Update status | PostgreSQL DB |
| Always | Log event | webhook_events |

---

## Implementation Status

| Component | Status | Work Needed |
|-----------|--------|------------|
| Core monitoring | ✅ Done (915 lines) | None |
| Amount verification | ⚠️ Partial | Add ~150 lines |
| Configuration | ✅ Done | Set 3 vars |
| Testing | ✅ Available | Run 20 tests |
| Documentation | ✅ Complete (5 files) | Read 1-2 before coding |

---

## Quick Start (5 Steps)

### 1. Understand (15 min)
```bash
Read: PAYMENT_MONITORING_SETUP.md
# Learn architecture and flow
```

### 2. Configure (5 min)
```bash
# Set environment variables
export STELLAR_NETWORK=TESTNET
export SYSTEM_WALLET_ADDRESS=GAB5VYQBOVP2R7R7...
export CNGN_ISSUER_TESTNET=GBNRQ4REC45UCRQMD...
```

### 3. Enhance (45 min)
```bash
# Follow PAYMENT_MONITORING_ENHANCEMENT.md
# Add PaymentVerificationResult type
# Enhance is_incoming_cngn_payment()
# Add mismatch logging
# Write unit tests
```

### 4. Test (45 min)
```bash
# Follow PAYMENT_MONITORING_TESTING.md
# Run 20 test cases
# Verify all pass
# Sign off checklist
```

### 5. Deploy (15 min)
```bash
cargo run --release
# Monitor logs
# Set up alerts
```

---

## Code Changes Required (3 Locations)

### 1. Add Type (20 lines)
**File**: `src/workers/transaction_monitor.rs` (top of file)
```rust
pub struct PaymentVerificationResult {
    pub is_cngn_payment: bool,
    pub amount: Option<String>,
    pub amount_matches: bool,
    pub error: Option<String>,
}
```

### 2. Modify Method (30 lines)
**File**: `src/workers/transaction_monitor.rs` (is_incoming_cngn_payment)
- Add expected_amount parameter
- Compare amounts
- Return PaymentVerificationResult

### 3. Update Call Site (50 lines)
**File**: `src/workers/transaction_monitor.rs` (scan_incoming_transactions)
- Look up expected amount first
- Call enhanced is_incoming_cngn_payment
- Handle amount mismatch

---

## Test Scenarios (Key 5)

### Test 1: Payment Detected ✅
- Send cNGN to system wallet
- Memo: WD-{hash}
- Wait 20 seconds
- Status changes to `cngn_received`

### Test 2: Amount Matches ✅
- Send exact amount from transaction
- Should process successfully
- Status updates

### Test 3: Amount Mismatch ⭐ NEW
- Send wrong amount
- Should NOT process
- Webhook event: stellar.incoming.amount_mismatch

### Test 4: Retry on Error ✅
- Transient error occurs
- Monitor retries with backoff
- Eventually succeeds or fails

### Test 5: Multiple Payments ✅
- Send 5 payments
- All detected and processed
- Within 1-2 polling cycles

---

## Logs to Expect

```
[INFO] stellar transaction monitor worker started
       has_system_wallet=true
       poll_interval_secs=7

[INFO] incoming cNGN payment matched and updated
       transaction_id=...
       incoming_hash=...
       status=cngn_received

[WARN] cNGN payment amount mismatch (when NEW feature active)
       expected=50000.0000000
       received=49500.0000000
```

---

## Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `invalid address` | Bad wallet address | Check $SYSTEM_WALLET_ADDRESS |
| `config error: CNGN_ISSUER not set` | Missing issuer | Set CNGN_ISSUER_TESTNET or MAINNET |
| `Payment not detected` | Transaction not on ledger | Wait 5-10 sec, check Horizon directly |
| `Amount mismatch` | Wrong amount sent | Resend with exact cngn_amount |
| `Cannot find transaction` | Memo doesn't match | Verify memo format WD-{hash} |

---

## File Locations

| Purpose | File | Lines |
|---------|------|-------|
| Monitor logic | `src/workers/transaction_monitor.rs` | 915 |
| Stellar client | `src/chains/stellar/client.rs` | 502 |
| Database ops | `src/database/transaction_repository.rs` | varies |
| Webhook events | `src/database/webhook_repository.rs` | varies |

---

## DB Schema (Relevant Fields)

```sql
-- What gets updated when payment found
transactions.status = 'cngn_received'

-- What gets stored in metadata
metadata['incoming_hash'] = 'abc123...'
metadata['incoming_ledger'] = 12345
metadata['incoming_confirmed_at'] = '2026-02-24T12:00:00Z'
metadata['incoming_amount'] = '50000.0000000'  -- NEW with enhancement
metadata['amount_verified'] = true             -- NEW with enhancement
```

---

## Integration Points

### Upstream (Input)
- **Issue #62** (offramp/initiate)
  - Creates transactions in `pending_payment` status
  - Provides system wallet address
  - Returns payment memo

### Downstream (Output)
- **Issue #34** (Withdrawal Processor)
  - Looks for `cngn_received` status
  - Processes payment automatically
  - Updates to `completed`

---

## Performance

| Metric | Value |
|--------|-------|
| Poll interval | 7 seconds |
| Detection latency | 15-25 seconds |
| Webhook event overhead | <50ms |
| Database update | <10ms |
| Scalable to | 10,000 tx/day |

---

## Monitoring Metrics

```
monitor_cycle_seconds: How long each poll cycle takes
payments_detected: Payments matched per cycle
payment_latency: Time from send to status update
amount_mismatches: (NEW) Wrong amount payments
verification_errors: Failed verification attempts
webhook_events_logged: Audit trail events
retry_count: Retry attempts needed
```

---

## Alerting Suggestions

- Monitor not running > 30 sec ❌
- Amount mismatches > 5 per hour ⚠️
- Verification errors increasing ⚠️
- Payment latency > 60 sec ⚠️
- Database errors increasing ❌

---

## Environment: Development Setup

```bash
STELLAR_NETWORK=TESTNET
HORIZON_URL=https://horizon-testnet.stellar.org
SYSTEM_WALLET_ADDRESS=GBNRQ4REC45UCRQMDQ5RGZDZXXOXUGKPZFVVQQFVBW6XZZQZFVZZRUXYM
CNGN_ISSUER_TESTNET=GCNYYVQFJ4YXHXNHW64LJ7UYIYF7QJVVNPZ2QBSDRZJJ3FL3NOFV4DI
TX_MONITOR_POLL_INTERVAL_SECONDS=7
DATABASE_URL=postgres://user:pass@localhost/aframp
RUST_LOG=debug
```

---

## Environment: Production Setup

```bash
STELLAR_NETWORK=MAINNET
# HORIZON_URL omitted - uses default
SYSTEM_WALLET_ADDRESS=${SECRET_SYSTEM_WALLET}
SYSTEM_WALLET_SECRET=${SECRET_SYSTEM_WALLET_SECRET}
CNGN_ISSUER_MAINNET=${SECRET_CNGN_ISSUER}
TX_MONITOR_POLL_INTERVAL_SECONDS=10
TX_MONITOR_PENDING_TIMEOUT_SECONDS=1800
DATABASE_URL=postgres://...
RUST_LOG=info
```

---

## Common Commands

```bash
# See if monitor is running
docker logs aframp-backend | grep "transaction monitor"

# Check pending transactions
psql -c "SELECT transaction_id, status FROM transactions WHERE status='pending_payment';"

# Check processed transactions
psql -c "SELECT transaction_id, status, metadata->>'incoming_hash' FROM transactions WHERE status='cngn_received';"

# Check webhook events
psql -c "SELECT event_type, created_at FROM webhook_events WHERE event_type LIKE 'stellar%' ORDER BY created_at DESC LIMIT 10;"

# Test amount verification (after enhancement)
curl "http://localhost:3000/api/paymentmonitor/test" -d '{"amount":"50000.0000000"}'
```

---

## Documentation Map

```
START HERE
    ↓
ISSUE_12_PAYMENT_MONITORING_GUIDE.md (this file's overview)
    ↓
    ├─→ PAYMENT_MONITORING_SETUP.md (detailed architecture)
    ├─→ PAYMENT_MONITORING_ENHANCEMENT.md (code implementation)
    ├─→ PAYMENT_MONITORING_CONFIGURATION.md (env vars & tuning)
    └─→ PAYMENT_MONITORING_TESTING.md (20 test procedures)
```

---

## Success Criteria

✅ When complete, you should see:

1. **Logs**: `incoming cNGN payment matched and updated`
2. **Database**: Transaction status changes to `cngn_received`
3. **Webhook**: `stellar.offramp.received` event logged
4. **Performance**: Detection latency < 30 seconds
5. **Amount Verification**: Mismatches detected and logged ⭐
6. **Integration**: Issue #34 processor triggers automatically
7. **Testing**: All 20 tests passing
8. **Monitoring**: Alerts configured and working

---

## Time Estimates

| Task | Time |
|------|------|
| Read setup doc | 15 min |
| Set configuration | 5 min |
| Code enhancement | 45 min |
| Unit tests | 10 min |
| Integration tests | 30 min |
| Debugging/tweaking | 15 min |
| Deployment | 15 min |
| **Total** | **2-3 hours** |

---

## Emergency Troubleshooting

**Monitor not starting?**
```bash
# Check config
env | grep STELLAR
env | grep SYSTEM_WALLET

# Check logs
docker logs aframp | tail -50
```

**Payment not detected?**
```bash
# Check Horizon
curl https://horizon-testnet.stellar.org/accounts/$SYSTEM_WALLET/transactions

# Check database
psql -c "SELECT * FROM transactions WHERE status='pending_payment';"
```

**Amount mismatch occurring?**
```bash
# Check webhook events
psql -c "SELECT * FROM webhook_events WHERE event_type='stellar.incoming.amount_mismatch';"
```

---

## Next Steps After Implementation

1. ✅ Implement amount verification
2. ✅ Run all 20 tests
3. ✅ Deploy to staging
4. ✅ Monitor for 2 hours
5. ✅ Deploy to production
6. ✅ Set up monitoring/alerts
7. ✅ Create runbook for ops team
8. ✅ Document known issues

---

**Estimated Total Implementation & Testing Time: 2-3 hours**

**Status: ✅ READY TO START IMMEDIATELY**

See: [ISSUE_12_PAYMENT_MONITORING_GUIDE.md](./ISSUE_12_PAYMENT_MONITORING_GUIDE.md)
