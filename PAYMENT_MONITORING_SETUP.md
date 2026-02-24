# Payment Monitoring Setup - Issue #12

## Overview

Payment monitoring watches the Aframp system wallet for incoming cNGN payments and matches them to withdrawal transactions. When a payment is detected and verified, the transaction status is updated to `cngn_received`, triggering the withdrawal processor (Issue #34).

---

## Architecture

### Components

```
┌─────────────────────────────────────────────────────────────┐
│                   Stellar Blockchain                        │
│  (User sends cNGN to System Wallet with WD-* memo)          │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ├──→ Horizon API (Query payments)
                     │
        ┌────────────┴────────────────┐
        │                             │
        │   Transaction Monitor       │
        │   (src/workers/             │
        │    transaction_monitor.rs)  │
        │   - Query system wallet     │
        │   - Parse memo              │
        │   - Verify amount           │
        │   - Update DB               │
        │                             │
        └────────────────┬────────────┘
                         │
                         ├──→ PostgreSQL (Update transaction)
                         │
                         ├──→ Webhook Events (Log event)
                         │
                         └──→ Trigger Issue #34 (Withdrawal Processor)
```

### Flow Diagram

```
                    ╔════════════════════════════════╗
                    ║ User Initiates Withdrawal      ║
                    ║ (Issue #62 - offramp/initiate) ║
                    ╚══════════════┬═════════════════╝
                                   │
                    ┌──────────────┴──────────────┐
                    │                             │
                    ▼                             ▼
         Transaction Created          System Wallet Address
         in Database                  Returned to User
         Status: pending_payment       
                    │                             │
                    └──────────────┬──────────────┘
                                   │
                                   ▼
                    ╔════════════════════════════════╗
                    ║ User Sends cNGN Payment        ║
                    ║ - To: System Wallet            ║
                    ║ - Amount: cngn_amount          ║
                    ║ - Memo: WD-{8_hex}             ║
                    ╚══════════════┬═════════════════╝
                                   │
         ┌─────────────────────────┴─────────────────────────┐
         │                                                   │
         │   Payment on Stellar Ledger                       │
         │   (Search in Horizon API)                         │
         │                                                   │
         └─────────────────────────┬─────────────────────────┘
                                   │
        ╔══════════════════════════▼═══════════════════════╗
        ║  Transaction Monitor Worker (Polling)          ║
        ║  Every 7 seconds (configurable)                ║
        ║                                                 ║
        ║  1. Query /accounts/{system_wallet}/transactions
        ║  2. For each transaction:                       ║
        ║     - Check if successful                       ║
        ║     - Extract memo (WD-*)                       ║
        ║     - Get operations                            ║
        ║     - Verify cNGN payment                       ║
        ║     - Verify amount matches                     ║
        ║     - Look up transaction in DB                 ║
        ║                                                 ║
        ║  3. If all verified:                            ║
        ║     - Update status: cngn_received              ║
        ║     - Save Stellar hash                         ║
        ║     - Log webhook event                         ║
        ║     - Continues to next transaction             ║
        ╚══════════════════════════╦═══════════════════════╝
                                   │
         ┌─────────────────────────┴─────────────────────────┐
         │                                                   │
         ▼ Status: cngn_received                             ▼ Error
    ┌────────────────────────────┐               ┌────────────────────┐
    │ Database Updated            │               │ Logged as Failure  │
    │ - incoming_hash saved       │               │ Webhook event      │
    │ - incoming_ledger saved     │               │ sent to user       │
    │ - incoming_confirmed_at     │               │ tx stays pending   │
    │                             │               │                    │
    └──────────────┬──────────────┘               └────────────────────┘
                   │
                   ▼
    ╔═════════════════════════════════════════╗
    ║ Issue #34: Withdrawal Processor         ║
    ║ (Triggered automatically)               ║
    ║                                         ║
    ║ 1. Query for cngn_received status      ║
    ║ 2. Send NGN to user's bank             ║
    ║ 3. Update status: completed             ║
    ║ 4. Notify user                          ║
    ╚═════════════════════════════════════════╝
```

---

## Key Components

### 1. Transaction Monitor Worker

**File**: `src/workers/transaction_monitor.rs` (915 lines)

**Core Methods**:

- `run()` - Main loop that polls Stellar every 7 seconds
- `run_cycle()` - Executes both pending and incoming transaction checks
- `scan_incoming_transactions()` - Watches system wallet for incoming cNGN
- `is_incoming_cngn_payment()` - Verifies payment is cNGN to system wallet
- `process_pending_transactions()` - Handles outbound transaction confirmations
- `log_webhook_event()` - Records events for audit/notifications

**Status Flow for Offramp Transactions**:
```
pending_payment → (user sends cNGN) → cngn_received → (processor runs) → completed
```

**Attributes Monitored**:
- Transaction memo starts with `WD-`
- Asset is `cNGN` (Stellar asset)
- Destination is `SYSTEM_WALLET_ADDRESS`
- Asset issuer matches `CNGN_ISSUER` (testnet or mainnet)
- Amount matches transaction's `cngn_amount` ⭐ **[NEW - To Be Enhanced]**

### 2. Stellar Client

**File**: `src/chains/stellar/client.rs` (502 lines)

**Key Methods**:

```rust
// Query account transactions
pub async fn list_account_transactions(
    &self,
    account: &str,
    limit: usize,
    cursor: Option<&str>,
) -> StellarResult<HorizonTransactionsPage>

// Get transaction operations (including amounts)
pub async fn get_transaction_operations(&self, tx_hash: &str) 
    -> StellarResult<Vec<JsonValue>>

// Query a specific transaction
pub async fn get_transaction_by_hash(&self, tx_hash: &str)
    -> StellarResult<HorizonTransactionRecord>
```

### 3. Transaction Repository

**Database Operations**:

```rust
// Update transaction status and metadata
tx_repo.update_status_with_metadata(
    &transaction_id,
    "cngn_received",
    metadata
).await?

// Save the Stellar blockchain hash
tx_repo.update_blockchain_hash(
    &transaction_id,
    &stellar_hash
).await?

// Find transaction by ID (memo)
tx_repo.find_by_id(tx_id_str).await?
```

### 4. Webhook Events

**Events Logged**:

- `stellar.offramp.received` - cNGN payment received for offramp
- `stellar.incoming.matched` - cNGN payment matched to transaction
- `stellar.incoming.unmatched` - cNGN payment found but no matching transaction
- `stellar.transaction.confirmed` - Transaction confirmed on Stellar
- `stellar.transaction.timeout` - Transaction timed out
- `stellar.transaction.failed` - Transaction permanently failed

---

## What Gets Monitored

### System Wallet Configuration

**Environment Variables** (set in Docker/deployment):
```bash
SYSTEM_WALLET_ADDRESS=G...      # Public address to receive cNGN
CNGN_ISSUER_TESTNET=...         # cNGN issuer on testnet
CNGN_ISSUER_MAINNET=...         # cNGN issuer on mainnet
STELLAR_NETWORK=TESTNET         # Network selection
```

### Payment Verification Checklist

For each incoming payment to system wallet:

- ✅ **Transaction Successful**: `horizon.successful == true`
- ✅ **Destination Match**: `operation.destination == SYSTEM_WALLET_ADDRESS`
- ✅ **Asset Match**: `operation.asset_code == "cNGN"`
- ✅ **Issuer Match**: `operation.asset_issuer == CNGN_ISSUER`
- ✅ **Memo Extraction**: `tx.memo` starts with `WD-`
- ✅ **Transaction Lookup**: Find transaction in DB by memo
- ✅ **Status Check**: Transaction status is `pending_payment`
- ✅ **Amount Match**: `operation.amount == transaction.cngn_amount` ⭐ **[NEW - To Be Added]**

### Operation Structure (from Horizon)

```json
{
  "type": "payment",
  "to": "G...",                    // Destination (system wallet)
  "asset_code": "cNGN",            // Asset
  "asset_issuer": "G...",          // Issuer
  "amount": "50000.0000000",       // Amount in stroops (7 decimals)
  "from": "G...",                  // User's wallet
  "created_at": "2026-02-24T12:00:00Z"
}
```

---

## On Payment Detected

### Current Implementation (Works)

1. **Find transaction by memo**
   ```rust
   let tx_id_str = &memo[3..];  // Remove "WD-" prefix
   tx_repo.find_by_id(tx_id_str).await?
   ```

2. **Verify transaction status is pending**
   ```rust
   if db_tx.status == "pending_payment" {
       // Proceed
   }
   ```

3. **Update transaction status**
   ```rust
   tx_repo.update_status_with_metadata(
       &db_tx.transaction_id.to_string(),
       "cngn_received",
       metadata_with_hash_and_ledger,
   ).await?
   ```

4. **Save Stellar blockchain hash**
   ```rust
   tx_repo.update_blockchain_hash(
       &db_tx.transaction_id.to_string(),
       &tx.hash,
   ).await?
   ```

5. **Log webhook event**
   ```rust
   // Event type: "stellar.offramp.received"
   self.log_webhook_event(
       &db_tx.transaction_id.to_string(),
       "stellar.offramp.received",
       metadata,
   ).await;
   ```

6. **Trigger Withdrawal Processor** (via status change)
   - The offramp processor monitors for `cngn_received` status
   - When it sees this status, it processes the withdrawal
   - Sends NGN to user's bank account

### Missing: Amount Verification

**Current Gap**: The amount from the Stellar operation is NOT verified against `transaction.cngn_amount`.

**Why It Matters**:
- User could send wrong amount by mistake
- Prevents processing if amounts don't match
- Ensures accurate accounting

**Enhancement Needed**:
```rust
// In is_incoming_cngn_payment() or new method
let amount_from_op = op.get("amount")
    .and_then(|v| v.as_str())
    .and_then(|s| BigDecimal::from_str(s).ok())?;

let expected_amount = transaction.cngn_amount; // From DB

if amount_from_op != expected_amount {
    warn!("Amount mismatch: expected {}, got {}", 
          expected_amount, amount_from_op);
    // Log mismatch event, don't update status
    return Ok(false);
}
```

---

## Configuration

### Environment Variables

**Required** (in `.env` or Docker):

```env
# Stellar Configuration
STELLAR_NETWORK=TESTNET                    # TESTNET or MAINNET
HORIZON_URL=https://horizon-testnet.stellar.org  # Network endpoint

# System Wallet
SYSTEM_WALLET_ADDRESS=G...                # System wallet public key
SYSTEM_WALLET_SECRET=S...                 # System wallet secret (signing)
CNGN_ISSUER_TESTNET=G...                  # cNGN issuer testnet
CNGN_ISSUER_MAINNET=G...                  # cNGN issuer mainnet

# Transaction Monitor Configuration
TX_MONITOR_POLL_INTERVAL_SECONDS=7        # How often to check (default: 7)
TX_MONITOR_PENDING_TIMEOUT_SECONDS=600    # Absolute timeout (default: 600)
TX_MONITOR_MAX_RETRIES=5                  # Max retry attempts (default: 5)
TX_MONITOR_PENDING_BATCH_SIZE=200         # Transactions per cycle (default: 200)
TX_MONITOR_WINDOW_HOURS=24                # Monitoring lookback (default: 24)
TX_MONITOR_INCOMING_LIMIT=100             # Transactions per page (default: 100)
```

### Database Schema

**Transactions Table** (relevant fields):

```sql
CREATE TABLE transactions (
    transaction_id UUID PRIMARY KEY,
    
    -- Basic Info
    wallet_address TEXT NOT NULL,
    type VARCHAR(20) NOT NULL,  -- 'offramp', 'onramp'
    status VARCHAR(30) NOT NULL,  -- 'pending_payment', 'cngn_received', etc.
    
    -- Amounts
    cngn_amount DECIMAL(20, 7) NOT NULL,  -- Amount sent to system wallet
    ngn_amount DECIMAL(20, 7) NOT NULL,   -- Amount to be sent to bank
    
    -- Blockchain
    blockchain_hash TEXT,              -- Hex hash of confirmed tx
    incoming_hash TEXT,                -- Hash of incoming payment
    
    -- Metadata (JSONB)
    metadata JSONB DEFAULT '{}'::JSONB,  -- Stores:
                                           -- - incoming_ledger
                                           -- - incoming_confirmed_at
                                           -- - last_monitor_error
                                           -- - retry_count
    
    -- Timestamps
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_tx_status ON transactions(status);
CREATE INDEX idx_tx_memo ON transactions USING GIN(metadata);
CREATE INDEX idx_tx_created_at ON transactions(created_at);
```

### Metadata Structure (JSONB)

When payment is detected and processed:

```json
{
    "incoming_hash": "abc123def456...",
    "incoming_ledger": 12345678,
    "incoming_confirmed_at": "2026-02-24T12:00:00Z",
    "incoming_amount": "50000.0000000",
    "payment_memo": "WD-9F8E7D6C",
    "quote_id": "quote-uuid",
    "bank_code": "058",
    "account_number": "0123456789",
    "incoming_amount_matched": true,
    "matched_at": "2026-02-24T12:00:01Z"
}
```

---

## Deployment Checklist

### Prerequisites
- [ ] Stellar account created for system wallet
- [ ] cNGN asset issued or registered
- [ ] Database migrations applied
- [ ] All environment variables configured
- [ ] Redis cache initialized

### Deployment Steps

1. **Set Environment Variables**
   ```bash
   export SYSTEM_WALLET_ADDRESS="G..."
   export SYSTEM_WALLET_SECRET="S..."
   export CNGN_ISSUER_TESTNET="G..."
   export TX_MONITOR_POLL_INTERVAL_SECONDS=7
   ```

2. **Run Database Migrations**
   ```bash
   sqlx migrate run
   ```

3. **Start Services**
   ```bash
   cargo run --release
   ```

4. **Verify Startup Logs**
   ```bash
   # Look for:
   # "stellar transaction monitor worker started"
   # "has_system_wallet = true"
   ```

5. **Monitor Initial Cycles**
   ```bash
   # Watch logs for polling activity
   # "scan_incoming_transactions starting"
   # No errors should appear
   ```

---

## Testing

### Unit Tests

**Location**: `src/workers/transaction_monitor.rs` (lines 850+)

**Current Tests** (27 tests):
- Retry count extraction and increment
- Hash extraction from metadata
- Timeout detection
- Exponential backoff schedule
- Retry readiness checking
- Retryable error classification
- Metadata merging
- Error message formatting

**Run Tests**:
```bash
cargo test transaction_monitor --lib
```

### Integration Test Scenario

**Manual test flow** (15 minutes):

1. **Create a withdrawal transaction** (POST /api/offramp/initiate)
   ```bash
   curl -X POST http://localhost:3000/api/offramp/initiate \
     -H "Content-Type: application/json" \
     -d '{
       "wallet_address": "G...",
       "quote_id": "quote-123",
       "bank_code": "058",
       "account_number": "0123456789"
     }'
   ```
   Response includes:
   - `transaction_id`
   - `system_wallet_address`
   - `cngn_amount`
   - `payment_memo` (WD-...)

2. **Send cNGN payment** (from Stellar wallet)
   ```
   To: {system_wallet_address}
   Asset: cNGN
   Amount: {cngn_amount}
   Memo: {payment_memo}
   ```

3. **Wait for monitoring** (7 seconds)
   - Transaction monitor polls Horizon
   - Should find the payment
   - Should match memo to transaction
   - Should update DB status to `cngn_received`

4. **Verify database update**
   ```bash
   psql -c "SELECT transaction_id, status, metadata FROM transactions 
           WHERE status = 'cngn_received' 
           LIMIT 1;"
   ```

5. **Check webhook events**
   ```bash
   psql -c "SELECT event_type, payload FROM webhook_events 
           WHERE event_type LIKE 'stellar.offramp%' 
           ORDER BY created_at DESC LIMIT 5;"
   ```

### Observability

**Logs to Monitor**:

```bash
# Transaction monitor startup
"stellar transaction monitor worker started"
  "poll_interval_secs=7"
  "has_system_wallet=true"

# Polling cycle
"scan_incoming_transactions completed"
  "transactions_processed=N"

# Payment matched
"incoming cNGN payment matched and updated"
  "transaction_id=..."
  "incoming_hash=..."
  "status=cngn_received"

# Amount mismatch (when added)
"Amount mismatch in payment verification"
  "transaction_id=..."
  "expected_amount=50000"
  "received_amount=49500"
```

**Metrics to Track**:
- Payments matched per cycle
- Average cycle time
- Unmatched incoming payments
- Amount mismatches (once implemented)

---

## Troubleshooting

### Monitor Not Starting

**Error**: `stellar transaction monitor worker failed to start`

**Solutions**:
1. Check `SYSTEM_WALLET_ADDRESS` is set
2. Verify Stellar network configuration
3. Verify database connection
4. Check logs for specific error

### Payments Not Being Detected

**Symptoms**: 
- Payment sent to system wallet
- Status doesn't change to `cngn_received`

**Debug Steps**:
1. Check transaction exists in `/accounts/{system_wallet}/transactions` on Horizon
2. Verify memo starts with `WD-`
3. Verify asset is `cNGN` with correct issuer
4. Check if transaction lookup by memo succeeds
5. Verify transaction is in `pending_payment` status

**Query Example**:
```bash
# Check Horizon directly
curl "https://horizon-testnet.stellar.org/accounts/G.../transactions"

# Check database transaction
psql -c "SELECT * FROM transactions WHERE status = 'pending_payment' LIMIT 1;"
```

### Amount Mismatch

**When Implemented**:
- Payment received but amount doesn't match
- Check `metadata.incoming_amount` vs `cngn_amount`
- May indicate user error or exchange rate issue

---

## Next Steps

### Issue #34: Withdrawal Processor

Once `cngn_received` status is set, the offramp processor automatically:
1. Reads the transaction
2. Extracts bank details
3. Sends NGN to user's bank
4. Updates status to `completed`
5. Notifies user

### Monitoring & Alerting

Production setup should add:
- Alert when payment monitor misses cycles
- Alert when amount mismatches occur
- Dashboard showing incoming payment rates
- Webhook delivery monitoring

---

## References

**Related Issues**:
- Issue #62: POST /api/offramp/initiate (Creates transaction)
- Issue #34: Withdrawal Processor (Sends money to bank)
- Issue #12: Payment Monitoring (This document)

**Files**:
- `src/workers/transaction_monitor.rs` - Main monitoring loop
- `src/chains/stellar/client.rs` - Horizon API client
- `src/database/transaction_repository.rs` - Database operations
- `src/workers/offramp_processor.rs` - Withdrawal processing

**Stellar Documentation**:
- https://developers.stellar.org/api/introduction/
- https://developers.stellar.org/learn/fundamentals/transactions

---

**Last Updated**: February 24, 2026  
**Status**: ✅ MONITORING ACTIVE - Ready for enhancement with amount verification
