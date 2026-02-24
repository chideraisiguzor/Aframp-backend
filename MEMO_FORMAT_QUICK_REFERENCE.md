# Memo Format Quick Reference

## 📋 Quick Facts

- **Format**: `WD-{8_hex_chars}`
- **Example**: `WD-9F8E7D6C`
- **Length**: 11 characters
- **Character Set**: Uppercase hex (0-9, A-F)
- **Uniqueness**: Based on UUID (2^128 possibilities)
- **Stellar Limit**: 28 bytes (✅ well under limit at 11 bytes)

## 🔄 Complete Flow

```
User Initiates Withdrawal
         ↓
Generate unique memo (WD-9F8E7D6C)
         ↓
Create transaction in DB
         ↓
Return payment instructions with memo
         ↓
User opens Stellar wallet
         ↓
User sends cNGN to system wallet WITH MEMO
         ↓
System detects payment via transaction monitor
         ↓
System matches payment to transaction using memo
         ↓
System processes withdrawal to bank
         ↓
User receives NGN in bank account
```

## 📝 API Response Structure

```json
{
  "transaction_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "pending_payment",
  "payment_instructions": {
    "send_to_address": "GSYSTEMWALLET...",
    "send_amount": "50000.00",
    "send_asset": "cNGN",
    "memo_text": "WD-9F8E7D6C",
    "memo_type": "text",
    "memo_required": true
  },
  "next_steps": [
    "Open your Stellar wallet",
    "Send exactly 50000.00 cNGN",
    "To address: GSYSTEMWALLET...",
    "Include memo: WD-9F8E7D6C (REQUIRED)",
    "..."
  ]
}
```

## 🔍 Memo Matching Logic

**Database Query:**
```sql
SELECT * FROM transactions
WHERE metadata->>'payment_memo' = 'WD-9F8E7D6C'
  AND status = 'pending_payment'
  AND type = 'offramp'
```

**When Payment is Detected:**
1. Extract memo from incoming payment: `WD-9F8E7D6C`
2. Query database for matching memo
3. Verify transaction is in correct status
4. Verify payment amount
5. Update transaction to `cngn_received`
6. Trigger withdrawal processor
7. Send NGN to user's bank

## ⚠️ Critical Notes

- **Memo is REQUIRED** - Payment without memo will fail
- **Memo must match exactly** - Typos will cause payment failure
- **Each transaction gets unique memo** - Never reuse memos
- **Memo is generated automatically** - Users don't create it
- **Memo expires after 30 minutes** - New quote needed if not sent
- **Uppercase only** - Normalize all memos to uppercase

## 🛠️ Testing

**Generate memo:**
```rust
let uuid = Uuid::new_v4();
let memo = generate_withdrawal_memo(&uuid);
// Returns: "WD-9F8E7D6C" format
```

**Validation:**
```
✅ Starts with "WD-"
✅ Exactly 11 characters
✅ Next 8 chars are uppercase hex
✅ ASCII only
✅ Under 28 byte Stellar limit
```

## 📊 Memo Lifecycle States

| State | Status | Memo Active | User Action |
|-------|--------|-------------|------------|
| Created | pending_payment | ✅ Yes | User must send payment |
| Received | cngn_received | ✅ Yes | System processing |
| Processing | processing_withdrawal | ✅ Yes | System transferring NGN |
| Completed | completed | ❌ No | Done |
| Expired | expired | ❌ No | Quote expired, ask for new |
| Failed | failed | ❌ No | Retry or contact support |

## 🔐 Security

- **No Sensitive Data**: Memo doesn't contain user/account info
- **Collision-Free**: UUID-based (< 1 in 10^36 chance)
- **Deterministic**: Same UUID always generates same memo
- **Immutable**: Once stored, memo cannot be changed
- **Auditable**: Complete trail from generation to usage

## 📞 Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| "Memo not included" | User forgot memo | Resend payment with memo |
| "Invalid memo" | Wrong memo copied | Use exact memo from response |
| "Memo mismatch" | Typo in memo | Copy-paste memo, don't type |
| "Pending payment" after 30 min | Expired quote | Request new quote with new memo |
| Payment disappeared | Different system wallet | Verify system wallet address |

## 🌐 Wallets Tested

✅ Freighter (Chrome extension)
✅ Lobstr (web)
✅ Solar (web)
✅ Stellar Expert (web)
✅ LedgerLive (hardware)

All support `text` memo type with our format.

## 📈 Metrics to Monitor

```
- Memos generated per hour
- Payment success rate by memo matching
- Memo format violations
- Memo collision occurrences (should be 0)
- Average memo lifespan to completion
- Orphaned payments (memo not found)
```

---

**For complete documentation, see**: [MEMO_GENERATION_GUIDE.md](./MEMO_GENERATION_GUIDE.md)
