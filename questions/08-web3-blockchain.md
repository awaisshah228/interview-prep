# Web3 & Blockchain - Interview Q&A

> 20+ questions covering Solana, Ethereum, Anchor, DeFi, wallet integration, and indexing

---

## Table of Contents

- [Blockchain Fundamentals](#blockchain-fundamentals)
- [Solana Architecture](#solana-architecture)
- [Anchor Framework](#anchor-framework)
- [Ethereum & EVM](#ethereum--evm)
- [DeFi Concepts](#defi-concepts)
- [Frontend Integration](#frontend-integration)
- [Indexing & Data](#indexing--data)

---

## Blockchain Fundamentals

### Q1: Explain the difference between Solana and Ethereum.

**Answer:**

| Feature | Solana | Ethereum |
|---|---|---|
| Consensus | Proof of History + PoS | Proof of Stake |
| TPS | ~65,000 theoretical, ~4,000 actual | ~15-30 (L1), ~4,000 (L2) |
| Block time | ~400ms | ~12 seconds |
| Transaction cost | ~$0.00025 | ~$1-50 (varies) |
| Smart contracts | Rust (Anchor), C | Solidity, Vyper |
| Account model | Account-based (stateless programs) | Account-based (stateful contracts) |
| Parallel execution | Yes (Sealevel) | No (sequential) |
| Ecosystem maturity | Growing | Mature |
| State storage | Separate accounts | Contract storage |

**Solana's unique features:**
- **Proof of History**: Cryptographic clock, verifiable passage of time
- **Sealevel**: Parallel smart contract runtime
- **Gulf Stream**: Mempool-less transaction forwarding
- **Turbine**: Block propagation protocol
- **Pipelining**: Transaction processing pipeline

---

### Q2: What is a consensus mechanism and why does it matter?

**Answer:**

```
Consensus = How nodes agree on the state of the blockchain

Proof of Work (PoW) - Bitcoin:
├─ Miners solve computational puzzles
├─ First to solve gets to add block + reward
├─ Very secure, very energy-intensive
└─ Slow (10 min blocks for Bitcoin)

Proof of Stake (PoS) - Ethereum:
├─ Validators stake ETH as collateral
├─ Selected to propose blocks based on stake
├─ Slashing penalty for malicious behavior
├─ Energy efficient
└─ Faster than PoW

Proof of History (PoH) - Solana:
├─ Cryptographic clock using SHA-256 hashing
├─ Creates verifiable passage of time
├─ Validators don't need to agree on time
├─ Enables parallel processing
└─ Sub-second block times

Why it matters for developers:
- Determines transaction speed (UX impact)
- Determines cost (gas/fees)
- Determines finality (when is a tx "confirmed"?)
- Determines programming model
```

---

## Solana Architecture

### Q3: Explain Solana's Account Model.

**Answer:**

```
Everything on Solana is an ACCOUNT:
┌─────────────────────────────────────┐
│ Account                              │
│ ├─ lamports: u64 (SOL balance)      │
│ ├─ data: Vec<u8> (arbitrary bytes)  │
│ ├─ owner: Pubkey (program that      │
│ │          controls this account)    │
│ ├─ executable: bool (is it a        │
│ │              program?)             │
│ └─ rent_epoch: u64                   │
└─────────────────────────────────────┘

Key concepts:
1. Programs are STATELESS - they don't store data
2. Data lives in separate ACCOUNTS
3. Programs read/write accounts passed as instruction args
4. Only the OWNER program can modify an account's data
5. Anyone can credit lamports, only owner can debit

Account Types:
├─ System Account:  owned by System Program, holds SOL
├─ Program Account: executable, holds compiled BPF bytecode
├─ Data Account:    owned by a program, stores state
└─ Token Account:   owned by Token Program, holds SPL tokens

Rent:
- Accounts must pay rent or be rent-exempt
- Rent-exempt = ~2 years of rent upfront (~0.00089 SOL per byte)
- Accounts with 0 lamports and no data are garbage collected
```

---

### Q4: Explain PDAs (Program Derived Addresses).

**Answer:**

```rust
// PDA = deterministic address derived from seeds + program_id
// PDAs are NOT on the ed25519 curve → no private key exists
// Only the PROGRAM can "sign" transactions for a PDA

// How PDAs are derived:
// 1. Hash(seeds + program_id + bump)
// 2. Check if result is on ed25519 curve
// 3. If on curve, decrement bump and try again
// 4. First off-curve result = PDA

// bump = "bump seed" (0-255), usually 255 and decremented
```

```typescript
// Client-side PDA derivation
import { PublicKey } from '@solana/web3.js';

const [stakePda, bump] = PublicKey.findProgramAddressSync(
  [
    Buffer.from('stake'),             // seed 1: static string
    userPublicKey.toBuffer(),         // seed 2: user's pubkey
    poolPublicKey.toBuffer(),         // seed 3: pool's pubkey
  ],
  programId
);
// Always returns the same PDA for the same seeds + program_id
// bump is stored and passed to the program for verification
```

```rust
// Anchor - PDA in account validation
#[derive(Accounts)]
pub struct Stake<'info> {
    #[account(
        init_if_needed,
        payer = user,
        space = 8 + StakeAccount::INIT_SPACE,
        seeds = [b"stake", user.key().as_ref(), pool.key().as_ref()],
        bump, // Anchor finds and verifies the bump
    )]
    pub stake_account: Account<'info, StakeAccount>,
    // ...
}

// PDA as signer (program can sign on behalf of PDA)
let seeds = &[b"vault", pool.key().as_ref(), &[bump]];
let signer = &[&seeds[..]];

let cpi_ctx = CpiContext::new_with_signer(
    token_program,
    Transfer { from: vault, to: user_ata, authority: vault_pda },
    signer, // PDA signs the CPI
);
token::transfer(cpi_ctx, amount)?;
```

**PDA Use Cases:**
1. **Deterministic accounts** - Find account without storing address
2. **Program-controlled vaults** - PDA owns tokens, only program can transfer
3. **Unique per-user accounts** - seeds include user's pubkey
4. **Config/state accounts** - Global program state with known address
5. **Authority delegation** - PDA as mint authority, freeze authority

---

### Q5: Explain CPIs (Cross-Program Invocations).

**Answer:**

```rust
// CPI = one program calling another program's instruction
// Like a function call between smart contracts

// Example: Your program calls the Token Program to transfer tokens
use anchor_spl::token::{self, Transfer, Token, TokenAccount};

pub fn withdraw(ctx: Context<Withdraw>, amount: u64) -> Result<()> {
    // Build CPI context
    let cpi_accounts = Transfer {
        from: ctx.accounts.vault.to_account_info(),       // source token account
        to: ctx.accounts.user_token.to_account_info(),    // destination
        authority: ctx.accounts.vault_authority.to_account_info(), // PDA signer
    };

    let cpi_program = ctx.accounts.token_program.to_account_info();

    // PDA signs the CPI
    let seeds = &[b"vault", ctx.accounts.pool.key().as_ref(), &[ctx.bumps.vault_authority]];
    let signer_seeds = &[&seeds[..]];

    let cpi_ctx = CpiContext::new_with_signer(cpi_program, cpi_accounts, signer_seeds);

    token::transfer(cpi_ctx, amount)?;

    Ok(())
}

// CPI depth limit: 4 levels deep
// Program A → Program B → Program C → Program D (max)

// Security considerations:
// 1. Always validate accounts passed to CPI
// 2. Check program IDs (don't call fake programs)
// 3. Be careful with signer privileges (PDA authority)
// 4. Account re-entrancy attacks (check account state after CPI)
```

---

### Q6: Explain Solana Transaction structure.

**Answer:**

```
Transaction Structure:
┌───────────────────────────────────────┐
│ Transaction                            │
│ ├─ Signatures: Vec<Signature>         │ ← Ed25519 signatures
│ └─ Message                             │
│     ├─ Header                          │
│     │   ├─ num_required_signatures     │
│     │   ├─ num_readonly_signed         │
│     │   └─ num_readonly_unsigned       │
│     ├─ Account Keys: Vec<Pubkey>      │ ← all accounts referenced
│     ├─ Recent Blockhash               │ ← prevents replay, ~60s validity
│     └─ Instructions: Vec<Instruction> │
│         ├─ program_id_index           │ ← which program to call
│         ├─ account_indexes: Vec<u8>   │ ← which accounts to pass
│         └─ data: Vec<u8>             │ ← serialized instruction args
└───────────────────────────────────────┘

Limits:
- Max transaction size: 1232 bytes
- Max accounts per transaction: 64 (128 with Address Lookup Tables)
- Compute budget: 200K CU default (1.4M max with request)
```

```typescript
// Building a transaction (web3.js)
import {
  Transaction,
  SystemProgram,
  sendAndConfirmTransaction,
} from '@solana/web3.js';

const transaction = new Transaction();

// Add instructions
transaction.add(
  SystemProgram.transfer({
    fromPubkey: sender.publicKey,
    toPubkey: recipient,
    lamports: 1_000_000_000, // 1 SOL
  })
);

// Versioned transactions (v0) with Address Lookup Tables
import { TransactionMessage, VersionedTransaction } from '@solana/web3.js';

const messageV0 = new TransactionMessage({
  payerKey: sender.publicKey,
  recentBlockhash: blockhash,
  instructions: [transferIx, stakeIx, swapIx],
}).compileToV0Message([lookupTable]); // reduces tx size

const versionedTx = new VersionedTransaction(messageV0);
versionedTx.sign([sender]);
await connection.sendTransaction(versionedTx);
```

---

## Anchor Framework

### Q7: Complete Anchor program example - Token Staking.

**Answer:**

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, TokenAccount, Transfer, Mint};

declare_id!("Stake11111111111111111111111111111111111111");

#[program]
pub mod staking {
    use super::*;

    pub fn initialize_pool(
        ctx: Context<InitializePool>,
        reward_rate: u64, // rewards per second per staked token
    ) -> Result<()> {
        let pool = &mut ctx.accounts.pool;
        pool.authority = ctx.accounts.authority.key();
        pool.stake_mint = ctx.accounts.stake_mint.key();
        pool.reward_rate = reward_rate;
        pool.total_staked = 0;
        pool.last_update_time = Clock::get()?.unix_timestamp;
        pool.bump = ctx.bumps.pool;
        Ok(())
    }

    pub fn stake(ctx: Context<Stake>, amount: u64) -> Result<()> {
        require!(amount > 0, StakingError::InvalidAmount);

        // Update pending rewards before changing stake
        let clock = Clock::get()?;
        let stake_info = &mut ctx.accounts.stake_info;

        if stake_info.amount > 0 {
            let pending = calculate_rewards(
                stake_info.amount,
                ctx.accounts.pool.reward_rate,
                stake_info.last_stake_time,
                clock.unix_timestamp,
            );
            stake_info.pending_rewards += pending;
        }

        // Transfer tokens: user → vault
        let cpi_ctx = CpiContext::new(
            ctx.accounts.token_program.to_account_info(),
            Transfer {
                from: ctx.accounts.user_token_account.to_account_info(),
                to: ctx.accounts.vault.to_account_info(),
                authority: ctx.accounts.user.to_account_info(),
            },
        );
        token::transfer(cpi_ctx, amount)?;

        // Update state
        stake_info.amount += amount;
        stake_info.last_stake_time = clock.unix_timestamp;
        ctx.accounts.pool.total_staked += amount;

        emit!(StakeEvent {
            user: ctx.accounts.user.key(),
            amount,
            total_staked: stake_info.amount,
        });

        Ok(())
    }

    pub fn unstake(ctx: Context<Unstake>, amount: u64) -> Result<()> {
        let stake_info = &ctx.accounts.stake_info;
        require!(amount <= stake_info.amount, StakingError::InsufficientStake);

        // Transfer tokens: vault → user (PDA signs)
        let seeds = &[b"pool", ctx.accounts.pool.stake_mint.as_ref(), &[ctx.accounts.pool.bump]];
        let signer = &[&seeds[..]];

        let cpi_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            Transfer {
                from: ctx.accounts.vault.to_account_info(),
                to: ctx.accounts.user_token_account.to_account_info(),
                authority: ctx.accounts.pool.to_account_info(),
            },
            signer,
        );
        token::transfer(cpi_ctx, amount)?;

        // Update state
        let stake_info = &mut ctx.accounts.stake_info;
        stake_info.amount -= amount;
        ctx.accounts.pool.total_staked -= amount;

        Ok(())
    }
}

// Account structs
#[account]
#[derive(InitSpace)]
pub struct StakePool {
    pub authority: Pubkey,
    pub stake_mint: Pubkey,
    pub reward_rate: u64,
    pub total_staked: u64,
    pub last_update_time: i64,
    pub bump: u8,
}

#[account]
#[derive(InitSpace)]
pub struct StakeInfo {
    pub owner: Pubkey,
    pub amount: u64,
    pub pending_rewards: u64,
    pub last_stake_time: i64,
}

// Account validation
#[derive(Accounts)]
pub struct Stake<'info> {
    #[account(mut)]
    pub user: Signer<'info>,

    #[account(
        mut,
        seeds = [b"pool", pool.stake_mint.as_ref()],
        bump = pool.bump,
    )]
    pub pool: Account<'info, StakePool>,

    #[account(
        init_if_needed,
        payer = user,
        space = 8 + StakeInfo::INIT_SPACE,
        seeds = [b"stake", user.key().as_ref(), pool.key().as_ref()],
        bump,
    )]
    pub stake_info: Account<'info, StakeInfo>,

    #[account(
        mut,
        constraint = user_token_account.mint == pool.stake_mint,
        constraint = user_token_account.owner == user.key(),
    )]
    pub user_token_account: Account<'info, TokenAccount>,

    #[account(
        mut,
        seeds = [b"vault", pool.key().as_ref()],
        bump,
    )]
    pub vault: Account<'info, TokenAccount>,

    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
}

