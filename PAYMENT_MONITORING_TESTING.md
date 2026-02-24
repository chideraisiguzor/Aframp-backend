# Payment Monitoring Testing Checklist - Issue #12

## Overview

This document provides comprehensive testing procedures for the payment monitoring system. Use this checklist to verify the system works correctly before deployment and after changes.

**Estimated Duration**: 45-60 minutes for complete testing

---

## Pre-Testing Checklist

Before running tests, verify:

- [ ] Application is running (`cargo run --release` or Docker)
- [ ] Database is initialized with migrations applied
- [ ] Redis cache is running (if used)
- [ ] System wallet address is configured and funded
- [ ] Sufficient Testnet XLM in system wallet for fees
- [ ] Corresponding BankAccount created for test user
- [ ] Test Stellar wallet has cNGN to send
- [ ] Network connectivity to Horizon API is good
- [ ] Application logs are accessible

**Quick Health Check**:
```bash
# Check application health
curl http://localhost:3000/health

# Verify worker started
docker logs aframp-backend 2>&1 | grep "transaction monitor worker started"
```

---

## Section 1: Configuration Verification

### Test 1.1: Configuration Loading

**Objective**: Verify all configuration variables are loaded correctly

**Steps**:
1. Check startup logs for configuration summary
2. Verify system wallet address is shown

**Expected Output** (in logs):
```
stellar transaction monitor worker started
  poll_interval_secs=7
  pending_timeout_secs=600
  max_retries=5
  has_system_wallet=true
```

**Result**: ✅ PASS / ❌ FAIL

---

### Test 1.2: Stellar Network Connection

**Objective**: Verify connection to Horizon API

**Steps**:
1. Run application
2. Check for Horizon connectivity errors

**Expected**:
- No network timeout errors in logs
- Successful account queries should appear in logs

**Commands**:
```bash
# Manually verify Horizon
curl -s https://horizon-testnet.stellar.org/health | jq

# Should return status "ok"
```

**Result**: ✅ PASS / ❌ FAIL

---

## Section 2: Database Integration

### Test 2.1: Transaction Storage

**Objective**: Verify transactions are stored correctly in database

**Steps**:
1. Create a withdrawal transaction via API
2. Verify it's stored in database with correct status

**API Call**:
```bash
curl -X POST http://localhost:3000/api/offramp/initiate \
  -H "Content-Type: application/json" \
  -d '{
    "wallet_address": "GBUQWP3BOUZX34ULNQG23RQ6F5LGNQJKPKTHMYCM2747BEONEATLCSE7",
    "quote_id": "quote-test-001",
    "bank_code": "058",
    "account_number": "0123456789",
    "cngn_amount": 50000.0
  }'
```

**Database Query**:
```bash
psql -c "SELECT transaction_id, status, cngn_amount 
         FROM transactions 
         WHERE status = 'pending_payment' 
         LIMIT 1;"
```

**Expected**:
- Transaction appears in database
- Status is `pending_payment`
- cngn_amount matches request

**Result**: ✅ PASS / ❌ FAIL

---

### Test 2.2: Metadata Storage

**Objective**: Verify transaction metadata is stored correctly

**Steps**:
1. Query transaction metadata
2. Verify payment_memo is set

**Database Query**:
```bash
psql -c "SELECT transaction_id, 
         metadata->>'payment_memo' as memo,
         metadata->>'quote_id' as quote_id
         FROM transactions 
         WHERE status = 'pending_payment' 
         LIMIT 1;"
```

**Expected**:
- payment_memo is in format WD-{8_hex_chars}
- quote_id matches request
- Other metadata fields present

**Result**: ✅ PASS / ❌ FAIL

---

## Section 3: Payment Detection

### Test 3.1: Unconfirmed Payment (Pending)

**Objective**: Verify system detects incoming cNGN on Stellar ledger

**Setup**:
- Have a test Stellar account with cNGN
- Know the system wallet address
- Have payment memo from test

