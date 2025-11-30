# zmix Backend - Solana Privacy Mixer

Open-source backend for the zmix privacy mixer. Features zkSNARK privacy pools, multi-hop routing, and automatic fund recovery.

## Features

- **Hybrid Privacy**: zkSNARK privacy pool + multi-hop chain obfuscation
- **Fund Recovery**: Pre-transfer encrypted backups prevent fund loss
- **Non-Custodial**: Client generates keys, server stores encrypted backups
- **Fixed Tiers**: 0.1, 0.5, 1.0, 5.0 SOL denominations for anonymity sets
- **Multi-Hop Routing**: 2-6 hops with jittered delays and amount variance

## Tech Stack

- **Runtime**: Node.js 20+ with TypeScript
- **Framework**: Express.js
- **Database**: PostgreSQL (Neon serverless driver)
- **Blockchain**: Solana via @solana/web3.js
- **ZK Proofs**: snarkjs with Groth16 (BN128 curve)
- **Encryption**: AES-256-CBC for private key storage

## Project Structure

```
server/
├── index.ts              # Express server entry point
├── routes.ts             # All API endpoints
├── storage.ts            # Database operations (Drizzle ORM)
├── db.ts                 # Database connection
├── feeCalculator.ts      # 2% platform fee logic
├── lib/
│   ├── encryption.ts     # AES-256-CBC encryption/decryption
│   └── zk/
│       ├── depositPool.ts    # Privacy pool deposit/withdraw
│       ├── merkleTree.ts     # 20-level Merkle tree (Poseidon)
│       ├── snarkjsProver.ts  # Groth16 proof generation
│       └── solanaVerifier.ts # On-chain verification design
shared/
├── schema.ts             # Drizzle ORM schema (all tables)
├── schemas.ts            # Zod validation schemas
└── types.ts              # TypeScript types
circuits/
├── mixer.circom          # Main mixer circuit
└── build/                # Compiled circuit artifacts
```

## API Endpoints

### Wallet Management
- `POST /api/wallets` - Create encrypted wallet
- `GET /api/wallets` - List user wallets
- `DELETE /api/wallets/:id` - Burn wallet (with optional sweep)
- `POST /api/wallets/:id/private-key` - Fetch decrypted private key

### Mixer Sessions
- `POST /api/mixer/session/create` - Start mix session
- `POST /api/mixer/session/:id/checkpoint` - Update mix progress
- `POST /api/mixer/session/:id/complete` - Mark mix complete
- `POST /api/mixer/session/:id/fail` - Mark mix failed

### Fund Recovery (CRITICAL)
- `POST /api/mixer/recovery/save-all` - Save hop wallet keys BEFORE transfer
- `POST /api/mixer/recovery/save` - Save single recovery record
- `POST /api/mixer/recovery/execute` - Auto-sweep stuck funds
- `POST /api/mixer/recovery/sweep-all` - Admin: sweep all pending

### Privacy Pool (ZK)
- `GET /api/pool/stats` - Pool statistics and anonymity sets
- `POST /api/zk/deposit` - Create ZK deposit commitment
- `POST /api/zk/withdraw` - Generate withdrawal proof
- `GET /api/zk/tree` - Merkle tree state

## Environment Variables

```bash
# Required
DATABASE_URL=postgresql://user:pass@host:5432/dbname
WALLET_ENCRYPTION_KEY=your-32-char-secret-key
POOL_PRIVATE_KEY=base58-encoded-pool-wallet-key

# Optional
SOLANA_RPC_URL=https://api.mainnet-beta.solana.com
SESSION_SECRET=your-session-secret
NODE_ENV=production
```

## Database Schema

Key tables (see `shared/schema.ts`):
- `wallets` - Encrypted user wallets
- `mixer_sessions` - Mix session state/progress
- `hop_wallet_recovery` - Encrypted hop wallet backups
- `zk_pool_deposits` - Privacy pool deposits
- `pricing_tiers` - Fee tier configuration

## Security Features

1. **Pre-Transfer Backups**: Hop wallet keys saved to DB BEFORE any SOL moves
2. **AES-256-CBC Encryption**: All private keys encrypted at rest
3. **Anonymous Sessions**: No KYC, cookie-based session IDs
4. **Rate Limiting**: Mixer endpoints rate-limited
5. **Automatic Sweep**: Stuck funds auto-recovered on app load

## Fund Recovery Flow

```
1. Client generates hop wallets
2. Client calls /api/mixer/recovery/save-all with ALL hop wallet keys
3. Server encrypts and stores keys (24-48hr expiry)
4. Only AFTER success → client executes SOL transfers
5. If transfer fails/browser crashes → auto-sweep recovers funds
```

## Installation

```bash
# Install dependencies
npm install

# Set up database
npm run db:push

# Start development server
npm run dev
```

## Dependencies

```json
{
  "@neondatabase/serverless": "^0.10.4",
  "@solana/web3.js": "^1.98.0",
  "bcryptjs": "^2.4.3",
  "bs58": "^6.0.0",
  "crypto-js": "^4.2.0",
  "drizzle-orm": "^0.36.4",
  "express": "^4.21.1",
  "express-rate-limit": "^7.5.0",
  "express-session": "^1.18.1",
  "snarkjs": "^0.7.5",
  "circomlibjs": "^0.1.7",
  "zod": "^3.24.1"
}
```

## ZK Circuit

The mixer circuit (`circuits/mixer.circom`) implements:
- Poseidon hash for commitments
- Merkle tree membership proof (20 levels, 1M+ deposits)
- Nullifier generation for double-spend prevention

### Compiling the Circuit

```bash
# Install circom
npm install -g circom

# Compile circuit
circom circuits/mixer.circom --r1cs --wasm --sym -o circuits/build

# Trusted setup (use existing pot14_final.ptau for testing)
snarkjs groth16 setup circuits/build/mixer.r1cs pot14_final.ptau mixer_0000.zkey
snarkjs zkey contribute mixer_0000.zkey mixer_final.zkey
snarkjs zkey export verificationkey mixer_final.zkey verification_key.json
```

## License

MIT License

## Platform Fee

- 2% fee on successful mixes (collected at final hop)
- 0.5% referral rewards available

## Contributing

1. Fork the repository
2. Create feature branch (`git checkout -b feature/amazing`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing`)
5. Open Pull Request

---

**Privacy is a right, not a privilege.**

Built by the zmix team.
