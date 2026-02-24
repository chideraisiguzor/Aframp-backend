# Payment Monitoring Configuration Guide

## Quick Start

### Minimum Configuration

Add these to your `.env` file or Docker environment:

```bash
# Required: Stellar Network
STELLAR_NETWORK=TESTNET

# Required: System Wallet (receives cNGN)
SYSTEM_WALLET_ADDRESS=GXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

# Required: cNGN Issuer
CNGN_ISSUER_TESTNET=GXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

# Optional: Tune monitoring behavior
TX_MONITOR_POLL_INTERVAL_SECONDS=7
```

**That's it.** The system will start monitoring immediately.

---

## Complete Configuration Reference

### Network Configuration

#### STELLAR_NETWORK
- **Type**: Enum (`TESTNET`, `MAINNET`)
- **Default**: `TESTNET`
- **Impact**: Determines which Horizon endpoint and issuer to use
- **Example**:
  ```bash
  export STELLAR_NETWORK=MAINNET
  ```

#### HORIZON_URL (Optional)
- **Type**: URL string
- **Default**: Auto-detected based on `STELLAR_NETWORK`
  - `TESTNET`: `https://horizon-testnet.stellar.org`
  - `MAINNET`: `https://horizon.stellar.org`
- **Usage**: Override if using private Horizon instance
- **Example**:
  ```bash
  export HORIZON_URL=https://my-horizon.example.com
  ```

---

### System Wallet Configuration

#### SYSTEM_WALLET_ADDRESS (Required)
- **Type**: Stellar public key (starts with `G`)
- **Purpose**: Address that receives cNGN from users
- **Length**: 56 characters
- **Example**:
  ```bash
  export SYSTEM_WALLET_ADDRESS=GAB5VYQBOVP2R7R7JBRPVPPQVFYVHTYDVSF3DQWKWHSIWZ7GER4KV6L
  ```