// Custom errors
#[error_code]
pub enum StakingError {
    #[msg("Invalid amount")]
    InvalidAmount,
    #[msg("Insufficient stake")]
    InsufficientStake,
}

// Events
#[event]
pub struct StakeEvent {
    pub user: Pubkey,
    pub amount: u64,
    pub total_staked: u64,
}

fn calculate_rewards(amount: u64, rate: u64, from: i64, to: i64) -> u64 {
    let duration = (to - from) as u64;
    amount.checked_mul(rate).unwrap().checked_mul(duration).unwrap() / 1_000_000
}
```

---

## DeFi Concepts

### Q8: Explain AMMs (Automated Market Makers).

**Answer:**

```
Traditional Exchange (Order Book):
  Buyers and sellers place orders at specific prices
  Matching engine matches buy/sell orders

AMM (Decentralized):
  No order book. Price determined by a mathematical formula.

Constant Product Formula (Uniswap/Raydium):
  x * y = k
  where x = token A reserve, y = token B reserve, k = constant

Example:
  Pool: 1000 SOL × 100,000 USDC = 100,000,000 (k)
  Price: 100,000 / 1000 = 100 USDC per SOL

  Buy 10 SOL:
  New SOL: 990, New USDC: 100,000,000 / 990 = 101,010.10
  Cost: 101,010.10 - 100,000 = 1,010.10 USDC for 10 SOL
  Effective price: 101.01 USDC/SOL (price impact: 1.01%)

