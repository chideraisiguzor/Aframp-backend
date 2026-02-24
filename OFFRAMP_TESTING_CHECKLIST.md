# POST /api/offramp/initiate - Testing Checklist

## Pre-Testing Setup

- [ ] Database running with migrations applied
- [ ] Redis running and accessible
- [ ] Flutterwave test API key configured (or `SKIP_FLUTTERWAVE=true`)
- [ ] Paystack test API key configured (or `SKIP_PAYSTACK=true`)
- [ ] Test user account created with wallet
- [ ] Server running on `http://localhost:8000`

## Part 1: Request Validation

### Quote Validation

- [ ] **Valid quote**: Returns 200 with transaction
- [ ] **Missing quote**: Returns 400 with `QUOTE_NOT_FOUND`
- [ ] **Expired quote** (>5 min old): Returns 400 with `QUOTE_EXPIRED`
- [ ] **Used quote**: Returns 400 with `QUOTE_ALREADY_USED`
- [ ] **Quote with wrong status**: Returns 400 error
- [ ] **Wallet mismatch**: Quote wallet ≠ request wallet → 400 `WALLET_MISMATCH`
- [ ] **Amount mismatch**: Quote amount ≠ request expected → 400 error

### Wallet Address Validation

- [ ] **Valid Stellar address**: Accepted (56 chars, starts with 'G')
- [ ] **Invalid format**: Returns 400 `INVALID_WALLET_ADDRESS`
- [ ] **Too short**: Returns 400 error
- [ ] **Too long**: Returns 400 error
- [ ] **Invalid prefix**: Returns 400 (not 'G')
- [ ] **Non-ASCII characters**: Returns 400

### Bank Code Validation

- [ ] **Valid GTB (044)**: Accepted
- [ ] **Valid Zenith (037)**: Accepted
- [ ] **Valid First Bank (011)**: Accepted
- [ ] **Invalid code (999)**: Returns 400 `INVALID_BANK_CODE`
- [ ] **Too short (01)**: Returns 400
- [ ] **Too long (0044)**: Returns 400
- [ ] **Non-numeric**: Returns 400
- [ ] **Empty**: Returns 400

### Account Number Validation

- [ ] **Valid 10-digit**: Accepted
- [ ] **Too short (9 digits)**: Returns 400 `INVALID_ACCOUNT_NUMBER`
- [ ] **Too long (11 digits)**: Returns 400
- [ ] **Non-numeric**: Returns 400 error
- [ ] **With dashes**: Returns 400 (must be plain digits)
- [ ] **With spaces**: Returns 400
- [ ] **Empty**: Returns 400
- [ ] **Leading zeros**: Accepted (e.g., 0012345678)

### Account Name Validation

- [ ] **Valid simple name**: Accepted
- [ ] **Valid complex name** (with spaces): Accepted
- [ ] **Too long** (>200 chars): Returns 400
- [ ] **Empty**: Returns 400
- [ ] **Special chars allowed**: Accepted (spaces, hyphens)
- [ ] **Unicode characters**: Check handling

## Part 2: Bank Verification

### Format Validation Only (No Flutterwave/Paystack)

- [ ] **Format check passes**: Transaction created
- [ ] **Invalid bank code rejected**: Before API call
- [ ] **Invalid account rejected**: Before API call
- [ ] **Invalid name rejected**: Before API call

### With Bank Verification APIs Enabled

- [ ] **Valid account**: Account verified, transaction created
- [ ] **Invalid account**: Returns 400 `VERIFICATION_FAILED`
- [ ] **Account not found**: Returns 400 with provider error
- [ ] **Name doesn't match**: Returns 400 `ACCOUNT_NAME_MISMATCH`
- [ ] **API timeout** (>30s): Returns 503 `VERIFICATION_TIMEOUT`
- [ ] **Flutterwave fails, Paystack succeeds**: Auto-fallback works
- [ ] **Both fail**: Returns 503-504 with fallback info
- [ ] **Network error**: Returns appropriate error code

### Fuzzy Name Matching

- [ ] **Exact match**: Accepted (John Doe = JOHN DOE)
- [ ] **Case mismatch**: Accepted (john doe = JOHN DOE)
- [ ] **Extra spaces**: Accepted (JOHN  DOE = JOHN DOE)
- [ ] **70% match**: Accepted
- [ ] **69% match**: Rejected with `ACCOUNT_NAME_MISMATCH`
- [ ] **Middle names**: Fuzzy matching handles variations
- [ ] **Abbreviations**: Tested (J. Doe vs John Doe)

