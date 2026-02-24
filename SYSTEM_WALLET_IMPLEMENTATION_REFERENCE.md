# System Wallet Information - Implementation Reference

## Quick Summary

The POST `/api/offramp/initiate` endpoint returns complete system wallet information for users to initiate cNGN withdrawals.

---

## What Gets Returned

### 1. ✅ System Stellar Wallet Address
```json
"payment_instructions": {
  "send_to_address": "GSYSTEMWALLET1234567890ABCDEFGH1234567890XYZ"
}
```
**Where it comes from**: Environment variable `SYSTEM_WALLET_ADDRESS`  
**Used for**: User sends cNGN to this address  
**Configured in**: `src/main.rs` (lines 525-534)

---

### 2. ✅ cNGN Amount to Send (Exact from Quote)
```json
"payment_instructions": {
  "send_amount": "50000.00"
}
```
**Where it comes from**: `quote.amount_cngn` from Redis cache  
**Purpose**: User must send exactly this amount  
**Format**: String with decimal places  
**Example Flow**: 
- User requests quote for 50,000 NGN ← Issue #32
- System returns quote with "50000.00 cNGN"
- User calls offramp initiate with that quote_id
- System returns same amount: "50000.00"

---

### 3. ✅ Payment Memo (Required)
```json
"payment_instructions": {
  "memo_text": "WD-9F8E7D6C",
  "memo_type": "text",
  "memo_required": true
}
```
**Format**: `WD-` followed by 8 uppercase hex characters  
**Example**: `"WD-9F8E7D6C"` (11 total characters)  
**Purpose**: System uses memo to match incoming payment to transaction  
**Critical**: Stellar transaction memo field MUST include this  
**Mentioned in**: `next_steps[3]` with "REQUIRED" emphasis