Slippage:
  Larger trades → more price impact
  Low liquidity pools → more slippage

Impermanent Loss:
  LP provides 50/50 value. If one token moons, they have less
  of that token than if they just held. "Loss" compared to holding.

LP (Liquidity Provider):
  Deposits equal value of both tokens
  Earns trading fees (0.3% typical)
  Risks: impermanent loss, smart contract risk
```

---

### Q9: Explain Oracle integration (Pyth, Switchboard).

**Answer:**

```typescript
// Oracles bring off-chain data (prices) to on-chain programs

// Pyth Network (your experience)
import { PythSolanaReceiver } from '@pythnetwork/pyth-solana-receiver';

// In Anchor program:
#[derive(Accounts)]
pub struct ExecuteTrade<'info> {
    // Pyth price feed account
    /// CHECK: Verified by Pyth SDK
    pub price_feed: AccountInfo<'info>,
}

pub fn execute_trade(ctx: Context<ExecuteTrade>, amount: u64) -> Result<()> {
    // Read price from Pyth
    let price_feed = &ctx.accounts.price_feed;
    let price_data = PriceFeed::from_account_info(price_feed)?;

    let current_price = price_data
        .get_price_no_older_than(&Clock::get()?, 60) // max 60s old
        .ok_or(TradingError::StalePrice)?;

    let price = current_price.price as u64;
    let confidence = current_price.conf as u64;

    // Ensure price confidence is within acceptable range
    require!(
        confidence * 100 / price < 5, // less than 5% uncertainty
        TradingError::PriceUncertain
    );

    // Use price for trade execution
    let trade_value = amount * price / 10u64.pow(current_price.expo.unsigned_abs());

    Ok(())
}