## Part 3: Response Structure

### Success Response (200)

- [ ] `transaction_id`: Valid UUID
- [ ] `status`: Equals "pending_payment"
- [ ] `created_at`: Valid ISO timestamp
- [ ] `quote` object exists with:
  - [ ] `quote_id`: Matches request
  - [ ] `cngn_amount`: Matches request
  - [ ] `ngn_amount`: Calculated correctly
  - [ ] `exchange_rate`: Valid number
  - [ ] `expires_at`: ~5 minutes from now
- [ ] `payment_instructions` object with:
  - [ ] `send_to_address`: System wallet address (40+ chars)
  - [ ] `send_amount`: Matches quote amount
  - [ ] `send_asset`: "cNGN"
  - [ ] `send_issuer`: Valid Stellar address
  - [ ] `memo_text`: Format "WD-{8_HEX_CHARS}"
  - [ ] `memo_type`: "text"
  - [ ] `memo_required`: true
  - [ ] `instructions`: Non-empty array
- [ ] `withdrawal_details` object with:
  - [ ] `destination_bank`: Bank name
  - [ ] `account_number`: Matches request
  - [ ] `account_name`: Matches request
  - [ ] `withdrawal_amount`: Correct (NGN amount after fees)
  - [ ] `withdrawing_currency`: "NGN"
  - [ ] `bank_code`: Matches request
- [ ] `requirements` object with:
  - [ ] `user_has_trustline`: true/false (appropriate)
  - [ ] `system_wallet_funded`: true/false
  - [ ] `bank_account_verified`: true/false
  - [ ] `memo_required`: true
- [ ] `timeline` object with:
  - [ ] `quote_expiry_minutes`: 5
  - [ ] `payment_timeout_minutes`: 30
  - [ ] `estimated_processing_hours`: ~2
  - [ ] `typical_settlement_hours`: ~24
- [ ] `next_steps`: Non-empty array with guidance

### Error Response Structure

- [ ] Status code appropriate (400/503/504)
- [ ] `error` field present with error code
- [ ] `message` field human-readable
- [ ] `details` object with context
- [ ] All error fields consistent format

## Part 4: Memo Generation

### Memo Format

- [ ] **Prefix**: Starts with "WD-"
- [ ] **Length**: Exactly 11 characters
- [ ] **Characters**: Uppercase hex (0-9, A-F)
- [ ] **Format matches**: "WD-[0-9A-F]{8}"
- [ ] **Stored in response**: In payment_instructions
- [ ] **Stored in database**: In transaction metadata

### Memo Uniqueness

- [ ] **Different memos**: Each transaction gets unique memo
  - [ ] Create 5 transactions
  - [ ] All have different memos
- [ ] **No collisions**: Check database for duplicate memos
- [ ] **UUID mapping**: Same memo from same transaction_id

### Memo Reproducibility

- [ ] **Same action**: Same inputs → same memo (if using same transaction_id)
- [ ] **Deterministic**: No randomness in memo (based on fixed UUID)

## Part 5: Database Integration

### Transaction Creation

- [ ] **Row inserted**: SELECT * FROM transactions where id = ?
- [ ] **Status field**: "pending_payment"
- [ ] **Type field**: "offramp"
- [ ] **Wallet matches**: Matches request wallet_address
- [ ] **Metadata stored**: JSON with memo, quote_id, bank details
- [ ] **Timestamp**: created_at is current time
- [ ] **Expiration**: expires_at is 30 minutes from now

### Metadata Storage

```sql
SELECT metadata FROM transactions WHERE id = 'xxx'::uuid
```

- [ ] **payment_memo**: "WD-{8_HEX}"
- [ ] **quote_id**: Valid UUID
- [ ] **bank_code**: 3 digits
- [ ] **account_number**: 10 digits
- [ ] **account_name**: String
- [ ] **expires_at**: ISO timestamp
- [ ] All data accessible and properly formatted

### Memo Lookup for Payment Matching

```sql
SELECT * FROM transactions 
WHERE metadata->>'payment_memo' = 'WD-9F8E7D6C'
```

- [ ] **Query works**: Returns correct transaction
- [ ] **Performance**: <100ms even with 1000+ transactions
- [ ] **Index exists**: Database has index on memo

## Part 6: Error Scenarios

### Quote-Related Errors