**How it's used**:
1. User sends cNGN with memo `WD-9F8E7D6C`
2. Transaction Monitor (Issue #12) detects incoming payment
3. Monitor extracts memo from Stellar transaction
4. Database query: `SELECT * FROM transactions WHERE metadata->>'payment_memo' = 'WD-9F8E7D6C'`
5. System finds matching transaction
6. Status updates to `cngn_received`

---

### 4. ✅ Minimum XLM for Transaction Fees
```json
"requirements": {
  "min_xlm_for_fees": "0.01"
}
```
**Amount**: 0.01 XLM (configurable constant)  
**Why**: Stellar network requires XLM for transaction fees  
**What user needs**: Must have ≥0.01 XLM in wallet  
**Separate from**: The cNGN being sent (different asset)  
**Example**: Send 50000 cNGN + need 0.01 XLM = 2 separate assets

---

### 5. ✅ Instructions for Sending
```json
"next_steps": [
  "Open your Stellar wallet (Freighter, Lobstr, etc.)",
  "Send exactly 50000.00 cNGN",
  "To address: GSYSTEMWALLET...",
  "Include memo: WD-9F8E7D6C (REQUIRED)",
  "Wait for confirmation",
  "NGN will be sent to your bank account (0123456789)"
]
```
**Format**: Array of clear, actionable steps  
**Count**: 6 steps covering entire flow  
**Language**: User-friendly, non-technical  

**Each step covers**:
1. Which app to use
2. Amount to send
3. Where to send it
4. Memo requirement (emphasized)
5. What to expect
6. What happens after

---

## 📋 Full Response Structure

```json
{
  "transaction_id": "550e8400-e29b-41d4-a716-446655440001",
  "status": "pending_payment",
  "created_at": "2025-01-23T10:30:45Z",
  
  "quote": {
    "cngn_amount": "50000.00",
    "ngn_amount": "49500.00",
    "total_fees": "500.00"
  },
  
  "payment_instructions": {
    "send_to_address": "GSYSTEMWALLET...",
    "send_amount": "50000.00",
    "send_asset": "cNGN",
    "asset_issuer": "GCNGN...",
    "memo_text": "WD-9F8E7D6C",
    "memo_type": "text",
    "memo_required": true
  },
  
  "requirements": {
    "min_xlm_for_fees": "0.01",
    "exact_amount_required": true,
    "memo_required": true
  },
  
  "withdrawal_details": {
    "bank_name": "Guaranty Trust Bank",
    "account_number": "0123456789",
    "account_name": "John Doe",
    "amount_to_receive": "49500.00 NGN"
  },
  
  "timeline": {
    "send_payment_by": "2025-01-23T10:35:45Z",
    "expected_confirmation": "5-10 seconds",
    "expected_withdrawal": "2-5 minutes after confirmation",
    "expires_at": "2025-01-23T10:35:45Z"
  },
  
  "next_steps": [
    "Open your Stellar wallet (Freighter, Lobstr, etc.)",
    "Send exactly 50000.00 cNGN",
    "To address: GSYSTEMWALLET...",
    "Include memo: WD-9F8E7D6C (REQUIRED)",
    "Wait for confirmation",
    "NGN will be sent to your bank account (0123456789)"
  ]
}
```

---

## 🔧 Implementation Details

### Where in Code
**File**: `src/api/offramp.rs`

**Structures**:
- Line 63-71: `PaymentInstructions` struct
- Line 109-120: `OfframpInitiateResponse` struct
- Line 145-152: `OfframpState` (contains wallet addresses)

**Response Building**:
- Lines 524-530: Payment instructions built from state
- Lines 535-537: Requirements info
- Lines 546-551: Next steps array formatted

**Constants**:
- Line 38-39: `MIN_XLM_FOR_FEES = "0.01"`
- Line 156-179: Supported banks list

### Configuration
**Environment Variables**:
```bash
SYSTEM_WALLET_ADDRESS="GSYSTEMWALLET..."
CNGN_ISSUER_ADDRESS="GCNGN..."
```

**Set in `src/main.rs`** (lines 525-534):
```rust
let system_wallet_address = env::var("SYSTEM_WALLET_ADDRESS")
    .expect("SYSTEM_WALLET_ADDRESS must be set");
let cngn_issuer_address = env::var("CNGN_ISSUER_ADDRESS")
    .expect("CNGN_ISSUER_ADDRESS must be set");
```

---

## 🔗 How It Works End-to-End

```
1. USER INITIATION
   └─> Calls POST /api/offramp/initiate
       - Provides: quote_id, wallet_address, bank_details

2. SYSTEM VALIDATION
   └─> Validates quote (exists, not expired, wallet matches)
   └─> Verifies bank account (format + provider API)

3. SYSTEM PREPARATION
   └─> Generates unique memo: WD-9F8E7D6C
   └─> Creates transaction in database
   └─> Status: pending_payment

4. SYSTEM RESPONSE
   └─> Returns all wallet information:
       ├─ System wallet address (GSYSTEMWALLET...)
       ├─ Exact cNGN amount (50000.00)
       ├─ Payment memo (WD-9F8E7D6C)
       ├─ Min XLM needed (0.01)
       └─ Clear instructions (6 steps)

5. USER ACTION
   └─> Opens Stellar wallet (Freighter, Lobstr, etc.)
   └─> Sends exactly 50000.00 cNGN
   └─> To: GSYSTEMWALLET...
   └─> With memo: WD-9F8E7D6C
   └─> Pays transaction fees in XLM (needs 0.01+)

6. STELLAR NETWORK
   └─> Confirms payment in 5-10 seconds
   └─> Broadcasts transaction

7. SYSTEM DETECTION
   └─> Transaction Monitor (Issue #12) polls wallet
   └─> Receives payment notification
   └─> Extracts memo: WD-9F8E7D6C
   └─> Queries database: SELECT ... WHERE memo = 'WD-9F8E7D6C'
   └─> Finds transaction (ID: 550e8400-...)
   └─> Updates status: pending_payment → cngn_received

8. WITHDRAWAL PROCESSING
   └─> Withdrawal Processor (Issue #34) detected cngn_received
   └─> Sends NGN to bank account in database
   └─> Status: processing_withdrawal → completed
   └─> User receives 49500 NGN in bank (after fees)
```

---

## ✅ Verification Checklist

When testing or deploying, verify:

- [ ] System wallet address set in environment variables
- [ ] cNGN issuer address set in environment variables
- [ ] Response includes `payment_instructions` with all 7 fields
- [ ] `send_to_address` matches SYSTEM_WALLET_ADDRESS
- [ ] `send_amount` matches quote cNGN amount exactly
- [ ] `memo_text` follows WD-{8_hex} format
- [ ] `memo_required` is true
- [ ] `asset_issuer` matches CNGN_ISSUER_ADDRESS
- [ ] `requirements` includes min_xlm_for_fees
- [ ] `next_steps` has all 6 user instructions
- [ ] All fields are strings (not null, not numbers without decimals)
- [ ] Timeline fields are ISO 8601 timestamps
- [ ] Response HTTP status is 200
- [ ] Error responses have appropriate status codes (400, 503, 504)

---

## 🐛 Common Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| "send_to_address is null" | SYSTEM_WALLET_ADDRESS not set | Set env var |
| "send_amount doesn't match" | Quote expired or wrong ID | Get fresh quote |
| "memo_required is false" | Code bug | Check implementation |
| "Memo format wrong" | UUID generation issue | Verify UUID parsing |
| "Payment not detected" | Wrong memo included | Verify memo matches exactly |
| "Unknown cNGN asset" | Wrong issuer address | Set CNGN_ISSUER_ADDRESS |
| "Insufficient fees" | Need more XLM | Add >0.01 XLM |

---

## 📚 Related Documents

- [OFFRAMP_QUICK_START.md](./OFFRAMP_QUICK_START.md) - Full API reference
- [MEMO_GENERATION_GUIDE.md](./MEMO_GENERATION_GUIDE.md) - Memo specification
- [MEMO_FORMAT_QUICK_REFERENCE.md](./MEMO_FORMAT_QUICK_REFERENCE.md) - Quick reference
- [OFFRAMP_TESTING_CHECKLIST.md](./OFFRAMP_TESTING_CHECKLIST.md) - Testing guide
- [ISSUE_62_COMPLETE_IMPLEMENTATION.md](./ISSUE_62_COMPLETE_IMPLEMENTATION.md) - Full implementation details

---

## 🎯 Summary

**All system wallet information is properly implemented and returned in the endpoint response.**

The POST /api/offramp/initiate endpoint provides users with everything they need to send cNGN to the system wallet for withdrawal processing, including clear instructions, exact amounts, required memo, minimum fees, and confirmation details.

✅ **Ready for deployment**