// Client-side: Fetch Pyth price
import { PriceServiceConnection } from '@pythnetwork/price-service-client';

const connection = new PriceServiceConnection('https://hermes.pyth.network');
const priceIds = ['0xef0d8b6fda2ceba41da15d4095d1da392a0d2f8ed0c6c7bc0f4cfac8c280b56d']; // SOL/USD
const prices = await connection.getLatestPriceFeeds(priceIds);
```

---

## Frontend Integration

### Q10: Wallet integration with Privy.

**Answer:**

```typescript
// Privy - unified auth (email + social + wallet)
import { PrivyProvider, usePrivy, useWallets } from '@privy-io/react-auth';
import { useSolanaWallets } from '@privy-io/react-auth/solana';

function App() {
  return (
    <PrivyProvider
      appId={process.env.NEXT_PUBLIC_PRIVY_APP_ID!}
      config={{
        loginMethods: ['email', 'wallet', 'google', 'twitter'],
        appearance: {
          theme: 'dark',
          accentColor: '#7C3AED',
        },
        embeddedWallets: {
          createOnLogin: 'users-without-wallets',
          noPromptOnSignature: false,
        },
        solanaClusters: [
          { name: 'mainnet-beta', rpcUrl: process.env.NEXT_PUBLIC_SOLANA_RPC! },
        ],
      }}
    >
      <WalletApp />
    </PrivyProvider>
  );
}