**Steps**:
1. Send cNGN payment to system wallet:
   - Amount: Exactly match the `cngn_amount` from transaction
   - To: System wallet address
   - Asset: cNGN
   - Memo: WD-{memo_from_transaction}
2. Wait for transaction to confirm on ledger (5-10 seconds)
3. Check if monitor process picks it up

**Stellar Command** (if using Stellar CLI):
```bash
# Assuming system wallet funding account is configured
stellar tx payment --destination $SYSTEM_WALLET \
  --amount 50000 \
  --asset cNGN:$CNGN_ISSUER \
  --memo-text "WD-9F8E7D6C" \
  --source-account test-account \
  --submit
```

**Wait**: 
```bash
# Monitor runs every 7 seconds by default
sleep 15  # Wait 15 seconds for next cycle
```

**Database Query**:
```bash
psql -c "SELECT transaction_id, status, metadata->>'incoming_hash' 
         FROM transactions 
         WHERE status = 'cngn_received'
         LIMIT 1;"
```

**Expected**:
- Transaction status changes to `cngn_received`
- metadata.incoming_hash is populated
- Payment found on Stellar ledger

**Log Output**:
```
incoming cNGN payment matched and updated
  transaction_id=...
  incoming_hash=...
  ledger=...
  status=cngn_received
```

**Result**: ✅ PASS / ❌ FAIL

---

### Test 3.2: Amount Verification (Exact Match)

**Objective**: Verify amount matches expected amount exactly

**Prerequisites**:
- Enhanced version with amount verification implemented

**Setup**:
1. Create new transaction with specific amount
   - Note the exact cngn_amount (e.g., 50000.0000000)

**Steps**:
1. Send payment with EXACT amount
2. Monitor should match and update status
3. Verify in database

**Expected**:
- Status changes to `cngn_received`
- Amount verified successfully
- No amount_mismatch event logged

**Result**: ✅ PASS / ❌ FAIL

---

### Test 3.3: Amount Mismatch Detection

**Objective**: Verify system rejects payments with wrong amount

**Prerequisites**:
- Enhanced version with amount verification implemented

**Setup**:
1. Create transaction with amount 50000.0
2. Send payment with different amount (e.g., 49500.0)
3. Monitor should detect mismatch

**Steps**:
1. Send cNGN payment:
   - Amount: 49500.0 (NOT 50000.0)
   - To: System wallet
   - Memo: WD-{memo}
2. Wait for monitor cycle
3. Check logs and database

**Expected**:
- Status remains `pending_payment`
- Log shows amount mismatch warning
- Webhook event logged: stellar.incoming.amount_mismatch

**Log Check**:
```bash
grep "incoming payment amount mismatch" logs/app.log
```

**Webhook Query**:
```bash
psql -c "SELECT event_type, payload 
         FROM webhook_events 
         WHERE event_type = 'stellar.incoming.amount_mismatch' 
         ORDER BY created_at DESC LIMIT 1;"
```

**Expected Webhook Payload**:
```json
{
  "transaction_id": "...",
  "reason": "Amount mismatch: expected 50000.0000000, received 49500.0000000",
  "hash": "...",
  "ledger": ...
}
```

**Result**: ✅ PASS / ❌ FAIL

---

## Section 4: Polling and Retry Logic

### Test 4.1: Monitor Polling Frequency

**Objective**: Verify monitor runs at configured interval

**Setup**:
- Set `TX_MONITOR_POLL_INTERVAL_SECONDS=5` for faster testing

**Steps**:
1. Enable debug logging:
   ```bash
   export RUST_LOG=transaction_monitor=debug
   ```
2. Start application
3. Wait 30 seconds
4. Count polls in logs

**Expected**:
- Should see ~6 poll cycles (30 sec ÷ 5 sec)
- Log entries like: "scan_incoming_transactions starting"
- No errors between cycles

**Log Pattern**:
```
scan_incoming_transactions completed
  transactions_processed=0
[5 second gap]
scan_incoming_transactions completed
  transactions_processed=0
```

**Result**: ✅ PASS / ❌ FAIL

