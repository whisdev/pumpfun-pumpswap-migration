# Pump.fun to Pump.amm Migration Tool

A specialized tool for migrating tokens from pump.fun bonding curves to pump.amm AMM pools on Solana. This project provides a streamlined migration function that handles the complex process of moving tokens from the bonding curve model to the AMM model.

## 🚀 Features

- **Atomic Migration**: Complete migration in a single transaction
- **Jito Bundle Support**: Handles large transactions through Jito bundles
- **Slippage Protection**: Built-in slippage tolerance management
- **Automatic Pool Detection**: Detects and validates pool existence
- **Transaction Size Management**: Handles oversized transactions automatically

## ⚙️ Environment Configuration

Create a `.env` file with the following variables:

```env
# Solana RPC Configuration
RPC_ENDPOINT=https://api.mainnet-beta.solana.com
RPC_WEBSOCKET_ENDPOINT=wss://api.mainnet-beta.solana.com
GEYSER_RPC=your_geyser_rpc_endpoint

# Jito Configuration (for bundle transactions)
JITO_KEY=your_jito_key
JITO_TIP=10000
BLOCKENGINE_URL=https://amsterdam.mainnet.block-engine.jito.wtf

# Wallet Configuration
RANDOM_NUM=your_random_number
MASTER_SECRET=your_encrypted_private_key
MASTER_PUBLIC_KEY=your_public_key

# WebSocket Configuration
SOCKET_URL=your_socket_url
```

### Environment Variables Explained

| Variable | Description | Required | Example |
|----------|-------------|----------|---------|
| `RPC_ENDPOINT` | Solana RPC endpoint | Yes | `https://api.mainnet-beta.solana.com` |
| `RPC_WEBSOCKET_ENDPOINT` | WebSocket endpoint for real-time updates | Yes | `wss://api.mainnet-beta.solana.com` |
| `GEYSER_RPC` | Geyser RPC for account subscriptions | No | `your_geyser_endpoint` |
| `JITO_KEY` | Jito API key for bundle transactions | Yes | `your_jito_api_key` |
| `JITO_TIP` | Tip amount for Jito bundles (lamports) | Yes | `10000` |
| `BLOCKENGINE_URL` | Jito block engine URL | Yes | `https://amsterdam.mainnet.block-engine.jito.wtf` |
| `RANDOM_NUM` | Random number for encryption | Yes | `your_random_string` |
| `MASTER_SECRET` | Encrypted private key | Yes | `encrypted_base58_key` |
| `MASTER_PUBLIC_KEY` | Public key of the wallet | Yes | `your_public_key` |
| `SOCKET_URL` | WebSocket URL for notifications | No | `your_socket_url` |

## 🔧 Usage

### Basic Migration

```typescript
import { migrateToken } from './src/migration';

const result = await migrateToken({
  tokenMint: "CxtvkkDHCUCVKH7qY8gYwJAj74yKq7xZSb1dxWHopump",
  buyAmount: 1000, // tokens to buy after migration
  slippage: 30, // 30% slippage tolerance
  maxMigrationFee: 20 // maximum SOL fee for migration
});

console.log(`Migration successful: ${result.signature}`);
```

### Advanced Migration with Custom Parameters

```typescript
import { migrateTokenWithCustomParams } from './src/migration';

const result = await migrateTokenWithCustomParams({
  tokenMint: "CxtvkkDHCUCVKH7qY8gYwJAj74yKq7xZSb1dxWHopump",
  buyAmount: 1000,
  slippage: 30,
  maxMigrationFee: 20,
  useJitoBundle: true,
  computeUnitPrice: 5000,
  tipLamports: 10000
});
```

## 📊 Input Values and Parameters

### Required Parameters

| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `tokenMint` | `string` | Token mint address | `"CxtvkkDHCUCVKH7qY8gYwJAj74yKq7xZSb1dxWHopump"` |
| `buyAmount` | `number` | Amount of tokens to buy after migration | `1000` |
| `slippage` | `number` | Slippage tolerance in basis points | `30` (30%) |
| `maxMigrationFee` | `number` | Maximum SOL fee for migration | `20` (20 SOL) |

### Optional Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `useJitoBundle` | `boolean` | `true` | Use Jito bundle for large transactions |
| `computeUnitPrice` | `number` | `5000` | Compute unit price in micro-lamports |
| `tipLamports` | `number` | `10000` | Tip amount for Jito bundles |
| `simulateOnly` | `boolean` | `false` | Simulate transaction without execution |

## 🔄 Migration Workflow

### 1. Pre-Migration Checks