function WalletApp() {
  const { login, logout, authenticated, user } = usePrivy();
  const { wallets } = useWallets();
  const { wallets: solanaWallets } = useSolanaWallets();

  const handleStake = async () => {
    const solanaWallet = solanaWallets[0];
    const connection = new Connection(RPC_URL);

    // Build transaction
    const tx = new Transaction();
    tx.add(
      await program.methods
        .stake(new BN(amount))
        .accounts({
          user: solanaWallet.address,
          pool: poolPda,
          stakeInfo: stakeInfoPda,
          userTokenAccount: userAta,
          vault: vaultPda,
          tokenProgram: TOKEN_PROGRAM_ID,
          systemProgram: SystemProgram.programId,
        })
        .instruction()
    );

    // Sign and send via Privy
    const signedTx = await solanaWallet.signTransaction(tx);
    const sig = await connection.sendRawTransaction(signedTx.serialize());
    await connection.confirmTransaction(sig, 'confirmed');
  };

  if (!authenticated) {
    return <button onClick={login}>Connect</button>;
  }

  return (
    <div>
      <p>Welcome {user.email || user.wallet?.address}</p>
      <button onClick={handleStake}>Stake SOL</button>
      <button onClick={logout}>Disconnect</button>
    </div>
  );
}
```

---

### Q11: Explain SPL Token standard on Solana.

**Answer:**

```typescript
// SPL Token = Solana Program Library Token (like ERC-20 on Ethereum)