---

### Test 4.2: Retry Exponential Backoff

**Objective**: Verify transient errors trigger exponential backoff

**Setup**:
- Have a transaction in pending_payment status
- Force a transient error (network, rate limit)

**Steps**:
1. Create a transaction waiting for payment
2. Block network (or wait for rate limit)
3. Monitor should retry with backoff
4. Verify retry pattern in logs

**Expected Retry Schedule**:
```
Attempt 1: 0s delay
Attempt 2: 10s delay
Attempt 3: 30s delay
Attempt 4: 120s (2 min) delay
Attempt 5: 300s (5 min) delay
Attempt 6+: 600s (10 min) delay
```

**Verify in Logs**:
```bash
grep "retry" logs/app.log | grep "next_retry_after_secs"
```

**Expected Output**:
```
transaction failed with retryable error; scheduled for retry
  retry_count=1
  next_retry_after_secs=10
```

**Result**: ✅ PASS / ❌ FAIL

---

## Section 5: Webhook Events

### Test 5.1: Payment Received Event

**Objective**: Verify webhook event is logged when payment confirmed

**Steps**:
1. Send cNGN payment (from Section 3.1)
2. Wait for monitor to detect it
3. Query webhook events

**Database Query**:
```bash
psql -c "SELECT event_type, payload, created_at 
         FROM webhook_events 
         WHERE event_type LIKE 'stellar.%' 
         ORDER BY created_at DESC LIMIT 5;"
```

**Expected Events** (in order):
1. `stellar.offramp.received` - Payment detected
2. `stellar.incoming.matched` - Transaction matched

**Event Payload Example**:
```json
{
  "incoming_hash": "abc123...",
  "incoming_ledger": 12345,
  "incoming_confirmed_at": "2026-02-24T12:00:00Z"
}
```

**Result**: ✅ PASS / ❌ FAIL

---

### Test 5.2: Unmatched Payment Event

**Objective**: Verify unmatched payments are logged

**Steps**:
1. Send cNGN payment to system wallet
2. Use memo that doesn't match any transaction (e.g., "WD-NOMATCH")
3. Wait for monitor
4. Check webhook events

**Expected**:
- Event type: `stellar.incoming.unmatched`
- Payload contains memo, hash, ledger

**Result**: ✅ PASS / ❌ FAIL

---

## Section 6: Status Transitions

### Test 6.1: Complete Flow: pending_payment → cngn_received

**Objective**: End-to-end test of transaction status flow

**Timeline**:
1. T=0s: Create transaction → status: `pending_payment`
2. T=1s: Send cNGN payment
3. T=5-10s: Stellar confirms transaction
4. T=12-20s: Monitor detects payment → status: `cngn_received`

**Steps**:
```bash
# 1. Create transaction
TX_RESPONSE=$(curl -X POST http://localhost:3000/api/offramp/initiate \
  -H "Content-Type: application/json" \
  -d '{
    "wallet_address": "GBUQWP3BOUZX34ULNQG23RQ6F5LGNQJKPKTHMYCM2747BEONEATLCSE7",
    "quote_id": "quote-flow-001",
    "bank_code": "058",
    "account_number": "0123456789",
    "cngn_amount": 50000.0
  }')

# Extract values from response
TX_ID=$(echo $TX_RESPONSE | jq -r '.transaction.transaction_id')
SYSTEM_WALLET=$(echo $TX_RESPONSE | jq -r '.system_wallet_info.wallet_address')
CNGN_AMOUNT=$(echo $TX_RESPONSE | jq -r '.system_wallet_info.cngn_amount')
MEMO=$(echo $TX_RESPONSE | jq -r '.system_wallet_info.payment_memo')

echo "Transaction: $TX_ID"
echo "Send to: $SYSTEM_WALLET"
echo "Amount: $CNGN_AMOUNT"
echo "Memo: $MEMO"

# 2. Check initial status
psql -c "SELECT status FROM transactions WHERE transaction_id = '$TX_ID';"
# Expected: pending_payment

# 3. Send payment (via Stellar CLI or SDK)
stellar tx payment --destination "$SYSTEM_WALLET" \
  --amount "$CNGN_AMOUNT" \
  --asset "cNGN:$CNGN_ISSUER" \
  --memo-text "$MEMO" \
  --submit

# 4. Wait for confirmation (5-10s) and monitoring (7-20s more)
sleep 30

# 5. Check final status
psql -c "SELECT status, metadata->>'incoming_hash' FROM transactions WHERE transaction_id = '$TX_ID';"
# Expected: cngn_received, hash populated
```

