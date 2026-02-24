# POST /api/offramp/initiate - Quick Start Guide

## 🚀 Endpoint Overview

Initiates a withdrawal by converting cNGN (Stellar) back to NGN (fiat) in user's bank account.

- **Route**: `POST /api/offramp/initiate`
- **Authentication**: Bearer token (from auth service)
- **Rate Limit**: TBD
- **Response Time**: ~2-5 seconds (includes bank verification)

## 📨 Request Format

```bash
POST /api/offramp/initiate
Content-Type: application/json
Authorization: Bearer {user_token}

{
  "quote_id": "550e8400-e29b-41d4-a716-446655440000",
  "wallet_address": "GUSER123ABCD...",
  "bank_details": {
    "bank_code": "044",
    "account_number": "0123456789",
    "account_name": "John Doe"
  }
}
```

## ✅ Request Validation

**Quote ID**: 
- UUID format
- Must exist in Redis cache
- Must not be expired (>5 minutes old)
- Must have status "pending"

**Wallet Address**:
- Stellar format (starts with 'G')
- 56 characters
- Must match quote wallet
- User must own this wallet

**Bank Code**:
- Exactly 3 digits
- Must be valid Nigerian bank code
- Supported banks: 15+ major Nigerian banks

**Account Number**:
- Exactly 10 digits
- Cannot contain special characters
- Will be verified via Flutterwave/Paystack API

**Account Name**:
- 1-200 characters
- Must match bank records (70% fuzzy match tolerance)
- Case-insensitive
- Can contain spaces

## 📤 Success Response (200 OK)

```json
{
  "transaction_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "pending_payment",
  "created_at": "2025-01-23T10:30:45Z",
  "quote": {
    "quote_id": "550e8400-e29b-41d4-a716-446655440000",
    "cngn_amount": "50000.00",
    "ngn_amount": "49500.00",
    "exchange_rate": 0.99,
    "expires_at": "2025-01-23T10:35:45Z"
  },
  "payment_instructions": {
    "send_to_address": "GSYSTEMWALLET123ABC...",
    "send_amount": "50000.00",
    "send_asset": "cNGN",
    "send_issuer": "GCNGN123ABC...",
    "memo_text": "WD-9F8E7D6C",
    "memo_type": "text",
    "memo_required": true,
    "instructions": [
      "Open your Stellar wallet",
      "Send exactly 50000.00 cNGN",
      "To address: GSYSTEMWALLET123ABC...",
      "Include memo: WD-9F8E7D6C (required)"
    ]
  },
  "withdrawal_details": {
    "destination_bank": "Guaranty Trust Bank",
    "account_number": "0123456789",
    "account_name": "John Doe",
    "withdrawal_amount": "49500.00",
    "withdrawing_currency": "NGN",
    "bank_code": "044"
  },
  "requirements": {
    "user_has_trustline": true,
    "system_wallet_funded": true,
    "bank_account_verified": true,
    "memo_required": true
  },
  "timeline": {
    "quote_expiry_minutes": 5,
    "payment_timeout_minutes": 30,
    "estimated_processing_hours": 2,
    "typical_settlement_hours": 24
  },
  "next_steps": [
    "Copy the memo: WD-9F8E7D6C",
    "Open your Stellar wallet",
    "Send exactly 50000.00 cNGN to the provided address",
    "Include the memo in your payment",
    "Wait for payment confirmation",
    "We'll process withdrawal and send NGN to your bank"
  ]
}
```

## ❌ Error Responses

### 400 Bad Request - Invalid Input

```json
{
  "error": "INVALID_BANK_CODE",
  "message": "Bank code '999' is not recognized. See supported_banks list.",
  "details": {
    "supported_banks": [
      { "code": "044", "name": "Guaranty Trust Bank" },
      { "code": "050", "name": "Zenith Bank" },
      { "code": "011", "name": "First Bank" }
    ]
  }
}
```

### 400 Bad Request - Invalid Account

```json
{
  "error": "INVALID_ACCOUNT_NUMBER",
  "message": "Account number must be exactly 10 digits",
  "details": {
    "provided": "123",
    "expected_length": 10
  }
}
```

### 400 Bad Request - Account Name Mismatch

```json
{
  "error": "ACCOUNT_NAME_MISMATCH",
  "message": "Account name did not match bank records",
  "details": {
    "provided": "JOHN DOE",
    "on_record": "JOHN ADEKUNLE DOE",
    "match_confidence": 0.65
  }
}
```

### 400 Bad Request - Quote Issue

```json
{
  "error": "QUOTE_EXPIRED",
  "message": "Quote expired 2 minutes ago. Request a new quote.",
  "details": {
    "expires_at": "2025-01-23T10:35:45Z",
    "current_time": "2025-01-23T10:38:00Z"
  }
}
```

### 400 Bad Request - Quote Already Used