// Key accounts:
// 1. Mint Account - defines the token (decimals, supply, authorities)
// 2. Token Account (ATA) - holds tokens for a specific user+mint pair
// 3. Associated Token Account - deterministic ATA address

import { createMint, mintTo, transfer, getOrCreateAssociatedTokenAccount }
  from '@solana/spl-token';

// Create a new token
const mint = await createMint(
  connection,
  payer,           // fee payer
  mintAuthority,   // who can mint new tokens
  freezeAuthority, // who can freeze accounts (null = no freeze)
  9,               // decimals (9 = standard for SOL-like tokens)
);

// Create Associated Token Account (ATA) for user
const userAta = await getOrCreateAssociatedTokenAccount(
  connection,
  payer,
  mint,
  userPublicKey,
);

// Mint tokens
await mintTo(
  connection,
  payer,
  mint,
  userAta.address,
  mintAuthority,
  1_000_000_000, // 1 token (with 9 decimals)
);

// Transfer tokens
await transfer(
  connection,
  payer,
  fromAta,
  toAta,
  owner,
  500_000_000, // 0.5 tokens
);

// Token-2022 (new standard):
// - Transfer fees
// - Interest-bearing tokens
// - Non-transferable tokens (soulbound)
// - Confidential transfers
// - Metadata extension (on-chain metadata)
```

---

## Indexing & Data

### Q12: Blockchain indexing approaches (Geyser, Helius, RPC).

**Answer:**

```
Comparison:
─────────────────────────────────────────────────
Method          Latency    Complexity   Cost
Geyser gRPC     ~100ms     High         Medium
Helius Webhooks ~1-2s      Low          Low
Helius DAS API  On-demand  Low          Low
RPC Polling     ~5-10s     Low          High (rate limits)
Custom Indexer  ~1-5s      Very High    Variable
```

```typescript
// 1. Geyser gRPC - Real-time streaming (lowest latency)
import Client from '@triton-one/yellowstone-grpc';

const client = new Client('https://grpc-endpoint.com', 'token');
const stream = await client.subscribe();

stream.write({
  accounts: {
    myProgram: {
      account: [],
      owner: ['YourProgramId'],
      filters: [
        {
          memcmp: {
            offset: 0,
            data: Buffer.from([/* discriminator */]).toString('base64'),
          },
        },
      ],
    },
  },
  commitment: CommitmentLevel.CONFIRMED,
});

stream.on('data', (update) => {
  if (update.account) {
    const data = update.account.account.data;
    const decoded = program.coder.accounts.decode('StakeInfo', Buffer.from(data));
    await db.upsert('stakes', decoded);
  }
});

// 2. Helius Webhooks - Managed, easy setup
const webhook = await helius.createWebhook({
  webhookURL: 'https://api.myapp.com/webhook/trades',
  transactionTypes: ['Any'],
  accountAddresses: [programId.toString()],
  webhookType: 'enhanced',
});

// Your webhook handler
app.post('/webhook/trades', async (req, res) => {
  const events = req.body;
  for (const event of events) {
    if (event.type === 'SWAP') {
      await processSwap(event);
    }
  }
  res.sendStatus(200);
});

// 3. Helius DAS API - Query digital assets
const { items } = await helius.rpc.getAssetsByOwner({
  ownerAddress: walletAddress,
  page: 1,
  limit: 100,
  displayOptions: { showFungible: true },
});