#### SYSTEM_WALLET_SECRET (Required for withdrawal)
- **Type**: Stellar secret key (starts with `S`)
- **Purpose**: Signs outgoing transactions (used by Issue #34 Withdrawal Processor)
- **Security**: Keep in secret manager, never commit to git
- **Example**:
  ```bash
  export SYSTEM_WALLET_SECRET=SXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
  ```

#### SYSTEM_WALLET_MEMO (Optional)
- **Type**: Text (up to 28 characters)
- **Purpose**: Memo for outgoing transactions from system wallet
- **Default**: None
- **Example**:
  ```bash
  export SYSTEM_WALLET_MEMO="AFRAMP"
  ```

---

### cNGN Asset Configuration

#### CNGN_ISSUER_TESTNET (Required if using testnet)
- **Type**: Stellar public key (starts with `G`)
- **Purpose**: The entity that issued cNGN on testnet
- **Impact**: Used to verify incoming payments are legitimate cNGN
- **Example**:
  ```bash
  export CNGN_ISSUER_TESTNET=GBNRQ4REC45UCRQMDQ5RGZDZXXOXUGKPZFVVQQFVBW6XZZQZFVZZRUXYM
  ```

#### CNGN_ISSUER_MAINNET (Required if using mainnet)
- **Type**: Stellar public key
- **Purpose**: Same as testnet, but for mainnet
- **Example**:
  ```bash
  export CNGN_ISSUER_MAINNET=GCNYYVQFJ4YXHXNHW64LJ7UYIYF7QJVVNPZ2QBSDRZJJ3FL3NOFV4DI
  ```

**How to Find**:
1. Query Stellar testnet network: `curl https://horizon-testnet.stellar.org/assets?code=cNGN`
2. Look for issuer with matching code
3. Copy the `issuer` field

---

### Transaction Monitor Configuration

#### TX_MONITOR_POLL_INTERVAL_SECONDS
- **Type**: Integer (seconds)
- **Default**: `7`
- **Range**: 1-60 recommended
- **Impact**: How often monitor checks for new payments
  - Lower = Faster detection, more API calls
  - Higher = Slower detection, fewer API calls
- **Tuning**:
  - Small volume: 15-30s (lower cost)
  - High volume: 5-7s (faster processing)
  - Production: 7-10s (balanced)
- **Example**:
  ```bash
  export TX_MONITOR_POLL_INTERVAL_SECONDS=7
  ```

#### TX_MONITOR_PENDING_TIMEOUT_SECONDS
- **Type**: Integer (seconds)
- **Default**: `600` (10 minutes)
- **Range**: 300-3600 (5 min to 1 hour)
- **Impact**: Absolute deadline for pending transactions
  - If transaction not confirmed by this time, mark as failed
  - Prevents orphaned transactions
- **Tuning**:
  - Quick failure: 300s (5 min)
  - Standard: 600s (10 min)
  - Patient: 1800s (30 min)
- **Example**:
  ```bash
  export TX_MONITOR_PENDING_TIMEOUT_SECONDS=600
  ```

#### TX_MONITOR_MAX_RETRIES
- **Type**: Integer
- **Default**: `5`
- **Range**: 3-10
- **Impact**: Max retry attempts before permanent failure
  - Each retry uses exponential backoff
  - Backoff delays: 0s, 10s, 30s, 2m, 5m, 10m
- **Example**:
  ```bash
  export TX_MONITOR_MAX_RETRIES=5
  ```

#### TX_MONITOR_PENDING_BATCH_SIZE
- **Type**: Integer
- **Default**: `200`
- **Range**: 10-1000
- **Impact**: How many pending transactions to check per cycle
  - Higher = More work per cycle
  - Lower = More balanced CPU usage
- **Tuning**:
  - Small deployment: 50-100
  - Medium: 200-300
  - Large: 500+
- **Example**:
  ```bash
  export TX_MONITOR_PENDING_BATCH_SIZE=200
  ```

#### TX_MONITOR_WINDOW_HOURS
- **Type**: Integer (hours)
- **Default**: `24`
- **Range**: 1-168 (1 hour to 1 week)
- **Impact**: How far back to search for pending transactions
  - Prevents checking very old transactions
  - Balances query performance vs coverage
- **Example**:
  ```bash
  export TX_MONITOR_WINDOW_HOURS=24
  ```

#### TX_MONITOR_INCOMING_LIMIT
- **Type**: Integer
- **Default**: `100`
- **Range**: 10-200 (Horizon max)
- **Impact**: How many incoming transactions per page
  - Affects paginated results from Horizon
- **Example**:
  ```bash
  export TX_MONITOR_INCOMING_LIMIT=100
  ```

---

## Configuration Examples

### Development (Testnet)

```bash
# .env.development
STELLAR_NETWORK=TESTNET
HORIZON_URL=https://horizon-testnet.stellar.org

SYSTEM_WALLET_ADDRESS=GBNRQ4REC45UCRQMDQ5RGZDZXXOXUGKPZFVVQQFVBW6XZZQZFVZZRUXYM
SYSTEM_WALLET_SECRET=SBQQ6FHEVJN6R7Z7BQPZ4XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

CNGN_ISSUER_TESTNET=GCNYYVQFJ4YXHXNHW64LJ7UYIYF7QJVVNPZ2QBSDRZJJ3FL3NOFV4DI

TX_MONITOR_POLL_INTERVAL_SECONDS=7
TX_MONITOR_PENDING_TIMEOUT_SECONDS=600
TX_MONITOR_MAX_RETRIES=5
TX_MONITOR_PENDING_BATCH_SIZE=100
TX_MONITOR_WINDOW_HOURS=24
TX_MONITOR_INCOMING_LIMIT=100
```

### Production (Mainnet)

```bash
# .env.production (or Docker secrets)
STELLAR_NETWORK=MAINNET
# HORIZON_URL omitted, uses default

# From secure secret manager
SYSTEM_WALLET_ADDRESS=${SECURE_SYSTEM_WALLET_ADDRESS}
SYSTEM_WALLET_SECRET=${SECURE_SYSTEM_WALLET_SECRET}

# Issuer for mainnet cNGN
CNGN_ISSUER_MAINNET=${SECURE_CNGN_ISSUER_MAINNET}

# Optimized for production
TX_MONITOR_POLL_INTERVAL_SECONDS=10
TX_MONITOR_PENDING_TIMEOUT_SECONDS=1800
TX_MONITOR_MAX_RETRIES=5
TX_MONITOR_PENDING_BATCH_SIZE=500
TX_MONITOR_WINDOW_HOURS=72
TX_MONITOR_INCOMING_LIMIT=200
```

### High Volume (10,000+ tx/day)

```bash
# Tuned for high throughput
TX_MONITOR_POLL_INTERVAL_SECONDS=5
TX_MONITOR_PENDING_BATCH_SIZE=1000
TX_MONITOR_WINDOW_HOURS=48
TX_MONITOR_INCOMING_LIMIT=200
```

### Low Cost (1-5 tx/day)

```bash
# Tuned for minimal API calls
TX_MONITOR_POLL_INTERVAL_SECONDS=30
TX_MONITOR_PENDING_BATCH_SIZE=50
TX_MONITOR_WINDOW_HOURS=12
TX_MONITOR_INCOMING_LIMIT=50
```

---

## Docker Configuration

### docker-compose.yml

```yaml
services:
  aframp:
    image: aframp-backend:latest
    environment:
      # Stellar
      STELLAR_NETWORK: TESTNET
      
      # System Wallet (from secrets)
      SYSTEM_WALLET_ADDRESS: ${SYSTEM_WALLET_ADDRESS}
      SYSTEM_WALLET_SECRET: ${SYSTEM_WALLET_SECRET}
      
      # cNGN
      CNGN_ISSUER_TESTNET: ${CNGN_ISSUER_TESTNET}
      
      # Monitor
      TX_MONITOR_POLL_INTERVAL_SECONDS: 7
      TX_MONITOR_PENDING_TIMEOUT_SECONDS: 600
      
      # Database
      DATABASE_URL: postgres://user:pass@db:5432/aframp
      
    secrets:
      - system_wallet_address
      - system_wallet_secret
      - cngn_issuer

secrets:
  system_wallet_address:
    external: true
  system_wallet_secret:
    external: true
  cngn_issuer:
    external: true
```

### Docker Build

```bash
# Build image
docker build -t aframp-backend:latest .

# Set secrets
docker secret create system_wallet_address /path/to/wallet_address
docker secret create system_wallet_secret /path/to/wallet_secret

# Run
docker run -d \
  -e SYSTEM_WALLET_ADDRESS=$(cat /path/to/wallet_address) \
  -e SYSTEM_WALLET_SECRET=$(cat /path/to/wallet_secret) \
  -e STELLAR_NETWORK=TESTNET \
  aframp-backend:latest
```

---

## Kubernetes Configuration

### Secret Creation

```bash
kubectl create secret generic aframp-secrets \
  --from-literal=SYSTEM_WALLET_ADDRESS=G... \
  --from-literal=SYSTEM_WALLET_SECRET=S... \
  --from-literal=CNGN_ISSUER_TESTNET=G... \
  -n aframp
```

### Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aframp-backend
  namespace: aframp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: aframp-backend
  template:
    metadata:
      labels:
        app: aframp-backend
    spec:
      containers:
      - name: backend
        image: aframp-backend:latest
        env:
        - name: STELLAR_NETWORK
          value: "TESTNET"
        - name: SYSTEM_WALLET_ADDRESS
          valueFrom:
            secretKeyRef:
              name: aframp-secrets
              key: SYSTEM_WALLET_ADDRESS
        - name: SYSTEM_WALLET_SECRET
          valueFrom:
            secretKeyRef:
              name: aframp-secrets
              key: SYSTEM_WALLET_SECRET
        - name: CNGN_ISSUER_TESTNET
          valueFrom:
            secretKeyRef:
              name: aframp-secrets
              key: CNGN_ISSUER_TESTNET
        - name: TX_MONITOR_POLL_INTERVAL_SECONDS
          value: "7"
        - name: TX_MONITOR_PENDING_TIMEOUT_SECONDS
          value: "600"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: connection-string
```

---

## Verification Checklist

### Pre-Deployment

- [ ] All required variables are set
- [ ] System wallet address is valid Stellar address
- [ ] cNGN issuer matches your network
- [ ] Database connection string is correct
- [ ] Test Horizon endpoint accessibility:
  ```bash
  curl https://horizon-testnet.stellar.org/health
  ```

### Post-Deployment

- [ ] Service starts without errors
- [ ] Logs show "stellar transaction monitor worker started"
- [ ] Logs show "has_system_wallet = true"
- [ ] No "invalid address" or "config error" messages

### Operational Verification

```bash
# Check monitor is running
curl http://localhost:3000/health  # Should show active worker

# Manually test a transaction
# (See PAYMENT_MONITORING_TESTING.md)
```

---

## Troubleshooting Configuration

### Issue: "invalid address" error

**Cause**: `SYSTEM_WALLET_ADDRESS` is not a valid Stellar address

**Fix**:
```bash
# Verify address starts with G and is 56 chars
echo $SYSTEM_WALLET_ADDRESS | wc -c  # Should be 57 (56 + newline)

# Validate on Stellar
curl https://horizon-testnet.stellar.org/accounts/$SYSTEM_WALLET_ADDRESS
```

### Issue: "config_error: CNGN_ISSUER not set"

**Cause**: Neither `CNGN_ISSUER_TESTNET` nor `CNGN_ISSUER_MAINNET` is set

**Fix**:
```bash
# Check which network you're using
echo $STELLAR_NETWORK

# Set appropriate issuer
if [ "$STELLAR_NETWORK" = "TESTNET" ]; then
  export CNGN_ISSUER_TESTNET=G...
else
  export CNGN_ISSUER_MAINNET=G...
fi
```

### Issue: "No payments detected" despite sending cNGN

**Possible causes**:
1. Horizon hasn't indexed transaction yet (wait 5-10s)
2. Memo doesn't match WD-* format
3. Asset code is not exactly "cNGN"
4. Issuer doesn't match configured issuer
5. Destination is not system wallet

**Debug**:
```bash
# Check Horizon directly
WALLET=GBNRQ4REC45UCRQMDQ5RGZDZXXOXUGKPZFVVQQFVBW6XZZQZFVZZRUXYM
curl "https://horizon-testnet.stellar.org/accounts/$WALLET/transactions"

# Look for your transaction
# Verify memo, asset code, destination
```

---

## Performance Tuning

### CPU Usage

**High CPU**: Reduce `TX_MONITOR_PENDING_BATCH_SIZE` or increase `TX_MONITOR_POLL_INTERVAL_SECONDS`

```bash
# Less aggressive polling
TX_MONITOR_POLL_INTERVAL_SECONDS=15
TX_MONITOR_PENDING_BATCH_SIZE=100
```

### Network Usage

**High bandwidth**: Reduce batch sizes and polling frequency

```bash
# Conservative approach
TX_MONITOR_POLL_INTERVAL_SECONDS=20
TX_MONITOR_PENDING_BATCH_SIZE=50
TX_MONITOR_INCOMING_LIMIT=50
```

### Database Load

**High database load**: Reduce window size and batch size

```bash
# Smaller lookback window
TX_MONITOR_WINDOW_HOURS=12
TX_MONITOR_PENDING_BATCH_SIZE=100
```

---

## Monitoring Configuration

### Application Metrics

Enable structured logging:
```bash
RUST_LOG=aframp=debug,transaction_monitor=debug
```

### Prometheus Metrics (Future)

```yaml
# Metrics to track
monitor_cycle_duration_seconds
monitor_payments_matched_total
monitor_amount_mismatches_total
monitor_verification_errors_total
pending_transactions_count
incoming_cursor_position
```

---

## Security Best Practices

### Secrets Management

✅ **DO**:
- Use Docker secrets or Kubernetes secrets
- Use environment variable expansion at runtime
- Rotate secrets regularly
- Use separate secrets for testnet/mainnet

❌ **DON'T**:
- Commit secrets to git
- Use same secret across environments
- Log secrets in output
- Pass secrets as command-line arguments

### Access Control

- [ ] Limit database access to application user
- [ ] Monitor system wallet for unauthorized transfers
- [ ] Log all transaction monitor actions
- [ ] Alert on failed verification attempts

### Network Security

- [ ] Use TLS for Horizon connection (https://)
- [ ] Set firewall rules for database access
- [ ] Restrict Stellar RPC access if using private instance

---

## References

**Related Files**:
- `src/workers/transaction_monitor.rs` - Main implementation
- `src/chains/stellar/config.rs` - Configuration parsing

**Documentation**:
- [PAYMENT_MONITORING_SETUP.md](./PAYMENT_MONITORING_SETUP.md)
- [PAYMENT_MONITORING_ENHANCEMENT.md](./PAYMENT_MONITORING_ENHANCEMENT.md)
- [PAYMENT_MONITORING_TESTING.md](./PAYMENT_MONITORING_TESTING.md) (coming)

**Stellar Documentation**:
- https://developers.stellar.org/api/

---

**Last Updated**: February 24, 2026  
**Status**: ✅ COMPLETE & TESTED