**Expected Status Journey**:
```
Time    Status              Event
0s      pending_payment     Transaction created
1s      pending_payment     Payment sent to Stellar
10s     pending_payment     Stellar confirms transaction
20s     cngn_received       Monitor detects payment
```

**Result**: ✅ PASS / ❌ FAIL

---

## Section 7: Error Handling

### Test 7.1: Non-existent Transaction

**Objective**: Verify unmatched payments don't crash system

**Steps**:
1. Send cNGN with memo that doesn't exist: "WD-INVALID123"
2. Monitor should handle gracefully
3. Log unmatched event

**Expected**:
- No crashes or exceptions
- Webhook event: `stellar.incoming.unmatched`
- Application continues running normally

**Result**: ✅ PASS / ❌ FAIL

---

### Test 7.2: Network Error Recovery

**Objective**: Verify system recovers from transient network errors

**Setup**:
- Pending transaction waiting for payment

**Steps**:
1. Temporarily block network to Horizon
2. Wait for monitor cycle
3. Restore network
4. Wait for next cycle

**Expected**:
- Transient error is logged
- Transaction scheduled for retry
- After network restored, monitor continues
- No permanent failures from transient issues

**Result**: ✅ PASS / ❌ FAIL

---

### Test 7.3: Invalid Configuration

**Objective**: Verify appropriate errors for bad config

**Steps**:
1. Set `SYSTEM_WALLET_ADDRESS` to invalid value
2. Start application
3. Check logs

**Expected**:
- Startup error logged clearly
- Error message indicates invalid address
- Application doesn't run if wallet config missing

**Result**: ✅ PASS / ❌ FAIL

---

## Section 8: Performance & Load

### Test 8.1: Single Payment Latency

**Objective**: Measure time from payment sent to status update

**Steps**:
```bash
# Record send time
SEND_TIME=$(date +%s%N | cut -b1-13)

# Send payment and note hash from Stellar explorer

# Wait and check DB
sleep 20

# Record update time
UPDATE_TIME=$(date +%s%N | cut -b1-13)
```

**Expected**:
- Payment detected within 20-30 seconds
- Latency factors:
  - Stellar confirmation: 5-10s
  - Monitor poll interval: 0-7s
  - Processing: 1-2s
  - Total: 10-20s typical, 30s worst case

**Result**: ✅ PASS | Latency: **____** seconds

---

### Test 8.2: Multiple Concurrent Payments

**Objective**: Verify system handles multiple payments

**Setup**:
- Create 5 test transactions

**Steps**:
1. Send 5 cNGN payments in quick succession
2. All to system wallet
3. Different memos matching transactions
4. Wait for processing

**Expected**:
- All 5 payments detected and status updated
- All within one polling cycle or next one
- No errors or missed payments

**Result**: ✅ PASS / ❌ FAIL | Processed: **5**/5

---

## Section 9: Logging & Observability

### Test 9.1: Structured Logging

**Objective**: Verify logs contain necessary information

**Setup**:
- Enable debug logging: `RUST_LOG=debug`

**Steps**:
1. Send payment through complete flow
2. Examine logs

**Expected Log Fields**:
```json
{
  "timestamp": "2026-02-24T12:00:00Z",
  "level": "INFO",
  "target": "transaction_monitor",
  "message": "incoming cNGN payment matched and updated",
  "transaction_id": "...",
  "incoming_hash": "...",
  "ledger": 12345,
  "status": "cngn_received"
}
```