```typescript
// Check if token exists in pump.fun
const pumpfunAdapter = await PumpfunAdapter.create(connection, tokenMint);
const reserves = await pumpfunAdapter.getPoolReserves();

// Validate migration conditions
if (reserves.reserveToken1 < buyAmount) {
  throw new Error("Insufficient tokens in pump.fun pool");
}
```

### 2. Calculate Migration Costs

```typescript
// Calculate optimal migration fee
const maxQuoteWithSlippage = pumpfunAdapter.getSwapQuote(
  reserves.reserveToken1, 
  NATIVE_MINT.toBase58(), 
  slippage / 100
);

// Check if migration fee is acceptable
if (maxQuoteWithSlippage.maxQuote / LAMPORTS_PER_SOL > maxMigrationFee) {
  throw new Error("Migration fee too high");
}
```

### 3. Prepare Transaction

```typescript
// Create pump.fun swap instruction
const pumpfunSwapIx = pumpfunAdapter.getSwapInstruction(
  maxQuoteWithSlippage.maxQuote,
  reserves.reserveToken1,
  { inputMint: NATIVE_MINT, payer: keypair.publicKey }
);

// Create migration instruction
const migrationIx = await pumpfunAdapter.getMigrationInstruction(keypair.publicKey);

// Create pump.amm buy instruction
const { tx: migrationBuyIx } = await migration_buyIx(
  tokenMint, 
  poolKeys.creator, 
  buyAmount, 
  slippage
);
```

### 4. Execute Transaction

```typescript
// For large transactions, use Jito bundle
if (transactionSize > MAX_TRANSACTION_SIZE) {
  const versionedTx = new VersionedTransaction(messageV0);
  versionedTx.sign([keypair]);
  const signature = await executeJitoTx([versionedTx], keypair, "confirmed");
} else {
  // Standard transaction
  const signature = await sendAndConfirmTransaction(connection, tx, [keypair]);
}
```

## 📏 Transaction Size Management

### Transaction Size Limits

- **Standard Transaction**: ~1,232 bytes
- **Jito Bundle**: Up to 64KB
- **Versioned Transaction**: Required for large transactions

### Size Optimization Strategies

1. **Instruction Batching**: Combine multiple instructions in single transaction
2. **Account Reuse**: Minimize account creation instructions
3. **Jito Bundles**: Use for transactions exceeding standard limits
4. **Versioned Transactions**: Enable larger transaction sizes

### Size Calculation

```typescript
const calculateTransactionSize = (instructions: TransactionInstruction[]): number => {
  let size = 0;
  
  // Header size
  size += 3; // num_required_signatures, num_readonly_signed_accounts, num_readonly_unsigned_accounts
  
  // Account keys
  const accountKeys = new Set<PublicKey>();
  instructions.forEach(ix => {
    ix.keys.forEach(key => accountKeys.add(key.pubkey));
  });
  size += accountKeys.size * 32; // 32 bytes per public key
  
  // Instructions
  instructions.forEach(ix => {
    size += 1; // program_id_index
    size += 1; // num_accounts
    size += ix.keys.length; // account indices
    size += 1; // data length
    size += ix.data.length; // instruction data
  });
  
  return size;
};
```

## ✅ Transaction Confirmation

### Confirmation Methods

1. **Standard Confirmation**
```typescript
const signature = await sendAndConfirmTransaction(connection, tx, [keypair], {
  commitment: "confirmed",
  maxRetries: 3
});
```

2. **Jito Bundle Confirmation**
```typescript
const signature = await executeJitoTx([versionedTx], keypair, "confirmed");
```

3. **Manual Confirmation Check**
```typescript
const confirmation = await connection.confirmTransaction(signature, "confirmed");
if (confirmation.value.err) {
  throw new Error(`Transaction failed: ${confirmation.value.err}`);
}
```

### Confirmation Status Levels

| Level | Description | Use Case |
|-------|-------------|----------|
| `processed` | Transaction received by cluster | Quick feedback |
| `confirmed` | Transaction confirmed by supermajority | Standard confirmation |
| `finalized` | Transaction finalized | Maximum security |

### Error Handling

```typescript
try {
  const result = await migrateToken(params);
  console.log(`Migration successful: ${result.signature}`);
} catch (error) {
  if (error.message.includes("Insufficient funds")) {
    console.error("Wallet balance too low");
  } else if (error.message.includes("Slippage tolerance exceeded")) {
    console.error("Price moved too much");
  } else if (error.message.includes("Transaction too large")) {
    console.error("Transaction exceeds size limit");
  } else {
    console.error(`Migration failed: ${error.message}`);
  }
}
```

## 🔍 Monitoring and Verification

### Transaction Verification

```typescript
// Verify transaction on Solscan
const solscanUrl = `https://solscan.io/tx/${signature}`;
console.log(`View transaction: ${solscanUrl}`);

