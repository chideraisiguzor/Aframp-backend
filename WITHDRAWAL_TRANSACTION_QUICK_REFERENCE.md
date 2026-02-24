# Withdrawal Transaction - Quick Reference Guide

## 🎯 What Gets Stored

When a user initiates a withdrawal, this transaction record is created in the database:

---

## 📊 Transaction Record Structure

```
TRANSACTION CREATED
├─ ID (UUID)
│  └─ 550e8400-e29b-41d4-a716-446655440001
│
├─ User Wallet
│  └─ GUSER123ABCD... (Stellar address)
│
├─ Quote Reference
│  └─ 550e8400-e29b-41d4-a716-446655440000
│
├─ Amounts
│  ├─ cNGN to receive: 50000.00
│  ├─ NGN to send:     49500.00
│  ├─ Exchange rate:   0.99
│  └─ Fees:            500.00
│
├─ Bank Details
│  ├─ Code:           044 (GTBank)
│  ├─ Account:        0123456789
│  ├─ Name:           John Doe
│  └─ Bank:           Guaranty Trust Bank
│
├─ Payment Identifier
│  └─ Memo: WD-9F8E7D6C (unique)
│
├─ Timeline
│  ├─ Created:  2025-01-23 10:30:45 UTC
│  └─ Expires:  2025-01-23 10:35:45 UTC (+30 min)
│
└─ Status
   └─ pending_payment (awaiting cNGN)
```

---

## 🔄 Status Evolution

```
INITIAL STATE                 PROCESSING                      TERMINAL STATES
     ↓                             ↓                               ↓
pending_payment           verifying_amount         ┌─ completed ✅
     ↓                             ↓                │
cngn_received          processing_withdrawal      ├─ refunded ↩️
     ↓                             ↓                │
     └─────────────→ transfer_pending             ├─ failed ❌
                             ↓                    │
                        completed ✅ ─────────────┘
                             
TIMEOUT PATH                OTHER PATHS
     ↓                           ↓
pending_payment              Any state
(30 min pass)                     ↓
     ↓                     RefundInitiated
   expired ❌                      ↓
                             Refunding
                                  ↓
                              Refunded ↩️
```

---

## 📋 All Stored Data

### In Main Columns
```
transaction_id    : 550e8400-e29b-41d4-a716-446655440001
wallet_address    : GUSER123ABCD...
type              : offramp
from_currency     : cNGN
to_currency       : NGN
from_amount       : 50000.00000000
to_amount         : 49500.00
cngn_amount       : 50000.00000000
status            : pending_payment
payment_provider  : NULL (set later)
payment_reference : WD-9F8E7D6C
blockchain_tx_hash: NULL (set when payment received)
error_message     : NULL
created_at        : 2025-01-23T10:30:45Z
updated_at        : 2025-01-23T10:30:45Z
```

### In Metadata (JSON)
```json
{
  "quote_id": "550e8400-e29b-41d4-a716-446655440000",
  "payment_memo": "WD-9F8E7D6C",
  "bank_code": "044",
  "account_number": "0123456789",
  "account_name": "John Doe",
  "bank_name": "Guaranty Trust Bank",
  "withdrawal_type": "offramp",
  "expires_at": "2025-01-23T10:35:45Z"
}
```

---

## 🎯 Key Identifiers

| Field | Value | Purpose |
|-------|-------|---------|
| transaction_id | UUID | Unique transaction ID |
| quote_id (in metadata) | UUID | Links to quote |
| payment_memo | WD-9F8E7D6C | Payment matching key |
| payment_reference | WD-9F8E7D6C | Alternative memo field |
| blockchain_tx_hash | NULL initially | Set when Stellar payment detected |

---

## 💰 Amount Mapping

```
From Quote Service
        ↓
├─ ngn_requested: 50000.00 NGN
├─ exchange_rate: 0.99
├─ fees: 500.00 NGN
└─ total_cost: 50506.06 NGN (in customer perspective)

To cNGN Conversion
        ↓
        cngn_amount = 50000.00 / 0.99
                    = 50505.05 cNGN
(Amount user must send)

Stored in Transaction
        ↓
├─ from_amount: 50505.05 cNGN (what we receive)
├─ to_amount: 50000.00 NGN (what bank sends)
├─ cngn_amount: 50505.05 (duplicate for queries)
└─ metadata.total_fees: 505.05 NGN
```

---

## ⏰ Timeline