**Result**: ✅ PASS / ❌ FAIL

---

### Test 9.2: Error Logging

**Objective**: Verify error messages are helpful

**Steps**:
1. Trigger various error scenarios
2. Check error messages

**Expected Error Messages** (descriptive):
```
"Amount mismatch: expected 50000.0000000, received 49500.0000000"
"Payment not found in database, transaction_id: invalid-id"
"Horizon API timeout after 30 seconds"
```

**Result**: ✅ PASS / ❌ FAIL

---

## Section 10: Integration Points

### Test 10.1: Status Ready for Withdrawal Processor

**Objective**: Verify status change triggers next system

**Setup**:
- Have Issue #34 (Withdrawal Processor) available to test

**Steps**:
1. Send payment through complete flow
2. Status becomes `cngn_received`
3. Withdrawal processor can read and process transaction

**Expected**:
- Withdrawal processor queries for `cngn_received` status
- Finds transaction
- Begins withdrawal processing

**Result**: ✅ PASS / ❌ FAIL

---

## Post-Testing Summary

### Test Results Table

| Test | Result | Notes |
|------|--------|-------|
| 1.1: Configuration Loading | ✅/❌ | |
| 1.2: Network Connection | ✅/❌ | |
| 2.1: Transaction Storage | ✅/❌ | |
| 2.2: Metadata Storage | ✅/❌ | |
| 3.1: Payment Detection | ✅/❌ | |
| 3.2: Amount Exact Match | ✅/❌ | |
| 3.3: Amount Mismatch | ✅/❌ | |
| 4.1: Polling Frequency | ✅/❌ | |
| 4.2: Retry Backoff | ✅/❌ | |
| 5.1: Payment Events | ✅/❌ | |
| 5.2: Unmatched Events | ✅/❌ | |
| 6.1: Complete Flow | ✅/❌ | Latency: __s |
| 7.1: Invalid Transaction | ✅/❌ | |
| 7.2: Network Recovery | ✅/❌ | |
| 7.3: Invalid Config | ✅/❌ | |
| 8.1: Single Payment | ✅/❌ | Latency: __s |
| 8.2: Multiple Payments | ✅/❌ | Processed: __/5 |
| 9.1: Structured Logging | ✅/❌ | |
| 9.2: Error Logging | ✅/❌ | |
| 10.1: Integration Ready | ✅/❌ | |

**Total Passed**: ____/20

### Sign-Off

- **Tester Name**: __________________
- **Date**: __________________
- **Environment**: TESTNET / MAINNET
- **Approved**: ✅ YES / ❌ NO

### Notes

```
[Space for additional notes, issues found, or comments]
```

---

## Regression Testing

Use this checklist after any code changes:

- [ ] Core monitoring still works (Test 3.1)
- [ ] Amount verification still works (Test 3.2, 3.3)
- [ ] Polling frequency unchanged (Test 4.1)
- [ ] Webhook events still logged (Test 5.1)
- [ ] No memory leaks in polling loop
- [ ] Logs still readable and useful
- [ ] Error messages still helpful

---

## Troubleshooting During Testing

### Payment Not Showing in Monitor

**Debug Steps**:
1. Verify payment on Stellar: `curl https://horizon-testnet.stellar.org/accounts/$WALLET/transactions`
2. Check memo format exactly matches
3. Check asset code is "cNGN"
4. Verify destination is system wallet
5. Check transaction is in database in `pending_payment` status

### Logs Not Appearing

```bash
# Check logging level
export RUST_LOG=debug
cargo run 2>&1 | tee logs.txt

# Or for Docker
docker logs -f aframp-backend
```

### Database Query Issues

```bash
# Validate PostgreSQL connection
psql -U postgres -d aframp -c "SELECT NOW();"

# List recent transactions
psql -c "SELECT transaction_id, status, created_at FROM transactions ORDER BY created_at DESC LIMIT 10;"
```

---

**Last Updated**: February 24, 2026  
**Status**: ✅ COMPLETE - Ready for Testing