// Use cases by approach:
// Geyser: Real-time trading bots, live dashboards, DEX aggregation
// Helius Webhooks: Notification systems, analytics, moderate real-time
// DAS API: Portfolio viewers, NFT galleries, on-demand queries
// RPC Polling: Simple apps, low-frequency updates
```

---

### Q13: How do you handle transaction confirmation and errors on Solana?

**Answer:**

```typescript
// Transaction lifecycle on Solana:
// 1. Build → 2. Sign → 3. Send → 4. Confirm → 5. Finalize

// Confirmation levels:
// processed  → node processed it (may be rolled back)
// confirmed  → supermajority of cluster confirmed (~400ms)
// finalized  → 31+ confirmed blocks (~13s, safe for high-value)

async function sendAndConfirmWithRetry(
  connection: Connection,
  transaction: Transaction,
  signers: Keypair[],
  maxRetries = 3,
) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      // Get fresh blockhash (valid for ~60s)
      const { blockhash, lastValidBlockHeight } =
        await connection.getLatestBlockhash('confirmed');

      transaction.recentBlockhash = blockhash;
      transaction.sign(...signers);

      const signature = await connection.sendRawTransaction(
        transaction.serialize(),
        {
          skipPreflight: false,          // simulate first
          preflightCommitment: 'confirmed',
          maxRetries: 0,                 // we handle retries
        }
      );

      // Wait for confirmation with timeout
      const confirmation = await connection.confirmTransaction(
        {
          signature,
          blockhash,
          lastValidBlockHeight,
        },
        'confirmed'
      );

      if (confirmation.value.err) {
        throw new Error(`Transaction failed: ${JSON.stringify(confirmation.value.err)}`);
      }

      return signature;
    } catch (error) {
      if (attempt === maxRetries - 1) throw error;

      // Check if blockhash expired
      if (error.message.includes('Blockhash not found')) {
        continue; // retry with new blockhash
      }

      // Check for specific program errors
      if (error.logs) {
        const programError = parseAnchorError(error.logs);
        if (programError) throw programError; // don't retry program errors
      }

      await sleep(1000 * (attempt + 1)); // exponential backoff
    }
  }
}

// Priority fees (for congested network)
import { ComputeBudgetProgram } from '@solana/web3.js';