```
T0: create_withdrawal_transaction()
    └─ Status: pending_payment
    └─ Expires: T+30 minutes

T1-T29: User has time to send cNGN
        │
        └─ User opens Stellar wallet
        └─ Sends 50505.05 cNGN
        └─ Includes memo: WD-9F8E7D6C
        └─ Stellar confirms in 5-10 seconds

T10-T30: Transaction Monitor sees payment
         │
         └─ Queries DB by memo
         └─ Finds matching transaction
         └─ Updates status: cngn_received
         └─ Sets blockchain_tx_hash

T31-T40: Withdrawal Processor processes
         │
         └─ Withdrawal Processor wakes up
         └─ Sees cngn_received status
         └─ Initiates NGN transfer to bank
         └─ Updates status: processing_withdrawal

T41-T300: Bank processes (24-48 hours)
          └─ Updates status as it progresses
          └─ Final: completed

T30+ (if no payment): Status → expired
```

---

## 🔍 Database Queries

### Create Transaction
```sql
INSERT INTO transactions (
    wallet_address, type, from_currency, to_currency,
    from_amount, to_amount, cngn_amount, status,
    payment_reference, metadata
) VALUES (
    'GUSER...', 'offramp', 'cNGN', 'NGN',
    50505.05, 50000.00, 50505.05, 'pending_payment',
    'WD-9F8E7D6C', '{"quote_id":"...", "payment_memo":"WD-9F8E7D6C",...}'
)
RETURNING *;
```

### Find by Memo (Payment Monitor)
```sql
SELECT * FROM transactions
WHERE metadata->>'payment_memo' = 'WD-9F8E7D6C'
  AND status = 'pending_payment';
```

### Find by Wallet (User History)
```sql
SELECT * FROM transactions
WHERE wallet_address = 'GUSER...'
  AND type = 'offramp'
ORDER BY created_at DESC;
```

### Update Status
```sql
UPDATE transactions
SET status = 'cngn_received',
    blockchain_tx_hash = 'STELLAR_TX_HASH...',
    updated_at = NOW()
WHERE transaction_id = '550e8400-e29b-41d4-a716-446655440001'::uuid;
```

---

## ✅ What's Included

**Essential Info** ✅
- ✅ User's wallet address
- ✅ cNGN amount to send
- ✅ NGN amount to receive
- ✅ Exchange rate used
- ✅ Bank account details
- ✅ Payment memo (WD-9F8E7D6C)

**Timeline** ✅
- ✅ Creation timestamp
- ✅ Expiration (30 minutes)
- ✅ Payment received time (when updated)

**Links & References** ✅
- ✅ Quote ID (audit trail)
- ✅ Transaction ID (tracking)
- ✅ Payment memo (matching)

**Processing** ✅
- ✅ Current status
- ✅ Error messages (if failed)
- ✅ Stellar tx hash (when confirmed)
- ✅ Bank reference (when sent)

---

## 🧪 Verification

### Check Transaction Created
```bash
# In your application after calling offramp/initiate
SELECT * FROM transactions 
WHERE transaction_id = '{transaction_id_from_response}'::uuid;
```

### Verify All Fields
```bash
# Check main fields
- status should be: 'pending_payment'
- type should be: 'offramp'
- from_currency should be: 'cNGN'
- to_currency should be: 'NGN'
- expires_at should be ~30 minutes in future

# Check metadata JSON
- payment_memo: 'WD-XXXXXXXX'
- quote_id: matches your quote
- bank_code: '044'
- account_number: matches request
- account_name: matches request
```

---

## 🔗 Integration Timeline

```
POST /api/offramp/initiate
         ↓ creates
[Transaction: pending_payment]
         ↓ (user sends cNGN with memo)
Issue #12: Transaction Monitor
         ↓ (detects payment, updates status)
[Transaction: cngn_received]
         ↓
Issue #34: Withdrawal Processor
         ↓ (sends NGN to bank)
[Transaction: processing_withdrawal → completed]
         ↓
User receives NGN in bank account ✅
```

---

## 📝 Code Files

| File | Purpose |
|------|---------|
| src/api/offramp.rs | Creates transaction (line 387-435) |
| src/api/offramp_models.rs | Models and types (450+ lines) |
| src/database/transaction_repository.rs | Database operations |
| WITHDRAWAL_TRANSACTION_CREATION.md | Full guide with examples |
| SECTION_5_WITHDRAWAL_TRANSACTION.md | Complete reference |

---

## ✅ Status: COMPLETE

All transaction data is created with:
- ✅ Unique identifier
- ✅ Link to quote
- ✅ All amounts and fees
- ✅ Bank details
- ✅ Payment memo
- ✅ Proper status tracking
- ✅ 30-minute expiration
- ✅ Complete state machine

**Ready for Issue #12 & #34 integration**