- [ ] **QUOTE_NOT_FOUND**: Non-existent quote_id
- [ ] **QUOTE_EXPIRED**: Quote>5 minutes old
- [ ] **QUOTE_ALREADY_USED**: Quote used twice
- [ ] **WALLET_MISMATCH**: Quote wallet ≠ request wallet
- [ ] Response includes helpful context

### Bank Validation Errors

- [ ] **INVALID_BANK_CODE**: Code not in database
- [ ] **INVALID_ACCOUNT_NUMBER**: not 10 digits
- [ ] **INVALID_ACCOUNT_NAME**: Empty or too long
- [ ] All return 400 status

### Bank Verification Errors

- [ ] **ACCOUNT_NOT_FOUND**: Account doesn't exist
- [ ] **ACCOUNT_NAME_MISMATCH**: Name doesn't match records
- [ ] **VERIFICATION_TIMEOUT**: API too slow (>30s)
- [ ] **BANK_VERIFICATION_FAILED**: Provider error
- [ ] Error messages include guidance

### Network/Provider Errors

- [ ] **Flutterwave timeout**: Falls back to Paystack
- [ ] **Paystack timeout**: Returns 503
- [ ] **Both fail**: Returns appropriate error
- [ ] **Network error**: Handled gracefully

## Part 7: State and Concurrency

### State Injection

- [ ] **Database pool**: Always available
- [ ] **Redis cache**: Connected
- [ ] **Bank verification service**: Initialized
- [ ] **Payment providers**: Configured
- [ ] **Wallet addresses**: Set from env vars

### Concurrent Requests

- [ ] **Multiple simultaneous requests**: All succeed
- [ ] **Race condition test**: Same user, different quotes
  - [ ] Each gets unique transaction_id
  - [ ] Each gets unique memo
  - [ ] Database handles correctly

## Part 8: Integration Points

### Ready for Transaction Monitor (Issue #12)

- [ ] **Memo in payment instructions**: ✓
- [ ] **Memo matches database**: ✓
- [ ] **Transaction status**: pending_payment
- [ ] **All required fields**: Present in response

### Ready for Withdrawal Processor (Issue #34)

- [ ] **Transaction created**: ✓
- [ ] **Bank details stored**: ✓
- [ ] **Status tracking field**: ✓
- [ ] **Metadata structure**: Processor can read

### Integration with Onramp Quote (Issue #32)

- [ ] **Quote lookup works**: ✓
- [ ] **Quote validation**: ✓
- [ ] **Expiry enforcement**: ✓

## Part 9: Performance & Load

### Response Time

- [ ] **Happy path**: <3 seconds
- [ ] **With bank verification**: <5 seconds
- [ ] **With API fallback**: <7 seconds
- [ ] **Database query**: <100ms
- [ ] **Memo generation**: <1ms

### Load Testing

- [ ] **100 concurrent requests**: All succeed
- [ ] **1000 requests**: No errors
- [ ] **Database connection pool**: Handles load
- [ ] **Redis**: No timeouts
- [ ] **Memory usage**: Stable

## Part 10: Documentation Compliance

- [ ] **Memo format documented**: MEMO_FORMAT_QUICK_REFERENCE.md
- [ ] **Endpoint documented**: OFFRAMP_QUICK_START.md
- [ ] **Code comments**: Clear and helpful
- [ ] **Error codes documented**: All mapped
- [ ] **Integration points**: Clear for next issues

## Sign-Off

- [ ] All core tests passed
- [ ] All error scenarios tested
- [ ] Database integration verified
- [ ] Performance acceptable
- [ ] Documentation complete
- [ ] Ready for development deployment

**Tested By**: _______________  
**Test Date**: _______________  
**Status**: ⬜ Not Started | 🟨 In Progress | 🟩 Passed | 🔴 Failed

---

## Quick Test Commands

### Basic Success Test
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
      "account_name": "John Doe"
    }
  }'
```

### Test Invalid Bank Code
```bash
# Change bank_code to "999"
# Expected: 400 INVALID_BANK_CODE
```

### Test Account Name Mismatch
```bash
# Use name that doesn't match bank records
# Expected: 400 ACCOUNT_NAME_MISMATCH
```

### Test Expired Quote
```bash
# Use quote from >5 minutes ago
# Expected: 400 QUOTE_EXPIRED
```

---

See [OFFRAMP_QUICK_START.md](./OFFRAMP_QUICK_START.md) for detailed test examples.