```json
{
  "error": "QUOTE_ALREADY_USED",
  "message": "This quote has already been used for a withdrawal",
  "details": {
    "used_by_transaction": "550e8400-e29b-41d4-a716-446655440001",
    "used_at": "2025-01-23T10:32:00Z"
  }
}
```

### 503 Service Unavailable - Bank Verification Timeout

```json
{
  "error": "VERIFICATION_TIMEOUT",
  "message": "Bank verification API timeout. Try again in a few moments.",
  "details": {
    "timeout_seconds": 30,
    "retry_after": 5
  }
}
```

### 504 Gateway Timeout - Provider Error

```json
{
  "error": "BANK_VERIFICATION_FAILED",
  "message": "Unable to verify bank account. Please check details and retry.",
  "details": {
    "primary_provider": "Flutterwave",
    "error": "Account not found",
    "fallback_result": "Same error from Paystack"
  }
}
```

## 🧪 Testing Locally

### 1. Create a Quote First

```bash
POST /api/quotes/create
{
  "wallet_address": "GUSER123ABCD...",
  "amount_cngn": "50000.00",
  "direction": "offramp"
}
```

Response: Get `quote_id` from response

### 2. Test Bank Details (Format Validation Only)

```bash
curl -X POST http://localhost:8000/api/offramp/initiate \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer test_token" \
  -d '{
    "quote_id": "550e8400-e29b-41d4-a716-446655440000",
    "wallet_address": "GUSER123ABCD...",
    "bank_details": {
      "bank_code": "044",
      "account_number": "0123456789",
      "account_name": "Test User"
    }
  }'
```

### 3. Test with Real Bank Verification

Set environment variables:
```bash
export FLUTTERWAVE_SECRET_KEY="sk_test_..."
export PAYSTACK_SECRET_KEY="sk_test_..."
export BANK_VERIFICATION_TIMEOUT_SECS="30"
```

Then make same request as #2 - will verify with real APIs

### 4. Verify Database

```sql
-- Check transaction created
SELECT * FROM transactions 
WHERE id = '550e8400-e29b-41d4-a716-446655440000'
AND type = 'offramp';

-- Check memo stored
SELECT metadata->>'payment_memo' 
FROM transactions 
WHERE id = '550e8400-e29b-41d4-a716-446655440000';
```

## 📊 Bank Codes (Partial List)

| Code | Name |
|------|------|
| 011 | First Bank |
| 012 | Union Bank |
| 014 | Standard Chartered |
| 015 | WEMA Bank |
| 017 | Guaranty Trust Holding Company |
| 032 | Heritage Bank |
| 033 | Alat by WEMA |
| 035 | Wema Bank |
| 037 | Zenith Bank |
| 039 | Stanbic IBTC |
| 040 | Ecobank Transnational Inc |
| 044 | Access Bank |
| 045 | Citizens Bank |
| 050 | Ecobank |
| 052 | One Finance |

See full list in bank service configuration.

## 🔄 Payment Flow After Initiation

1. **T0**: Endpoint returns transaction (status: `pending_payment`, memo: `WD-9F8E7D6C`)
2. **T1-T30**: User sends cNGN from Stellar wallet
3. **T30+**: System monitor detects payment via memo
4. **T35**: Transaction status: `cngn_received`
5. **T40-T70**: Withdrawal processor sends NGN to bank
6. **T70+**: Transaction status: `processing_withdrawal`
7. **T60-300min**: Bank sends funds to user
8. **Final**: Transaction status: `completed`

## ⚡ Performance Metrics

- **Response Time**: 2-5 seconds (includes bank verification)
- **Database Insert**: <100ms
- **Bank Verification**: 1-3 seconds (2 providers tried in parallel)
- **Memo Generation**: <1ms
- **Error Handling**: 100-500ms (includes provider fallback)

## 🔐 Security Notes

- Quote validation prevents double-spending
- Bank verification prevents sending to wrong account
- Memo is unique per transaction
- All data encrypted in database
- Wallet signature required (auth service)
- Rate limiting recommended (TBD implementation)

## 🛠️ Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| QUOTE_NOT_FOUND | Quote expired or invalid ID | Get new quote |
| QUOTE_EXPIRED | >5 minutes since quote created | Get new quote |  
| WALLET_MISMATCH | Quote wallet ≠ request wallet | Use correct wallet |
| INVALID_BANK_CODE | Bank not in database | Check code from list |
| ACCOUNT_NAME_MISMATCH | Name doesn't match records | Verify with bank |
| VERIFICATION_TIMEOUT | Bank API slow | Retry in 5 seconds |

---

**See Also**:
- [MEMO_FORMAT_QUICK_REFERENCE.md](./MEMO_FORMAT_QUICK_REFERENCE.md) - Memo details
- [MEMO_GENERATION_GUIDE.md](./MEMO_GENERATION_GUIDE.md) - Complete guide
- [src/api/offramp.rs](./src/api/offramp.rs) - Implementation
- [src/services/bank_verification.rs](./src/services/bank_verification.rs) - Bank verification