transaction.add(
  ComputeBudgetProgram.setComputeUnitLimit({ units: 200_000 }),
  ComputeBudgetProgram.setComputeUnitPrice({ microLamports: 50_000 }), // priority fee
);
```

---

## References & Deep Dive Resources

### Blockchain Fundamentals
| Topic | Resource |
|---|---|
| Blockchain Basics | [ethereum.org - Intro to Ethereum](https://ethereum.org/en/developers/docs/intro-to-ethereum/) |
| Consensus Mechanisms | [ethereum.org - Consensus Mechanisms](https://ethereum.org/en/developers/docs/consensus-mechanisms/) |
| Web3 University (Free) | [web3.university](https://www.web3.university/) |
| Crypto Zombies | [cryptozombies.io](https://cryptozombies.io/) — Interactive Solidity course |

### Solana
| Topic | Resource |
|---|---|
| Solana Docs (Official) | [solana.com/docs](https://solana.com/docs) |
| Solana Cookbook | [solanacookbook.com](https://solanacookbook.com/) — Practical recipes |
| Solana Account Model | [Solana Docs - Accounts](https://solana.com/docs/core/accounts) |
| PDAs Explained | [Solana Docs - PDAs](https://solana.com/docs/core/pda) |
| CPIs Explained | [Solana Docs - CPIs](https://solana.com/docs/core/cpi) |
| Transactions | [Solana Docs - Transactions](https://solana.com/docs/core/transactions) |
| SPL Token | [spl.solana.com/token](https://spl.solana.com/token) |
| Token-2022 | [spl.solana.com/token-2022](https://spl.solana.com/token-2022) |
| Solana Playground | [beta.solpg.io](https://beta.solpg.io/) — Browser IDE for Solana |
| Solana Stack Exchange | [solana.stackexchange.com](https://solana.stackexchange.com/) |
| Solana Bytes (YouTube) | [Solana Foundation (YouTube)](https://www.youtube.com/@SolanaFndn) |

### Anchor Framework
| Topic | Resource |
|---|---|
| Anchor Docs | [anchor-lang.com](https://www.anchor-lang.com/) |
| Anchor Book | [book.anchor-lang.com](https://book.anchor-lang.com/) — Comprehensive guide |
| Anchor Examples | [coral-xyz/anchor/examples](https://github.com/coral-xyz/anchor/tree/master/examples) |
| Anchor by Example | [examples.anchor-lang.com](https://examples.anchor-lang.com/) |
| Soldev.app | [soldev.app](https://soldev.app/) — Curated Solana dev resources |

### Ethereum & EVM
| Topic | Resource |
|---|---|
| Ethereum Docs | [ethereum.org/developers](https://ethereum.org/en/developers/) |
| Solidity Docs | [docs.soliditylang.org](https://docs.soliditylang.org/) |
| ERC Standards | [ethereum.org - Token Standards](https://ethereum.org/en/developers/docs/standards/tokens/) |
| Ethers.js Docs | [docs.ethers.org](https://docs.ethers.org/) |
| Viem Docs | [viem.sh](https://viem.sh/) — Modern alternative to ethers.js |
| Hardhat | [hardhat.org](https://hardhat.org/) — Ethereum dev environment |

### DeFi
| Topic | Resource |
|---|---|
| DeFi MOOC (Free) | [defi-learning.org](https://defi-learning.org/) — Stanford/Berkeley course |
| AMM Explained | [Uniswap V2 Docs](https://docs.uniswap.org/contracts/v2/concepts/protocol-overview/how-uniswap-works) |
| Impermanent Loss | [Chainlink - Impermanent Loss](https://chain.link/education-hub/impermanent-loss) |
| DeFi Llama | [defillama.com](https://defillama.com/) — DeFi analytics |
| Raydium Docs | [docs.raydium.io](https://docs.raydium.io/) — Solana AMM |
| Jupiter Docs | [station.jup.ag/docs](https://station.jup.ag/docs/) — Solana DEX aggregator |

### Wallet Integration
| Topic | Resource |
|---|---|
| Privy Docs | [docs.privy.io](https://docs.privy.io/) |
| Solana Wallet Adapter | [github.com/anza-xyz/wallet-adapter](https://github.com/anza-xyz/wallet-adapter) |
| Phantom Docs | [docs.phantom.app](https://docs.phantom.app/) |
| MetaMask Docs | [docs.metamask.io](https://docs.metamask.io/) |

### Indexing & Data
| Topic | Resource |
|---|---|
| Helius Docs | [docs.helius.dev](https://docs.helius.dev/) |
| Helius DAS API | [docs.helius.dev/solana-apis/digital-asset-standard-das-api](https://docs.helius.dev/solana-apis/digital-asset-standard-das-api) |
| Geyser gRPC | [docs.triton.one/project-yellowstone/yellowstone-grpc](https://docs.triton.one/project-yellowstone/yellowstone-grpc) |
| Pyth Network | [docs.pyth.network](https://docs.pyth.network/) |
| Switchboard | [docs.switchboard.xyz](https://docs.switchboard.xyz/) |
| Solana Explorer | [explorer.solana.com](https://explorer.solana.com/) |
| Solscan | [solscan.io](https://solscan.io/) |

### Payments
| Topic | Resource |
|---|---|
| NowPayments Docs | [nowpayments.io/help](https://nowpayments.io/help) |
| Solana Pay | [docs.solanapay.com](https://docs.solanapay.com/) |

---

> **Back to main**: [INTERVIEW_ROADMAP.md](../INTERVIEW_ROADMAP.md)
> **Prev**: [DevOps](./07-devops.md) | **Next**: [AI Integration](./09-ai-integration.md)