// Check transaction status
const status = await connection.getSignatureStatus(signature);
console.log(`Transaction status: ${status.value?.confirmationStatus}`);
```

### Pool State Verification

```typescript
// Verify pump.fun pool state
const pumpfunReserves = await pumpfunAdapter.getPoolReserves();
console.log("Pump.fun reserves:", pumpfunReserves);

// Verify pump.amm pool state
const pumpammAdapter = await PumpSwapAdapter.create(connection, poolId);
const pumpammReserves = await pumpammAdapter.getPoolReserves();
console.log("Pump.amm reserves:", pumpammReserves);
```

## 📈 Performance Optimization

### Gas Optimization

```typescript
// Set compute unit limit
const modifyComputeUnits = ComputeBudgetProgram.setComputeUnitLimit({
  units: 200_000
});

// Set priority fee
const addPriorityFee = ComputeBudgetProgram.setComputeUnitPrice({
  microLamports: 100_000
});
```

### Slippage Management

```typescript
// Dynamic slippage based on market conditions
const getOptimalSlippage = (marketVolatility: number): number => {
  if (marketVolatility > 0.5) return 50; // 50% for high volatility
  if (marketVolatility > 0.2) return 30; // 30% for medium volatility
  return 20; // 20% for low volatility
};
```

## 🛡️ Security Considerations

### Private Key Management

- Use encrypted private keys in environment variables
- Never hardcode private keys in source code
- Use hardware wallets for production environments

### Transaction Validation

```typescript
// Validate transaction before sending
const simulation = await connection.simulateTransaction(tx);
if (simulation.value.err) {
  throw new Error(`Transaction simulation failed: ${simulation.value.err}`);
}
```

### Error Recovery

```typescript
// Implement retry logic for failed transactions
const retryTransaction = async (tx: Transaction, maxRetries: number = 3) => {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await sendAndConfirmTransaction(connection, tx, [keypair]);
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)));
    }
  }
};
```

## 📚 API Reference

### Core Functions

#### `migrateToken(params: MigrationParams): Promise<MigrationResult>`

Performs a complete migration from pump.fun to pump.amm.

**Parameters:**
- `params.tokenMint`: Token mint address
- `params.buyAmount`: Amount of tokens to buy
- `params.slippage`: Slippage tolerance
- `params.maxMigrationFee`: Maximum migration fee

**Returns:**
- `signature`: Transaction signature
- `executedPrice`: Actual execution price
- `ammPoolAddress`: AMM pool address
- `migrationFee`: Actual migration fee paid

#### `migrateTokenWithCustomParams(params: CustomMigrationParams): Promise<MigrationResult>`

Advanced migration with custom parameters.

**Additional Parameters:**
- `params.useJitoBundle`: Use Jito bundle
- `params.computeUnitPrice`: Compute unit price
- `params.tipLamports`: Tip amount
- `params.simulateOnly`: Simulation mode

### Utility Functions

#### `calculateMigrationCost(tokenMint: string): Promise<MigrationCost>`

Calculates the cost of migrating a specific token.

#### `validateMigrationConditions(tokenMint: string): Promise<ValidationResult>`

Validates if migration is possible for a token.

#### `getPoolInfo(tokenMint: string): Promise<PoolInfo>`

Retrieves pool information for a token.

## 🧪 Testing

### Test Configuration

```typescript
// Test configuration
const testConfig = {
  testTokenMint: "CxtvkkDHCUCVKH7qY8gYwJAj74yKq7xZSb1dxWHopump",
  testBuyAmount: 1000,
  testSlippage: 30,
  testMaxFee: 20,
  simulateOnly: true
};
```

## 🚨 Troubleshooting

### Common Issues

1. **Insufficient Funds**
   - Check wallet balance
   - Verify migration fee calculation
   - Ensure sufficient SOL for gas fees

2. **Transaction Too Large**
   - Use Jito bundles for large transactions
   - Optimize instruction count
   - Use versioned transactions

3. **Slippage Tolerance Exceeded**
   - Increase slippage tolerance
   - Check market volatility
   - Retry with higher slippage

4. **Pool Not Found**
   - Verify token mint address
   - Check if pool exists on pump.fun
   - Ensure pool is not completed

### Debug Mode

```typescript
// Enable debug logging
const DEBUG = process.env.DEBUG === 'true';

if (DEBUG) {
  console.log('Transaction simulation:', await connection.simulateTransaction(tx));
  console.log('Pool reserves:', await pumpfunAdapter.getPoolReserves());
  console.log('Migration cost:', migrationCost);
}
```

## 📞 Support

For support and questions:
- Create an issue on GitHub
- Core function is sit on private repository under sign. Contact on telegram: https://t.me/whisdev
