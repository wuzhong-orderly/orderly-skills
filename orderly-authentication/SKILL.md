---
name: orderly-authentication
description: Complete guide to Orderly Network authentication — wallet signatures, account registration, Orderly key creation, REST API header authentication, WebSocket authentication, and working samples for TypeScript, Python, vanilla JS, cURL, and the Orderly SDK.
argument-hint: <auth-topic> [tech-stack]
---

# Orderly Authentication Skill

Use this skill for anything related to authenticating with the Orderly Network — registering accounts, creating API keys, signing requests, and authenticating WebSocket connections.

## Overview

Orderly Network uses a **two-layer authentication system** that supports both **EVM** and **Solana** wallets:

| Layer | EVM Standard | Solana Standard | Used For |
| :--- | :--- | :--- | :--- |
| **Wallet-level** | EIP-712 (typed data signing) | keccak256 + ABI-encode + `signMessage` | Account registration, Orderly key creation, withdrawals, settle PnL |
| **API-level** | ed25519 (request signing) | ed25519 (identical to EVM) | All private REST API calls and WebSocket authentication |

### Authentication Flow Summary

**EVM (MetaMask, WalletConnect, etc.):**
```
1. Connect EVM wallet
2. Register account   →  EIP-712 "Registration" signature     →  POST /v1/register_account
3. Create Orderly key →  EIP-712 "AddOrderlyKey" signature    →  POST /v1/orderly_key
4. Trade / query      →  ed25519 request signatures            →  Headers on every private request
5. WebSocket           →  ed25519 timestamp signature           →  Auth event or query-string
```

**Solana (Phantom, Solflare, etc.):**
```
1. Connect Solana wallet
2. Register account   →  keccak256(ABI-encode) + signMessage  →  POST /v1/register_account  (+ chainType: "SOL")
3. Create Orderly key →  keccak256(ABI-encode) + signMessage  →  POST /v1/orderly_key       (+ chainType: "SOL")
4. Trade / query      →  ed25519 request signatures (same)     →  Headers on every private request
5. WebSocket           →  ed25519 timestamp signature (same)    →  Auth event or query-string
```

> **Key Insight**: Steps 1–3 happen once (or when the key expires). Step 4 happens on every private API call. **After Orderly key creation, EVM and Solana behave identically** — the ed25519 request signing is the same. The only difference is *how the wallet signature is produced* during registration and key creation. The Orderly SDK handles all of this automatically for both chain types.

### EVM vs Solana: Key Differences

| Aspect | EVM | Solana |
| :--- | :--- | :--- |
| **Wallet signing method** | `eth_signTypedData_v4` (EIP-712) | `wallet.signMessage()` (raw bytes) |
| **Message format** | Structured EIP-712 JSON with domain + types | `keccak256(ABI-encode(hashed_fields))` → hex string → text-encode to bytes |
| **Chain IDs** | Native chain IDs (e.g., `421614` Arbitrum Sepolia) | Virtual chain IDs: `900900900` (mainnet), `901901901` (devnet) |
| **`chainType` in message** | `"EVM"` (or omitted for EVM-only) | `"SOL"` (required in message body) |
| **Signature output** | Hex string from EIP-712 | `"0x" + hex(ed25519_signature_bytes)` |
| **API endpoints** | Same (`/v1/register_account`, `/v1/orderly_key`) | Same endpoints |
| **Query params for lookups** | `chain_type=EVM` (or omit) | `chain_type=SOL` |
| **Request signing (post-setup)** | ed25519 (identical) | ed25519 (identical) |
| **Orderly key generation** | `@noble/ed25519` | `@noble/ed25519` (identical) |

---

## Configuration

| Environment | REST API Base URL | Public WS | Private WS |
| :--- | :--- | :--- | :--- |
| **Testnet** | `https://testnet-api.orderly.org` | `wss://ws-evm.orderly.org/ws/stream` | `wss://ws-private-evm.orderly.org/v2/ws/private/stream/{account_id}` |
| **Mainnet** | `https://api.orderly.org` | `wss://ws-evm.orderly.org/ws/stream` | `wss://ws-private-evm.orderly.org/v2/ws/private/stream/{account_id}` |

### Chain IDs

| Chain | Chain ID | Usage |
| :--- | :--- | :--- |
| Arbitrum Sepolia (EVM testnet) | `421614` | Default EVM testnet |
| Arbitrum One (EVM mainnet) | `42161` | EVM mainnet |
| Solana Devnet | `901901901` | Solana testnet (virtual chain ID) |
| Solana Mainnet | `900900900` | Solana mainnet (virtual chain ID) |

> **Note**: Solana doesn't natively use numeric chain IDs. Orderly assigns virtual chain IDs (`900900900` / `901901901`) to identify Solana within its omnichain system.

---

## Part 1: EIP-712 Typed Data (EVM Wallet Signatures)

> **This section applies to EVM wallets only** (MetaMask, WalletConnect, Coinbase Wallet, etc.). For Solana wallets, see [Part 1b: Solana Wallet Signatures](#part-1b-solana-wallet-signatures).

EIP-712 is used for **critical actions** that require wallet owner approval. The wallet prompts the user to sign structured data.

### EIP-712 Domain

There are two domains — **off-chain** (for registration and key creation) and **on-chain** (for withdrawals and settle PnL):

#### Off-Chain Domain (Registration + AddOrderlyKey)

```json
{
  "name": "Orderly",
  "version": "1",
  "chainId": <connected_chain_id>,
  "verifyingContract": "0xCcCCccccCCCCcCCCCCCcCcCccCcCCCcCcccccccC"
}
```

> The `verifyingContract` is always `0xCcCCccccCCCCcCCCCCCcCcCccCcCCCcCcccccccC` for off-chain operations. The `chainId` must match the user's connected chain (e.g., `421614` for Arbitrum Sepolia testnet).

#### On-Chain Domain (Withdraw + SettlePnl)

```json
{
  "name": "Orderly",
  "version": "1",
  "chainId": <connected_chain_id>,
  "verifyingContract": "<ledger_contract_address>"
}
```

> The Ledger contract address varies by network. Testnet example: `0x1826B75e2ef249173FC735149AE4B8e9ea10abff`. See the [Orderly addresses documentation](https://orderly.network/docs/build-on-omnichain/addresses) for current addresses.

### All EIP-712 Message Types

```js
{
  "EIP712Domain": [
    { "name": "name", "type": "string" },
    { "name": "version", "type": "string" },
    { "name": "chainId", "type": "uint256" },
    { "name": "verifyingContract", "type": "address" }
  ],
  "Registration": [
    { "name": "brokerId", "type": "string" },
    { "name": "chainId", "type": "uint256" },
    { "name": "timestamp", "type": "uint64" },
    { "name": "registrationNonce", "type": "uint256" }
  ],
  "AddOrderlyKey": [
    { "name": "brokerId", "type": "string" },
    { "name": "chainId", "type": "uint256" },
    { "name": "orderlyKey", "type": "string" },
    { "name": "scope", "type": "string" },
    { "name": "timestamp", "type": "uint64" },
    { "name": "expiration", "type": "uint64" }
  ],
  "Withdraw": [
    { "name": "brokerId", "type": "string" },
    { "name": "chainId", "type": "uint256" },
    { "name": "receiver", "type": "address" },
    { "name": "token", "type": "string" },
    { "name": "amount", "type": "uint256" },
    { "name": "withdrawNonce", "type": "uint64" },
    { "name": "timestamp", "type": "uint64" }
  ],
  "SettlePnl": [
    { "name": "brokerId", "type": "string" },
    { "name": "chainId", "type": "uint256" },
    { "name": "settleNonce", "type": "uint64" },
    { "name": "timestamp", "type": "uint64" }
  ],
  "DelegateSigner": [
    { "name": "delegateContract", "type": "address" },
    { "name": "brokerId", "type": "string" },
    { "name": "chainId", "type": "uint256" },
    { "name": "timestamp", "type": "uint64" },
    { "name": "registrationNonce", "type": "uint256" },
    { "name": "txHash", "type": "bytes32" }
  ],
  "DelegateAddOrderlyKey": [
    { "name": "delegateContract", "type": "address" },
    { "name": "brokerId", "type": "string" },
    { "name": "chainId", "type": "uint256" },
    { "name": "orderlyKey", "type": "string" },
    { "name": "scope", "type": "string" },
    { "name": "timestamp", "type": "uint64" },
    { "name": "expiration", "type": "uint64" }
  ],
  "DelegateWithdraw": [
    { "name": "delegateContract", "type": "address" },
    { "name": "brokerId", "type": "string" },
    { "name": "chainId", "type": "uint256" },
    { "name": "receiver", "type": "address" },
    { "name": "token", "type": "string" },
    { "name": "amount", "type": "uint256" },
    { "name": "withdrawNonce", "type": "uint64" },
    { "name": "timestamp", "type": "uint64" }
  ],
  "DelegateSettlePnl": [
    { "name": "delegateContract", "type": "address" },
    { "name": "brokerId", "type": "string" },
    { "name": "chainId", "type": "uint256" },
    { "name": "settleNonce", "type": "uint64" },
    { "name": "timestamp", "type": "uint64" }
  ]
}
```

---

## Part 1b: Solana Wallet Signatures

> **This section applies to Solana wallets only** (Phantom, Solflare, Backpack, etc.). For EVM wallets, see [Part 1](#part-1-eip-712-typed-data-evm-wallet-signatures).

Solana wallets cannot produce EIP-712 signatures. Instead, Orderly uses a **keccak256 hash of ABI-encoded parameters**, which the wallet signs as raw bytes via `signMessage()`.

### How Solana Signing Works

The signing process for each action:

1. **Hash each string field** individually with `solidityPackedKeccak256`
2. **ABI-encode** all fields (hashed strings + numeric values)
3. Take the **keccak256** of the ABI-encoded bytes
4. Convert the hash to a **hex string** (e.g., `"a1b2c3d4..."`) 
5. **Text-encode** the hex string to `Uint8Array` via `new TextEncoder().encode(hexString)`
6. Sign the `Uint8Array` with **`wallet.signMessage()`**
7. Prefix the result: `"0x" + bytesToHex(signatureBytes)`

### Solana Registration Message Construction

```typescript
import { keccak256 } from 'ethereum-cryptography/keccak';
import { bytesToHex, hexToBytes } from 'ethereum-cryptography/utils';
import { AbiCoder, solidityPackedKeccak256 } from 'ethers';

function buildSolanaRegisterMessage(inputs: {
  brokerId: string;
  chainId: number;    // 901901901 (devnet) or 900900900 (mainnet)
  timestamp: number;
  registrationNonce: string;
}) {
  const { brokerId, chainId, timestamp, registrationNonce } = inputs;

  // The message object sent to the API
  const message = { brokerId, chainId, timestamp, registrationNonce };

  // Build the bytes to sign:
  // 1. Hash the string field
  const brokerIdHash = solidityPackedKeccak256(['string'], [brokerId]);

  // 2. ABI-encode all fields
  const abicoder = AbiCoder.defaultAbiCoder();
  const encoded = abicoder.encode(
    ['bytes32', 'uint256', 'uint256', 'uint256'],
    [brokerIdHash, chainId, timestamp, registrationNonce]
  );

  // 3. keccak256 of encoded bytes
  const msgHash = keccak256(hexToBytes(encoded));

  // 4-5. Convert to hex string, then text-encode to Uint8Array
  const msgToSign = new TextEncoder().encode(bytesToHex(msgHash));

  return { message, msgToSign };
}
```

### Solana AddOrderlyKey Message Construction

```typescript
function buildSolanaAddKeyMessage(inputs: {
  brokerId: string;
  chainId: number;
  orderlyKey: string;  // "ed25519:<base58-pubkey>"
  scope: string;       // "read,trading"
  timestamp: number;
  expiration: number;
}) {
  const { brokerId, chainId, orderlyKey, scope, timestamp, expiration } = inputs;

  const message = {
    brokerId, chainId, orderlyKey, scope, timestamp, expiration,
    chainType: 'SOL'   // Required for Solana
  };

  // Hash all string fields individually
  const brokerIdHash = solidityPackedKeccak256(['string'], [brokerId]);
  const orderlyKeyHash = solidityPackedKeccak256(['string'], [orderlyKey]);
  const scopeHash = solidityPackedKeccak256(['string'], [scope]);

  const abicoder = AbiCoder.defaultAbiCoder();
  const encoded = abicoder.encode(
    ['bytes32', 'bytes32', 'bytes32', 'uint256', 'uint256', 'uint256'],
    [brokerIdHash, orderlyKeyHash, scopeHash, chainId, timestamp, expiration]
  );

  const msgHash = keccak256(hexToBytes(encoded));
  const msgToSign = new TextEncoder().encode(bytesToHex(msgHash));

  return { message, msgToSign };
}
```

### Solana Withdraw Message Construction

```typescript
import { decode as bs58Decode } from 'bs58';

function buildSolanaWithdrawMessage(inputs: {
  brokerId: string;
  chainId: number;
  receiver: string;  // Solana public key (base58)
  token: string;     // e.g., "USDC"
  amount: bigint;
  nonce: number;
}) {
  const { brokerId, chainId, receiver, token, amount, nonce } = inputs;
  const timestamp = Date.now();

  const message = {
    brokerId, chainId, receiver, token, amount,
    withdrawNonce: nonce, timestamp, chainType: 'SOL'
  };

  const brokerIdHash = solidityPackedKeccak256(['string'], [brokerId]);
  const tokenHash = solidityPackedKeccak256(['string'], [token]);
  const salt = keccak256(Buffer.from('Orderly Network'));

  const abicoder = AbiCoder.defaultAbiCoder();
  const encoded = abicoder.encode(
    ['bytes32', 'bytes32', 'uint256', 'bytes32', 'uint256', 'uint64', 'uint64', 'bytes32'],
    [brokerIdHash, tokenHash, chainId, bs58Decode(receiver), amount, nonce, timestamp, salt]
  );

  const msgHash = keccak256(hexToBytes(encoded));
  const msgToSign = new TextEncoder().encode(bytesToHex(msgHash));

  return { message, msgToSign };
}
```

### Solana SettlePnl Message Construction

```typescript
function buildSolanaSettleMessage(inputs: {
  brokerId: string;
  chainId: number;
  settlePnlNonce: number;
  timestamp: number;
}) {
  const { brokerId, chainId, settlePnlNonce, timestamp } = inputs;

  const message = {
    brokerId, chainId, timestamp,
    settleNonce: settlePnlNonce, chainType: 'SOL'
  };

  const brokerIdHash = solidityPackedKeccak256(['string'], [brokerId]);
  const abicoder = AbiCoder.defaultAbiCoder();
  const encoded = abicoder.encode(
    ['bytes32', 'uint256', 'uint64', 'uint64'],
    [brokerIdHash, chainId, settlePnlNonce, timestamp]
  );

  const msgHash = keccak256(hexToBytes(encoded));
  const msgToSign = new TextEncoder().encode(bytesToHex(msgHash));

  return { message, msgToSign };
}
```

### Full Solana Registration + Key Creation Example (Browser)

```typescript
import { getPublicKeyAsync, utils } from '@noble/ed25519';
import { keccak256 } from 'ethereum-cryptography/keccak';
import { bytesToHex, hexToBytes } from 'ethereum-cryptography/utils';
import { AbiCoder, solidityPackedKeccak256, encodeBase58 } from 'ethers';

const BASE_URL = 'https://testnet-api.orderly.org';
const BROKER_ID = 'woofi_dex';
const CHAIN_ID = 901901901; // Solana devnet

// Assumes Phantom or similar wallet is connected
// `solanaWallet` = wallet adapter instance with signMessage + publicKey

async function registerSolanaAccount(solanaWallet: {
  publicKey: { toBase58(): string };
  signMessage(message: Uint8Array): Promise<Uint8Array>;
}) {
  const userAddress = solanaWallet.publicKey.toBase58();

  // 1. Get registration nonce
  const nonceRes = await fetch(`${BASE_URL}/v1/registration_nonce`);
  const { data: { registration_nonce } } = await nonceRes.json();

  // 2. Build Solana signing message
  const timestamp = Date.now();
  const message = {
    brokerId: BROKER_ID,
    chainId: CHAIN_ID,
    timestamp,
    registrationNonce: registration_nonce
  };

  const brokerIdHash = solidityPackedKeccak256(['string'], [BROKER_ID]);
  const abicoder = AbiCoder.defaultAbiCoder();
  const encoded = abicoder.encode(
    ['bytes32', 'uint256', 'uint256', 'uint256'],
    [brokerIdHash, CHAIN_ID, timestamp, registration_nonce]
  );
  const msgHash = keccak256(hexToBytes(encoded));
  const msgToSign = new TextEncoder().encode(bytesToHex(msgHash));

  // 3. Sign with Solana wallet
  const signResult = await solanaWallet.signMessage(msgToSign);
  const signature = '0x' + bytesToHex(signResult);

  // 4. Register (note: chainType is NOT in message for registration, but userAddress is Solana pubkey)
  const regRes = await fetch(`${BASE_URL}/v1/register_account`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ message, signature, userAddress })
  });
  const regJson = await regRes.json();
  console.log('Solana account registered:', regJson.data.account_id);
  return regJson.data.account_id;
}

async function createSolanaOrderlyKey(solanaWallet: {
  publicKey: { toBase58(): string };
  signMessage(message: Uint8Array): Promise<Uint8Array>;
}) {
  const userAddress = solanaWallet.publicKey.toBase58();

  // 1. Generate ed25519 key pair (same as EVM)
  const privateKey = utils.randomPrivateKey();
  const orderlyKey = `ed25519:${encodeBase58(await getPublicKeyAsync(privateKey))}`;

  // 2. Build Solana signing message
  const timestamp = Date.now();
  const expiration = timestamp + 1_000 * 60 * 60 * 24 * 365; // 1 year
  const scope = 'read,trading';

  const message = {
    brokerId: BROKER_ID,
    chainId: CHAIN_ID,
    orderlyKey,
    scope,
    timestamp,
    expiration,
    chainType: 'SOL'  // Required for Solana key creation
  };

  const brokerIdHash = solidityPackedKeccak256(['string'], [BROKER_ID]);
  const orderlyKeyHash = solidityPackedKeccak256(['string'], [orderlyKey]);
  const scopeHash = solidityPackedKeccak256(['string'], [scope]);

  const abicoder = AbiCoder.defaultAbiCoder();
  const encoded = abicoder.encode(
    ['bytes32', 'bytes32', 'bytes32', 'uint256', 'uint256', 'uint256'],
    [brokerIdHash, orderlyKeyHash, scopeHash, CHAIN_ID, timestamp, expiration]
  );
  const msgHash = keccak256(hexToBytes(encoded));
  const msgToSign = new TextEncoder().encode(bytesToHex(msgHash));

  // 3. Sign with Solana wallet
  const signResult = await solanaWallet.signMessage(msgToSign);
  const signature = '0x' + bytesToHex(signResult);

  // 4. Register the key
  const keyRes = await fetch(`${BASE_URL}/v1/orderly_key`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ message, signature, userAddress })
  });
  const keyJson = await keyRes.json();
  console.log('Solana Orderly key created:', orderlyKey);

  // IMPORTANT: After this point, API request signing is identical to EVM
  // Use the same ed25519 signAndSendRequest() function from Part 4
  return { publicKey: orderlyKey, privateKey };
}
```

### Solana Ledger Wallet Support

Solana Ledger wallets cannot sign arbitrary messages via `signMessage()`. The SDK works around this by wrapping the message in a **Memo transaction instruction**:

```typescript
import { Transaction, TransactionInstruction } from '@solana/web3.js';

// For Ledger wallets, the SDK creates a transaction with a Memo instruction
// containing the message bytes, signs the transaction, and extracts the signature
async function signMessageWithLedger(
  message: Uint8Array,
  wallet: { signTransaction(tx: Transaction): Promise<Transaction>; publicKey: PublicKey },
  connection: Connection
): Promise<string> {
  const tx = new Transaction();
  tx.add(new TransactionInstruction({
    keys: [],
    programId: new PublicKey('MemoSq4gqABAXKb96qnH8TysNcWxMyWCqXgDLGmfcHr'),
    data: Buffer.from(message)
  }));
  tx.feePayer = wallet.publicKey;
  tx.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;

  const signedTx = await wallet.signTransaction(tx);
  const sig = signedTx.signatures[0].signature;
  return '0x' + Buffer.from(sig!).toString('hex');
}
```

---

## Part 2: Account Registration

### Account ID Calculation

Each account is deterministically identified by a keccak256 hash of the wallet address and broker ID:

```typescript
import { AbiCoder, keccak256, solidityPackedKeccak256 } from 'ethers';

function getAccountId(userAddress: string, brokerId: string): string {
  const abicoder = AbiCoder.defaultAbiCoder();
  return keccak256(
    abicoder.encode(
      ['address', 'bytes32'],
      [userAddress, solidityPackedKeccak256(['string'], [brokerId])]
    )
  );
}
```

### Pre-Check APIs (No Auth Required)

Before registration, check if the account already exists:

| Endpoint | Purpose |
| :--- | :--- |
| `GET /v1/get_account?address={addr}&broker_id={broker}` | Check if wallet is registered with a specific broker |
| `GET /v1/get_broker?address={addr}` | Check if address is registered with any broker |
| `GET /v1/account?account_id={id}` | Check if specific account_id exists |
| `GET /v1/get_all_accounts?address={addr}` | Get all accounts for an address |

> **Solana**: Add `&chain_type=SOL` to these query endpoints when checking Solana accounts. The `address` field should be the Solana public key (base58).

### Registration Steps

**EVM:**
1. **Get a registration nonce** — `GET /v1/registration_nonce` (valid for 2 minutes, single use)
2. **Sign EIP-712 `Registration` message** with the **off-chain domain**
3. **Post to** `POST /v1/register_account`

**Solana:**
1. **Get a registration nonce** — same endpoint `GET /v1/registration_nonce`
2. **Build keccak256(ABI-encode) message** and sign with `wallet.signMessage()`
3. **Post to** `POST /v1/register_account` — same endpoint, but `userAddress` is the Solana public key

### Registration API

#### `GET /v1/registration_nonce`
- **Auth:** None
- **Response:**
  ```json
  { "success": true, "data": { "registration_nonce": "194528949540" } }
  ```

#### `POST /v1/register_account`
- **Auth:** None (EIP-712 signature in body)
- **Request Body:**
  ```json
  {
    "message": {
      "brokerId": "woofi_dex",
      "chainId": 421614,
      "timestamp": 1685973017064,
      "registrationNonce": "194528949540"
    },
    "signature": "0x...",
    "userAddress": "0xYourWalletAddress"
  }
  ```
- **Response:**
  ```json
  { "success": true, "data": { "account_id": "0xabc123...", "user_id": 12345 } }
  ```

### Full Registration Example — TypeScript

```typescript
import { config } from 'dotenv';
import { ethers } from 'ethers';

config();

const MESSAGE_TYPES = {
  Registration: [
    { name: 'brokerId', type: 'string' },
    { name: 'chainId', type: 'uint256' },
    { name: 'timestamp', type: 'uint64' },
    { name: 'registrationNonce', type: 'uint256' }
  ]
};

const OFF_CHAIN_DOMAIN = {
  name: 'Orderly',
  version: '1',
  chainId: 421614,
  verifyingContract: '0xCcCCccccCCCCcCCCCCCcCcCccCcCCCcCcccccccC'
};

const BASE_URL = 'https://testnet-api.orderly.org';
const BROKER_ID = 'woofi_dex';
const CHAIN_ID = 421614;

async function registerAccount(): Promise<string> {
  const wallet = new ethers.Wallet(process.env.PRIVATE_KEY!);

  // Step 1: Get registration nonce
  const nonceRes = await fetch(`${BASE_URL}/v1/registration_nonce`);
  const nonceJson = await nonceRes.json();
  const registrationNonce = nonceJson.data.registration_nonce as string;

  // Step 2: Sign EIP-712 Registration message
  const registerMessage = {
    brokerId: BROKER_ID,
    chainId: CHAIN_ID,
    timestamp: Date.now(),
    registrationNonce
  };

  const signature = await wallet.signTypedData(
    OFF_CHAIN_DOMAIN,
    { Registration: MESSAGE_TYPES.Registration },
    registerMessage
  );

  // Step 3: Register account
  const registerRes = await fetch(`${BASE_URL}/v1/register_account`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      message: registerMessage,
      signature,
      userAddress: await wallet.getAddress()
    })
  });
  const registerJson = await registerRes.json();
  if (!registerJson.success) throw new Error(registerJson.message);

  console.log('Account ID:', registerJson.data.account_id);
  return registerJson.data.account_id;
}
```

### Full Registration Example — Python

```python
from datetime import datetime
import json
import math
import os
import requests
from eth_account import Account, messages

MESSAGE_TYPES = {
    "Registration": [
        {"name": "brokerId", "type": "string"},
        {"name": "chainId", "type": "uint256"},
        {"name": "timestamp", "type": "uint64"},
        {"name": "registrationNonce", "type": "uint256"},
    ],
}

OFF_CHAIN_DOMAIN = {
    "name": "Orderly",
    "version": "1",
    "chainId": 421614,
    "verifyingContract": "0xCcCCccccCCCCcCCCCCCcCcCccCcCCCcCcccccccC",
}

base_url = "https://testnet-api.orderly.org"
broker_id = "woofi_dex"
chain_id = 421614

account: Account = Account.from_key(os.environ.get("PRIVATE_KEY"))

# Step 1: Get registration nonce
res = requests.get(f"{base_url}/v1/registration_nonce")
registration_nonce = res.json()["data"]["registration_nonce"]

# Step 2: Sign EIP-712 Registration message
d = datetime.utcnow()
epoch = datetime(1970, 1, 1)
timestamp = math.trunc((d - epoch).total_seconds() * 1_000)

register_message = {
    "brokerId": broker_id,
    "chainId": chain_id,
    "timestamp": timestamp,
    "registrationNonce": registration_nonce,
}

encoded_data = messages.encode_typed_data(
    domain_data=OFF_CHAIN_DOMAIN,
    message_types={"Registration": MESSAGE_TYPES["Registration"]},
    message_data=register_message,
)
signed_message = account.sign_message(encoded_data)

# Step 3: Register account
res = requests.post(
    f"{base_url}/v1/register_account",
    headers={"Content-Type": "application/json"},
    json={
        "message": register_message,
        "signature": signed_message.signature.hex(),
        "userAddress": account.address,
    },
)
response = res.json()
orderly_account_id = response["data"]["account_id"]
print("Account ID:", orderly_account_id)
```

### Full Registration Example — Vanilla JS (Browser)

```javascript
// Requires: browser with MetaMask or any EIP-1193 wallet

const OFF_CHAIN_DOMAIN = {
  name: 'Orderly',
  version: '1',
  chainId: 421614, // Match the connected chain
  verifyingContract: '0xCcCCccccCCCCcCCCCCCcCcCccCcCCCcCcccccccC'
};

const REGISTRATION_TYPES = {
  Registration: [
    { name: 'brokerId', type: 'string' },
    { name: 'chainId', type: 'uint256' },
    { name: 'timestamp', type: 'uint64' },
    { name: 'registrationNonce', type: 'uint256' }
  ]
};

const BASE_URL = 'https://testnet-api.orderly.org';
const BROKER_ID = 'woofi_dex';
const CHAIN_ID = 421614;

async function registerAccount() {
  // Get user's wallet address via EIP-1193 provider
  const accounts = await window.ethereum.request({ method: 'eth_requestAccounts' });
  const userAddress = accounts[0];

  // Step 1: Get registration nonce
  const nonceRes = await fetch(`${BASE_URL}/v1/registration_nonce`);
  const { data: { registration_nonce } } = await nonceRes.json();

  // Step 2: Sign EIP-712 via eth_signTypedData_v4
  const registerMessage = {
    brokerId: BROKER_ID,
    chainId: CHAIN_ID,
    timestamp: Date.now(),
    registrationNonce: registration_nonce
  };

  const typedData = {
    types: { ...REGISTRATION_TYPES, EIP712Domain: [
      { name: 'name', type: 'string' },
      { name: 'version', type: 'string' },
      { name: 'chainId', type: 'uint256' },
      { name: 'verifyingContract', type: 'address' }
    ]},
    primaryType: 'Registration',
    domain: OFF_CHAIN_DOMAIN,
    message: registerMessage
  };

  const signature = await window.ethereum.request({
    method: 'eth_signTypedData_v4',
    params: [userAddress, JSON.stringify(typedData)]
  });

  // Step 3: Register
  const registerRes = await fetch(`${BASE_URL}/v1/register_account`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      message: registerMessage,
      signature,
      userAddress
    })
  });
  const result = await registerRes.json();
  console.log('Account ID:', result.data.account_id);
  return result.data.account_id;
}
```

---

## Part 3: Orderly Key Creation (ed25519)

The Orderly key is an ed25519 key pair used to sign all subsequent API requests. Creating one requires an EIP-712 wallet signature.

### Orderly Key Scopes

| Scope | Access |
| :--- | :--- |
| `read` | Private read-only APIs (positions, holdings, account info) |
| `trading` | Read + order APIs (create, edit, cancel orders) |
| `asset` | Asset-related operations |

Multiple scopes can be comma-separated: `"read,trading"`

### Orderly Key Expiration

Maximum allowed expiration: **365 days** from creation.

### Key Creation Steps

1. **Generate an ed25519 key pair** locally
2. **Encode the public key** as `ed25519:<Base58-encoded-pubkey>`
3. **Sign EIP-712 `AddOrderlyKey` message** with the off-chain domain
4. **Post to** `POST /v1/orderly_key`

### Key Creation API

#### `POST /v1/orderly_key`
- **Auth:** None (EIP-712 signature in body)
- **Request Body:**
  ```json
  {
    "message": {
      "brokerId": "woofi_dex",
      "chainId": 421614,
      "orderlyKey": "ed25519:HqN9uKJioHjAJZbadgQRGzq2e7huKg6foCyNY43hWbCk",
      "scope": "read,trading",
      "timestamp": 1685973094398,
      "expiration": 1717509094398
    },
    "signature": "0x...",
    "userAddress": "0xYourWalletAddress"
  }
  ```
- **Response:**
  ```json
  { "success": true, "data": { "orderly_key": "ed25519:HqN9...", "expiration": 1717509094398 } }
  ```

### Key Management APIs

| Method | Endpoint | Auth | Purpose |
| :--- | :--- | :--- | :--- |
| `GET` | `/v1/get_orderly_key?account_id={id}&orderly_key={key}` | None | Check if a specific key is valid |
| `GET` | `/v1/client/key_info?key_status=ACTIVE` | ed25519 | List all keys under account |
| `POST` | `/v1/client/remove_orderly_key` | ed25519 | Deactivate an Orderly key |

### Full Key Creation Example — TypeScript

```typescript
import { getPublicKeyAsync, utils } from '@noble/ed25519';
import { config } from 'dotenv';
import { encodeBase58, ethers } from 'ethers';
import { webcrypto } from 'node:crypto';

// Required in Node.js for @noble/ed25519
if (!globalThis.crypto) globalThis.crypto = webcrypto as any;

config();

const MESSAGE_TYPES = {
  AddOrderlyKey: [
    { name: 'brokerId', type: 'string' },
    { name: 'chainId', type: 'uint256' },
    { name: 'orderlyKey', type: 'string' },
    { name: 'scope', type: 'string' },
    { name: 'timestamp', type: 'uint64' },
    { name: 'expiration', type: 'uint64' }
  ]
};

const OFF_CHAIN_DOMAIN = {
  name: 'Orderly',
  version: '1',
  chainId: 421614,
  verifyingContract: '0xCcCCccccCCCCcCCCCCCcCcCccCcCCCcCcccccccC'
};

const BASE_URL = 'https://testnet-api.orderly.org';
const BROKER_ID = 'woofi_dex';
const CHAIN_ID = 421614;

async function createOrderlyKey(): Promise<{ publicKey: string; privateKey: Uint8Array }> {
  const wallet = new ethers.Wallet(process.env.PRIVATE_KEY!);

  // Step 1: Generate ed25519 key pair
  const privateKey = utils.randomPrivateKey();
  const orderlyKey = `ed25519:${encodeBase58(await getPublicKeyAsync(privateKey))}`;

  // Step 2: Sign EIP-712 AddOrderlyKey message
  const timestamp = Date.now();
  const addKeyMessage = {
    brokerId: BROKER_ID,
    chainId: CHAIN_ID,
    orderlyKey,
    scope: 'read,trading',
    timestamp,
    expiration: timestamp + 1_000 * 60 * 60 * 24 * 365 // 1 year
  };

  const signature = await wallet.signTypedData(
    OFF_CHAIN_DOMAIN,
    { AddOrderlyKey: MESSAGE_TYPES.AddOrderlyKey },
    addKeyMessage
  );

  // Step 3: Register the key
  const keyRes = await fetch(`${BASE_URL}/v1/orderly_key`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      message: addKeyMessage,
      signature,
      userAddress: await wallet.getAddress()
    })
  });
  const keyJson = await keyRes.json();
  if (!keyJson.success) throw new Error(keyJson.message);

  console.log('Orderly key created:', orderlyKey);

  // IMPORTANT: Save the privateKey securely — you need it for all future API requests
  return { publicKey: orderlyKey, privateKey };
}
```

### Full Key Creation Example — Python

```python
from datetime import datetime
import json
import math
import os
import requests
from base58 import b58encode
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PrivateKey
from eth_account import Account, messages

def encode_key(key_bytes: bytes) -> str:
    return f"ed25519:{b58encode(key_bytes).decode('utf-8')}"

MESSAGE_TYPES = {
    "AddOrderlyKey": [
        {"name": "brokerId", "type": "string"},
        {"name": "chainId", "type": "uint256"},
        {"name": "orderlyKey", "type": "string"},
        {"name": "scope", "type": "string"},
        {"name": "timestamp", "type": "uint64"},
        {"name": "expiration", "type": "uint64"},
    ],
}

OFF_CHAIN_DOMAIN = {
    "name": "Orderly",
    "version": "1",
    "chainId": 421614,
    "verifyingContract": "0xCcCCccccCCCCcCCCCCCcCcCccCcCCCcCcccccccC",
}

base_url = "https://testnet-api.orderly.org"
broker_id = "woofi_dex"
chain_id = 421614

account: Account = Account.from_key(os.environ.get("PRIVATE_KEY"))

# Step 1: Generate ed25519 key pair
orderly_private_key = Ed25519PrivateKey.generate()
orderly_public_key = encode_key(orderly_private_key.public_key().public_bytes_raw())

# Step 2: Sign EIP-712 AddOrderlyKey message
d = datetime.utcnow()
epoch = datetime(1970, 1, 1)
timestamp = math.trunc((d - epoch).total_seconds() * 1_000)

add_key_message = {
    "brokerId": broker_id,
    "chainId": chain_id,
    "orderlyKey": orderly_public_key,
    "scope": "read,trading",
    "timestamp": timestamp,
    "expiration": timestamp + 1_000 * 60 * 60 * 24 * 365,  # 1 year
}

encoded_data = messages.encode_typed_data(
    domain_data=OFF_CHAIN_DOMAIN,
    message_types={"AddOrderlyKey": MESSAGE_TYPES["AddOrderlyKey"]},
    message_data=add_key_message,
)
signed_message = account.sign_message(encoded_data)

# Step 3: Register the key
res = requests.post(
    f"{base_url}/v1/orderly_key",
    headers={"Content-Type": "application/json"},
    json={
        "message": add_key_message,
        "signature": signed_message.signature.hex(),
        "userAddress": account.address,
    },
)
response = res.json()
print("Orderly key created:", orderly_public_key)

# IMPORTANT: Save orderly_private_key securely — needed for all future API requests
```

### Full Key Creation Example — Vanilla JS (Browser)

```javascript
// Requires: browser with MetaMask + noble/ed25519 library loaded
// npm install @noble/ed25519  (or use a CDN/bundle)

import { getPublicKeyAsync, utils } from '@noble/ed25519';

// Base58 encoding (Bitcoin-style)
const BASE58_ALPHABET = '123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz';
function bytesToBase58(bytes) {
  let zeros = 0;
  for (let i = 0; i < bytes.length && bytes[i] === 0; i++) zeros++;
  const digits = [0];
  for (let i = 0; i < bytes.length; i++) {
    let carry = bytes[i];
    for (let j = 0; j < digits.length; j++) {
      carry += digits[j] << 8;
      digits[j] = carry % 58;
      carry = (carry / 58) | 0;
    }
    while (carry > 0) { digits.push(carry % 58); carry = (carry / 58) | 0; }
  }
  let result = BASE58_ALPHABET[0].repeat(zeros);
  for (let i = digits.length - 1; i >= 0; i--) result += BASE58_ALPHABET[digits[i]];
  return result;
}

const ADD_KEY_TYPES = {
  AddOrderlyKey: [
    { name: 'brokerId', type: 'string' },
    { name: 'chainId', type: 'uint256' },
    { name: 'orderlyKey', type: 'string' },
    { name: 'scope', type: 'string' },
    { name: 'timestamp', type: 'uint64' },
    { name: 'expiration', type: 'uint64' }
  ]
};

const OFF_CHAIN_DOMAIN = {
  name: 'Orderly', version: '1', chainId: 421614,
  verifyingContract: '0xCcCCccccCCCCcCCCCCCcCcCccCcCCCcCcccccccC'
};

const BASE_URL = 'https://testnet-api.orderly.org';
const BROKER_ID = 'woofi_dex';
const CHAIN_ID = 421614;

async function createOrderlyKey() {
  const accounts = await window.ethereum.request({ method: 'eth_requestAccounts' });
  const userAddress = accounts[0];

  // Step 1: Generate ed25519 key pair
  const privateKey = utils.randomPrivateKey();
  const publicKey = await getPublicKeyAsync(privateKey);
  const orderlyKey = `ed25519:${bytesToBase58(publicKey)}`;

  // Step 2: Sign EIP-712
  const timestamp = Date.now();
  const addKeyMessage = {
    brokerId: BROKER_ID,
    chainId: CHAIN_ID,
    orderlyKey,
    scope: 'read,trading',
    timestamp,
    expiration: timestamp + 1_000 * 60 * 60 * 24 * 365
  };

  const typedData = {
    types: {
      EIP712Domain: [
        { name: 'name', type: 'string' }, { name: 'version', type: 'string' },
        { name: 'chainId', type: 'uint256' }, { name: 'verifyingContract', type: 'address' }
      ],
      ...ADD_KEY_TYPES
    },
    primaryType: 'AddOrderlyKey',
    domain: OFF_CHAIN_DOMAIN,
    message: addKeyMessage
  };

  const signature = await window.ethereum.request({
    method: 'eth_signTypedData_v4',
    params: [userAddress, JSON.stringify(typedData)]
  });

  // Step 3: Register the key
  const res = await fetch(`${BASE_URL}/v1/orderly_key`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ message: addKeyMessage, signature, userAddress })
  });
  const result = await res.json();
  console.log('Key created:', orderlyKey);

  // Store privateKey in memory/localStorage (hex-encoded) for API signing
  localStorage.setItem('orderly_private_key', Array.from(privateKey).map(b => b.toString(16).padStart(2, '0')).join(''));

  return { publicKey: orderlyKey, privateKey };
}
```

---

## Part 4: REST API Header Authentication (ed25519)

Every private REST API request must include 4 authentication headers signed with your Orderly ed25519 key.

### Required Headers

| Header | Description | Example |
| :--- | :--- | :--- |
| `orderly-account-id` | Your account ID from registration | `0x0f29bfb4c1...` |
| `orderly-key` | ed25519 public key with `ed25519:` prefix | `ed25519:5FdH...` |
| `orderly-timestamp` | Current timestamp in milliseconds | `1649920583000` |
| `orderly-signature` | Base64URL-encoded ed25519 signature | `dG4bkKiqG0d...` |
| `Content-Type` | `application/x-www-form-urlencoded` for GET/DELETE, `application/json` for POST/PUT | |

### Signature Construction

The message to sign is a string concatenation of:

```
message = timestamp + method + path + body
```

| Component | Description | Example |
| :--- | :--- | :--- |
| `timestamp` | Same as `orderly-timestamp` header (ms) | `1649920583000` |
| `method` | HTTP method in UPPERCASE | `POST` |
| `path` | URL path **including** query params, **without** base URL | `/v1/orders?symbol=PERP_BTC_USDC` |
| `body` | JSON stringified body (empty string if no body) | `{"symbol":"PERP_ETH_USDC",...}` |

**Example message:**
```
1649920583000POST/v1/order{"symbol":"PERP_ETH_USDC","order_type":"LIMIT","order_price":1521.03,"order_quantity":2.11,"side":"BUY"}
```

Sign this message with the ed25519 private key, then encode the signature as **base64url** (no padding).

### Security Checks (3-Layer)

1. **Timestamp check** — Request rejected if `orderly-timestamp` differs from server time by > **300 seconds**
2. **Signature verification** — `orderly-signature` must be valid for the message content
3. **Key validity** — `orderly-key` must be registered, matched to account, and not expired

### Request Signing — TypeScript

```typescript
import { getPublicKeyAsync, signAsync } from '@noble/ed25519';
import { encodeBase58 } from 'ethers';

export async function signAndSendRequest(
  orderlyAccountId: string,
  privateKey: Uint8Array | string,
  input: URL | string,
  init?: RequestInit
): Promise<Response> {
  const timestamp = Date.now();
  const encoder = new TextEncoder();

  const url = new URL(input);
  let message = `${String(timestamp)}${init?.method ?? 'GET'}${url.pathname}${url.search}`;
  if (init?.body) {
    message += init.body;
  }
  const orderlySignature = await signAsync(encoder.encode(message), privateKey);

  return fetch(input, {
    headers: {
      'Content-Type':
        init?.method !== 'GET' && init?.method !== 'DELETE'
          ? 'application/json'
          : 'application/x-www-form-urlencoded',
      'orderly-timestamp': String(timestamp),
      'orderly-account-id': orderlyAccountId,
      'orderly-key': `ed25519:${encodeBase58(await getPublicKeyAsync(privateKey))}`,
      'orderly-signature': Buffer.from(orderlySignature).toString('base64url'),
      ...(init?.headers ?? {})
    },
    ...(init ?? {})
  });
}

// Usage:
// const res = await signAndSendRequest(accountId, privateKey, `${BASE_URL}/v1/client/holding`);
// const data = await res.json();
//
// const res = await signAndSendRequest(accountId, privateKey, `${BASE_URL}/v1/order`, {
//   method: 'POST',
//   body: JSON.stringify({ symbol: 'PERP_ETH_USDC', order_type: 'MARKET', order_quantity: 0.01, side: 'BUY' })
// });
```

### Request Signing — Python

```python
from base58 import b58encode
from base64 import urlsafe_b64encode
from datetime import datetime
import json
import math
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PrivateKey
from requests import PreparedRequest, Request
import urllib


def encode_key(key_bytes: bytes) -> str:
    return f"ed25519:{b58encode(key_bytes).decode('utf-8')}"


class Signer:
    def __init__(self, account_id: str, private_key: Ed25519PrivateKey):
        self._account_id = account_id
        self._private_key = private_key

    def sign_request(self, req: Request) -> PreparedRequest:
        d = datetime.utcnow()
        epoch = datetime(1970, 1, 1)
        timestamp = math.trunc((d - epoch).total_seconds() * 1_000)

        json_str = ""
        if req.json is not None:
            json_str = json.dumps(req.json)

        url = urllib.parse.urlparse(req.url)
        message = str(timestamp) + req.method + url.path + json_str
        if len(url.query) > 0:
            message += "?" + url.query

        orderly_signature = urlsafe_b64encode(
            self._private_key.sign(message.encode())
        ).decode("utf-8")

        req.headers = {
            "orderly-timestamp": str(timestamp),
            "orderly-account-id": self._account_id,
            "orderly-key": encode_key(self._private_key.public_key().public_bytes_raw()),
            "orderly-signature": orderly_signature,
        }
        if req.method == "GET" or req.method == "DELETE":
            req.headers["Content-Type"] = "application/x-www-form-urlencoded"
        elif req.method == "POST" or req.method == "PUT":
            req.headers["Content-Type"] = "application/json"

        return req.prepare()


# Usage:
# from base58 import b58decode
# from requests import Session
#
# key = b58decode(os.environ.get("ORDERLY_SECRET"))
# orderly_key = Ed25519PrivateKey.from_private_bytes(key)
# signer = Signer(orderly_account_id, orderly_key)
# session = Session()
#
# # GET request
# req = signer.sign_request(Request("GET", f"{base_url}/v1/client/holding"))
# res = session.send(req)
#
# # POST request (place order)
# req = signer.sign_request(Request("POST", f"{base_url}/v1/order", json={
#     "symbol": "PERP_ETH_USDC", "order_type": "MARKET", "order_quantity": 0.01, "side": "BUY"
# }))
# res = session.send(req)
```

### Request Signing — cURL

```bash
# cURL does not natively support ed25519 signing.
# Pre-compute the signature using the TypeScript or Python tools above,
# then pass it directly:

curl --request POST \
  --url https://api.orderly.org/v1/order \
  --header 'Content-Type: application/json' \
  --header 'orderly-account-id: <your-account-id>' \
  --header 'orderly-key: ed25519:8tm7dnKYkSc3FzgPuJaw1wztr79eeZpN35nHW5pL5XhX' \
  --header 'orderly-signature: <base64url-encoded-signature>' \
  --header 'orderly-timestamp: 1649920583000' \
  --data '{
    "symbol": "PERP_ETH_USDC",
    "order_type": "MARKET",
    "side": "BUY",
    "order_quantity": 0.01
  }'
```

---

## Part 5: WebSocket Authentication

### WebSocket Endpoints

| Type | URL |
| :--- | :--- |
| **Public** (no auth) | `wss://ws-evm.orderly.org/ws/stream` |
| **Private** (auth required) | `wss://ws-private-evm.orderly.org/v2/ws/private/stream/{account_id}` |

### WebSocket Signature

For WebSocket authentication, the message to sign is **only the timestamp** — no method, path, or body:

```
message = timestamp_string   // e.g., "1621910107900"
```

Sign this with ed25519 and encode as base64url (or hex).

### Method 1: Query String on Connection URL

Authenticate at connection time by appending auth params to the URL:

```
wss://ws-private-evm.orderly.org/v2/ws/private/stream/{account_id}?orderly_key={key}&timestamp={ts}&sign={signature}
```

### Method 2: Auth Event After Connection

Connect first, then send an auth event:

```json
{
  "id": "auth-request-1",
  "event": "auth",
  "params": {
    "orderly_key": "ed25519:CUS69ZJOXwSV38xo...",
    "sign": "base64url_encoded_signature",
    "timestamp": 1621910107900
  }
}
```

**Success response:**
```json
{
  "id": "auth-request-1",
  "event": "auth",
  "success": true,
  "ts": 1621910107315
}
```

### WebSocket Auth — TypeScript

```typescript
import { getPublicKeyAsync, signAsync } from '@noble/ed25519';
import { encodeBase58 } from 'ethers';
import WebSocket from 'ws';

async function connectPrivateWs(
  accountId: string,
  privateKey: Uint8Array
): Promise<WebSocket> {
  const timestamp = Date.now();
  const encoder = new TextEncoder();

  // For WebSocket, sign ONLY the timestamp
  const message = String(timestamp);
  const signature = await signAsync(encoder.encode(message), privateKey);
  const sign = Buffer.from(signature).toString('base64url');
  const orderlyKey = `ed25519:${encodeBase58(await getPublicKeyAsync(privateKey))}`;

  // Method 1: Auth via query string
  const wsUrl = `wss://ws-private-evm.orderly.org/v2/ws/private/stream/${accountId}?orderly_key=${encodeURIComponent(orderlyKey)}&timestamp=${timestamp}&sign=${encodeURIComponent(sign)}`;

  const ws = new WebSocket(wsUrl);

  ws.on('open', () => {
    console.log('Connected and authenticated');

    // Subscribe to private topics
    ws.send(JSON.stringify({
      id: 'sub-1',
      event: 'subscribe',
      topic: 'executionreport'
    }));
  });

  ws.on('message', (data) => {
    console.log('Received:', JSON.parse(data.toString()));
  });

  return ws;
}

// Method 2: Auth via event after connection
async function connectAndAuth(
  accountId: string,
  privateKey: Uint8Array
): Promise<WebSocket> {
  const ws = new WebSocket(
    `wss://ws-private-evm.orderly.org/v2/ws/private/stream/${accountId}`
  );

  ws.on('open', async () => {
    const timestamp = Date.now();
    const message = String(timestamp);
    const signature = await signAsync(new TextEncoder().encode(message), privateKey);
    const sign = Buffer.from(signature).toString('base64url');
    const orderlyKey = `ed25519:${encodeBase58(await getPublicKeyAsync(privateKey))}`;

    ws.send(JSON.stringify({
      id: 'auth-1',
      event: 'auth',
      params: {
        orderly_key: orderlyKey,
        sign,
        timestamp
      }
    }));
  });

  ws.on('message', (data) => {
    const msg = JSON.parse(data.toString());
    if (msg.event === 'auth' && msg.success) {
      console.log('WebSocket authenticated');
      // Now subscribe to private topics
      ws.send(JSON.stringify({ id: 'sub-1', event: 'subscribe', topic: 'executionreport' }));
    }
  });

  return ws;
}
```

### WebSocket Auth — Python

```python
import asyncio
import json
import math
from base58 import b58encode
from base64 import urlsafe_b64encode
from datetime import datetime
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PrivateKey
import websockets

def encode_key(key_bytes: bytes) -> str:
    return f"ed25519:{b58encode(key_bytes).decode('utf-8')}"

async def connect_private_ws(account_id: str, private_key: Ed25519PrivateKey):
    d = datetime.utcnow()
    epoch = datetime(1970, 1, 1)
    timestamp = math.trunc((d - epoch).total_seconds() * 1_000)

    # For WebSocket, sign ONLY the timestamp
    message = str(timestamp)
    sign = urlsafe_b64encode(private_key.sign(message.encode())).decode("utf-8")
    orderly_key = encode_key(private_key.public_key().public_bytes_raw())

    # Method 1: Auth via query string
    ws_url = (
        f"wss://ws-private-evm.orderly.org/v2/ws/private/stream/{account_id}"
        f"?orderly_key={orderly_key}&timestamp={timestamp}&sign={sign}"
    )

    async with websockets.connect(ws_url) as ws:
        print("Connected and authenticated")

        # Subscribe to private topics
        await ws.send(json.dumps({
            "id": "sub-1",
            "event": "subscribe",
            "topic": "executionreport"
        }))

        async for message in ws:
            data = json.loads(message)
            print("Received:", data)

# Method 2: Auth via event
async def connect_and_auth(account_id: str, private_key: Ed25519PrivateKey):
    ws_url = f"wss://ws-private-evm.orderly.org/v2/ws/private/stream/{account_id}"

    async with websockets.connect(ws_url) as ws:
        d = datetime.utcnow()
        epoch = datetime(1970, 1, 1)
        timestamp = math.trunc((d - epoch).total_seconds() * 1_000)

        message = str(timestamp)
        sign = urlsafe_b64encode(private_key.sign(message.encode())).decode("utf-8")

        await ws.send(json.dumps({
            "id": "auth-1",
            "event": "auth",
            "params": {
                "orderly_key": encode_key(private_key.public_key().public_bytes_raw()),
                "sign": sign,
                "timestamp": timestamp
            }
        }))

        async for message in ws:
            data = json.loads(message)
            if data.get("event") == "auth" and data.get("success"):
                print("Authenticated!")
                await ws.send(json.dumps({
                    "id": "sub-1", "event": "subscribe", "topic": "executionreport"
                }))
            else:
                print("Received:", data)
```

---

## Part 6: Orderly SDK Authentication (Automatic)

When using the Orderly JS SDK (`@orderly.network/react-app` or `@orderly.network/hooks`), **authentication is handled entirely automatically**. You do not need to manually sign requests, manage keys, or construct headers.

### How the SDK Handles Auth

The SDK uses an internal `Account` class with a **polymorphic wallet adapter** pattern — `DefaultEvmWalletAdapter` for EVM and `DefaultSolanaWalletAdapter` for Solana. Both implement the same `BaseWalletAdapter` interface, so the auth flow is transparent:

| SDK Action | EVM (Internal) | Solana (Internal) |
| :--- | :--- | :--- |
| Wallet connects | SDK checks `GET /v1/get_account?address=...&broker_id=...` | Same, with `&chain_type=SOL` |
| User clicks "Create Account" | EIP-712 `Registration` → `POST /v1/register_account` | keccak256+ABI-encode → `signMessage()` → same endpoint with `chainType: "SOL"` |
| User clicks "Enable Trading" | Generates ed25519 key → EIP-712 `AddOrderlyKey` → `POST /v1/orderly_key` | Generates ed25519 key → keccak256+ABI-encode → `signMessage()` → same endpoint with `chainType: "SOL"` |
| Any private API call | Identical ed25519 request signing | Identical ed25519 request signing |
| WebSocket connection | Identical ed25519 timestamp signing | Identical ed25519 timestamp signing |

> **After key creation, EVM and Solana are identical.** The `DefaultSolanaWalletAdapter` only differs in how it produces the initial wallet signatures for registration and key creation. All subsequent API/WebSocket auth uses the same ed25519 key.

### Account Status State Machine

The SDK tracks authentication state via `AccountStatusEnum`:

```typescript
enum AccountStatusEnum {
  EnableTradingWithoutConnected = -1, // Key imported, no wallet connected
  NotConnected     = 0,              // Wallet not connected
  Connected        = 1,              // Wallet connected, address known
  NotSignedIn      = 2,              // Account not registered on Orderly
  SignedIn         = 3,              // Account exists but no valid orderly key
  DisabledTrading  = 4,              // Account exists, orderly key expired
  EnableTrading    = 5               // Fully authenticated, ready to trade
}
```

### SDK Setup — Full App (React, EVM + Solana)

The `WalletConnectorProvider` supports both EVM and Solana wallets out of the box:

```tsx
import { OrderlyAppProvider } from '@orderly.network/react-app';
import { WalletConnectorProvider } from '@orderly.network/wallet-connector';

function App() {
  return (
    <WalletConnectorProvider>
      <OrderlyAppProvider
        brokerId="your_broker_id"
        brokerName="Your DEX"
        networkId="testnet"
        appIcons={{ main: { img: '/logo.png' } }}
      >
        <YourApp />
      </OrderlyAppProvider>
    </WalletConnectorProvider>
  );
}
```

> **That's it for both EVM and Solana.** The SDK detects the wallet type, uses the correct adapter (`DefaultEvmWalletAdapter` or `DefaultSolanaWalletAdapter`), and handles all signing automatically. The `<SignInDialog>` component provides the UI for the "Create Account" and "Enable Trading" flows for both chain types.

The SDK internally:
- Uses `@web3-onboard` for EVM wallet management
- Uses `@solana/wallet-adapter-react` for Solana wallet management (Phantom, Solflare, Backpack, Ledger)
- Routes signing to the correct adapter based on `ChainNamespace` (`evm` or `solana`)

### Using Auth State in Components

```tsx
import { useAccount } from '@orderly.network/hooks';
import { AccountStatusEnum } from '@orderly.network/types';

function TradingComponent() {
  const { state } = useAccount();

  if (state.status < AccountStatusEnum.EnableTrading) {
    // The SDK's ConnectWalletButton or your custom button triggers the sign-in flow
    return <p>Please connect wallet and enable trading</p>;
  }

  // All hooks that make private API calls work automatically when status === EnableTrading
  return <TradingUI />;
}
```

### SDK Hooks That Auto-Authenticate

These hooks automatically sign requests — no manual auth code needed:

```tsx
import { useQuery, useMutation } from '@orderly.network/hooks';

// Auto-signed private query
function Holdings() {
  const { data } = useQuery('/v1/client/holding');
  // data is auto-fetched with auth headers
  return <pre>{JSON.stringify(data, null, 2)}</pre>;
}

// Auto-signed private mutation
function PlaceOrder() {
  const [doPost] = useMutation('/v1/order', 'POST');

  const handleOrder = async () => {
    const result = await doPost({
      symbol: 'PERP_ETH_USDC',
      order_type: 'MARKET',
      order_quantity: 0.01,
      side: 'BUY'
    });
    console.log('Order placed:', result);
  };

  return <button onClick={handleOrder}>Buy ETH</button>;
}
```

### SDK Setup — Hooks Only (No UI Components)

If you're using `@orderly.network/hooks` directly (without `react-app`), use `OrderlyConfigProvider`:

```tsx
import { OrderlyConfigProvider } from '@orderly.network/hooks';

function App() {
  return (
    <OrderlyConfigProvider
      brokerId="your_broker_id"
      networkId="testnet"
    >
      <YourApp />
    </OrderlyConfigProvider>
  );
}
```

You'll need to implement your own sign-in UI using the `useAccount` hook:

```tsx
import { useAccount } from '@orderly.network/hooks';
import { AccountStatusEnum } from '@orderly.network/types';

function SignInButton() {
  const { account, state, createOrderlyKey, createAccount } = useAccount();

  const handleSignIn = async () => {
    if (state.status === AccountStatusEnum.NotSignedIn) {
      // Step 1: Register account (triggers EIP-712 wallet signature)
      await createAccount();
    }
    if (state.status === AccountStatusEnum.SignedIn || state.status === AccountStatusEnum.DisabledTrading) {
      // Step 2: Create orderly key (triggers EIP-712 wallet signature)
      await createOrderlyKey(true); // true = remember for 365 days
    }
  };

  return (
    <button onClick={handleSignIn} disabled={state.status === AccountStatusEnum.EnableTrading}>
      {state.status === AccountStatusEnum.EnableTrading ? 'Trading Enabled' : 'Sign In'}
    </button>
  );
}
```

---

## Part 7: Complete End-to-End Examples

### Example A: Full Flow — TypeScript (Node.js, No SDK)

Complete example: register account → create orderly key → place an order → connect WebSocket.

```typescript
import { getPublicKeyAsync, signAsync, utils } from '@noble/ed25519';
import { encodeBase58, ethers } from 'ethers';
import { webcrypto } from 'node:crypto';
import WebSocket from 'ws';

if (!globalThis.crypto) globalThis.crypto = webcrypto as any;

// ─── Config ───
const BASE_URL = 'https://testnet-api.orderly.org';
const BROKER_ID = 'woofi_dex';
const CHAIN_ID = 421614;

const MESSAGE_TYPES = {
  Registration: [
    { name: 'brokerId', type: 'string' },
    { name: 'chainId', type: 'uint256' },
    { name: 'timestamp', type: 'uint64' },
    { name: 'registrationNonce', type: 'uint256' }
  ],
  AddOrderlyKey: [
    { name: 'brokerId', type: 'string' },
    { name: 'chainId', type: 'uint256' },
    { name: 'orderlyKey', type: 'string' },
    { name: 'scope', type: 'string' },
    { name: 'timestamp', type: 'uint64' },
    { name: 'expiration', type: 'uint64' }
  ]
};

const OFF_CHAIN_DOMAIN = {
  name: 'Orderly',
  version: '1',
  chainId: CHAIN_ID,
  verifyingContract: '0xCcCCccccCCCCcCCCCCCcCcCccCcCCCcCcccccccC'
};

// ─── Helper: Sign & Send Request ───
async function signAndSendRequest(
  accountId: string,
  privateKey: Uint8Array,
  input: string,
  init?: RequestInit
): Promise<Response> {
  const timestamp = Date.now();
  const url = new URL(input);
  let message = `${timestamp}${init?.method ?? 'GET'}${url.pathname}${url.search}`;
  if (init?.body) message += init.body;

  const sig = await signAsync(new TextEncoder().encode(message), privateKey);

  return fetch(input, {
    ...init,
    headers: {
      'Content-Type': init?.method === 'POST' ? 'application/json' : 'application/x-www-form-urlencoded',
      'orderly-timestamp': String(timestamp),
      'orderly-account-id': accountId,
      'orderly-key': `ed25519:${encodeBase58(await getPublicKeyAsync(privateKey))}`,
      'orderly-signature': Buffer.from(sig).toString('base64url'),
      ...(init?.headers ?? {})
    }
  });
}

// ─── Main Flow ───
async function main() {
  const wallet = new ethers.Wallet(process.env.PRIVATE_KEY!);
  const userAddress = await wallet.getAddress();

  // 1. Check if already registered
  const checkRes = await fetch(`${BASE_URL}/v1/get_account?address=${userAddress}&broker_id=${BROKER_ID}`);
  const checkJson = await checkRes.json();
  let accountId: string;

  if (checkJson.success && checkJson.data?.account_id) {
    accountId = checkJson.data.account_id;
    console.log('Already registered:', accountId);
  } else {
    // 2. Register account
    const nonceRes = await fetch(`${BASE_URL}/v1/registration_nonce`);
    const { data: { registration_nonce } } = await nonceRes.json();

    const registerMessage = {
      brokerId: BROKER_ID, chainId: CHAIN_ID,
      timestamp: Date.now(), registrationNonce: registration_nonce
    };
    const regSig = await wallet.signTypedData(
      OFF_CHAIN_DOMAIN,
      { Registration: MESSAGE_TYPES.Registration },
      registerMessage
    );
    const regRes = await fetch(`${BASE_URL}/v1/register_account`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ message: registerMessage, signature: regSig, userAddress })
    });
    const regJson = await regRes.json();
    accountId = regJson.data.account_id;
    console.log('Registered:', accountId);
  }

  // 3. Create Orderly key
  const orderlySecret = utils.randomPrivateKey();
  const orderlyKey = `ed25519:${encodeBase58(await getPublicKeyAsync(orderlySecret))}`;
  const ts = Date.now();
  const addKeyMessage = {
    brokerId: BROKER_ID, chainId: CHAIN_ID, orderlyKey,
    scope: 'read,trading', timestamp: ts,
    expiration: ts + 1_000 * 60 * 60 * 24 * 365
  };
  const keySig = await wallet.signTypedData(
    OFF_CHAIN_DOMAIN,
    { AddOrderlyKey: MESSAGE_TYPES.AddOrderlyKey },
    addKeyMessage
  );
  const keyRes = await fetch(`${BASE_URL}/v1/orderly_key`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ message: addKeyMessage, signature: keySig, userAddress })
  });
  console.log('Key created:', (await keyRes.json()).data);

  // 4. Place an order
  const orderRes = await signAndSendRequest(accountId, orderlySecret, `${BASE_URL}/v1/order`, {
    method: 'POST',
    body: JSON.stringify({
      symbol: 'PERP_ETH_USDC',
      order_type: 'LIMIT',
      order_price: 1500,
      order_quantity: 0.01,
      side: 'BUY'
    })
  });
  console.log('Order:', await orderRes.json());

  // 5. Connect private WebSocket
  const wsTimestamp = Date.now();
  const wsSig = await signAsync(new TextEncoder().encode(String(wsTimestamp)), orderlySecret);
  const wsSign = Buffer.from(wsSig).toString('base64url');

  const ws = new WebSocket(
    `wss://ws-private-evm.orderly.org/v2/ws/private/stream/${accountId}` +
    `?orderly_key=${encodeURIComponent(orderlyKey)}&timestamp=${wsTimestamp}&sign=${encodeURIComponent(wsSign)}`
  );
  ws.on('open', () => {
    console.log('WS connected');
    ws.send(JSON.stringify({ id: '1', event: 'subscribe', topic: 'executionreport' }));
  });
  ws.on('message', (d) => console.log('WS:', JSON.parse(d.toString())));
}

main().catch(console.error);
```

### Example B: Full Flow — Python (No SDK)

```python
import asyncio
import json
import math
import os
from base58 import b58decode, b58encode
from base64 import urlsafe_b64encode
from datetime import datetime
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PrivateKey
from eth_account import Account, messages
from requests import Request, Session
import urllib
import websockets

# ─── Config ───
BASE_URL = "https://testnet-api.orderly.org"
BROKER_ID = "woofi_dex"
CHAIN_ID = 421614

OFF_CHAIN_DOMAIN = {
    "name": "Orderly", "version": "1", "chainId": CHAIN_ID,
    "verifyingContract": "0xCcCCccccCCCCcCCCCCCcCcCccCcCCCcCcccccccC",
}

REG_TYPES = {"Registration": [
    {"name": "brokerId", "type": "string"}, {"name": "chainId", "type": "uint256"},
    {"name": "timestamp", "type": "uint64"}, {"name": "registrationNonce", "type": "uint256"},
]}

KEY_TYPES = {"AddOrderlyKey": [
    {"name": "brokerId", "type": "string"}, {"name": "chainId", "type": "uint256"},
    {"name": "orderlyKey", "type": "string"}, {"name": "scope", "type": "string"},
    {"name": "timestamp", "type": "uint64"}, {"name": "expiration", "type": "uint64"},
]}


def encode_key(key_bytes): return f"ed25519:{b58encode(key_bytes).decode('utf-8')}"
def now_ms():
    d = datetime.utcnow(); return math.trunc((d - datetime(1970, 1, 1)).total_seconds() * 1_000)

# ─── Signer ───
class Signer:
    def __init__(self, account_id, private_key):
        self._account_id = account_id
        self._private_key = private_key

    def sign_request(self, req):
        timestamp = now_ms()
        json_str = json.dumps(req.json) if req.json else ""
        url = urllib.parse.urlparse(req.url)
        message = str(timestamp) + req.method + url.path + json_str
        if url.query: message += "?" + url.query

        req.headers = {
            "orderly-timestamp": str(timestamp),
            "orderly-account-id": self._account_id,
            "orderly-key": encode_key(self._private_key.public_key().public_bytes_raw()),
            "orderly-signature": urlsafe_b64encode(self._private_key.sign(message.encode())).decode("utf-8"),
            "Content-Type": "application/x-www-form-urlencoded" if req.method in ("GET", "DELETE") else "application/json",
        }
        return req.prepare()


def main():
    import requests as req_lib

    wallet = Account.from_key(os.environ["PRIVATE_KEY"])

    # 1. Check registration
    check = req_lib.get(f"{BASE_URL}/v1/get_account", params={"address": wallet.address, "broker_id": BROKER_ID}).json()
    if check.get("success") and check.get("data", {}).get("account_id"):
        account_id = check["data"]["account_id"]
        print(f"Already registered: {account_id}")
    else:
        # 2. Register
        nonce = req_lib.get(f"{BASE_URL}/v1/registration_nonce").json()["data"]["registration_nonce"]
        reg_msg = {"brokerId": BROKER_ID, "chainId": CHAIN_ID, "timestamp": now_ms(), "registrationNonce": nonce}
        encoded = messages.encode_typed_data(domain_data=OFF_CHAIN_DOMAIN, message_types=REG_TYPES, message_data=reg_msg)
        sig = wallet.sign_message(encoded)
        res = req_lib.post(f"{BASE_URL}/v1/register_account", json={
            "message": reg_msg, "signature": sig.signature.hex(), "userAddress": wallet.address
        }).json()
        account_id = res["data"]["account_id"]
        print(f"Registered: {account_id}")

    # 3. Create Orderly key
    orderly_secret = Ed25519PrivateKey.generate()
    orderly_pub = encode_key(orderly_secret.public_key().public_bytes_raw())
    ts = now_ms()
    key_msg = {
        "brokerId": BROKER_ID, "chainId": CHAIN_ID, "orderlyKey": orderly_pub,
        "scope": "read,trading", "timestamp": ts, "expiration": ts + 1_000 * 60 * 60 * 24 * 365
    }
    encoded = messages.encode_typed_data(domain_data=OFF_CHAIN_DOMAIN, message_types=KEY_TYPES, message_data=key_msg)
    sig = wallet.sign_message(encoded)
    key_res = req_lib.post(f"{BASE_URL}/v1/orderly_key", json={
        "message": key_msg, "signature": sig.signature.hex(), "userAddress": wallet.address
    }).json()
    print(f"Key created: {key_res}")

    # 4. Place order
    signer = Signer(account_id, orderly_secret)
    session = Session()
    order_req = signer.sign_request(Request("POST", f"{BASE_URL}/v1/order", json={
        "symbol": "PERP_ETH_USDC", "order_type": "LIMIT",
        "order_price": 1500, "order_quantity": 0.01, "side": "BUY"
    }))
    order_res = session.send(order_req)
    print(f"Order: {order_res.json()}")

    # 5. WebSocket
    asyncio.run(ws_connect(account_id, orderly_secret))


async def ws_connect(account_id, private_key):
    ts = now_ms()
    sign = urlsafe_b64encode(private_key.sign(str(ts).encode())).decode("utf-8")
    orderly_key = encode_key(private_key.public_key().public_bytes_raw())
    url = f"wss://ws-private-evm.orderly.org/v2/ws/private/stream/{account_id}?orderly_key={orderly_key}&timestamp={ts}&sign={sign}"

    async with websockets.connect(url) as ws:
        print("WS connected")
        await ws.send(json.dumps({"id": "1", "event": "subscribe", "topic": "executionreport"}))
        async for msg in ws:
            print(f"WS: {json.loads(msg)}")

if __name__ == "__main__":
    main()
```

---

## Part 8: Delegate Signer (Smart Contract Accounts)

Smart contracts cannot produce EIP-712 signatures directly. Orderly supports "Delegate Signer" to link an EOA (externally owned account) to a smart contract for trading.

### Flow

1. Smart contract calls `delegateSigner(VaultDelegate)` on the Orderly Vault contract
2. EOA confirms via `POST /v1/delegate_signer` with `DelegateSigner` EIP-712 type + tx hash
3. Add Orderly key via `POST /v1/delegate_orderly_key` with `DelegateAddOrderlyKey` EIP-712 type
4. Trade normally using the Orderly key
5. Withdraw/Settle PnL via `DelegateWithdraw`/`DelegateSettlePnl` types

### `POST /v1/delegate_signer`

```json
{
  "message": {
    "delegateContract": "0xSmartContractAddress",
    "brokerId": "woofi_pro",
    "chainId": 421614,
    "timestamp": 1704879369551,
    "registrationNonce": "161111791392",
    "txHash": "0xTransactionHash"
  },
  "signature": "0x...",
  "userAddress": "0xEOAAddress"
}
```

### `POST /v1/delegate_orderly_key`

```json
{
  "message": {
    "delegateContract": "0xSmartContractAddress",
    "brokerId": "woofi_pro",
    "chainId": 421614,
    "orderlyKey": "ed25519:...",
    "scope": "read,trading",
    "timestamp": 1704879369551,
    "expiration": 1736415369551
  },
  "signature": "0x...",
  "userAddress": "0xEOAAddress"
}
```

---

## Part 9: Required Libraries by Language

### TypeScript / JavaScript (Node.js)

```bash
npm install @noble/ed25519 ethers
# Optional for WebSocket: npm install ws
```

| Package | Purpose |
| :--- | :--- |
| `@noble/ed25519` | ed25519 key generation + request signing |
| `ethers` | EIP-712 typed data signing + Base58 encoding |
| `ws` | WebSocket client (Node.js only; browsers use native WebSocket) |

> **Node.js note:** Requires `globalThis.crypto = webcrypto` polyfill for `@noble/ed25519`:
> ```typescript
> import { webcrypto } from 'node:crypto';
> if (!globalThis.crypto) globalThis.crypto = webcrypto as any;
> ```

### Python

```bash
pip install eth-account cryptography base58 requests websockets
```

| Package | Purpose |
| :--- | :--- |
| `eth-account` | EIP-712 typed data signing |
| `cryptography` | ed25519 key generation + request signing (`Ed25519PrivateKey`) |
| `base58` | Base58 encoding for Orderly key format |
| `requests` | HTTP client |
| `websockets` | WebSocket client |

### Java

| Package | Purpose |
| :--- | :--- |
| `org.web3j:crypto` | EIP-712 typed data signing |
| `net.i2p.crypto:eddsa` | ed25519 key generation + signing |
| `org.bitcoinj:base58` | Base58 encoding |
| `com.squareup.okhttp3:okhttp` | HTTP client |

### TypeScript / JavaScript — Solana Wallet Signing

```bash
npm install @noble/ed25519 ethers ethereum-cryptography bs58
# Plus Solana wallet adapter:
npm install @solana/wallet-adapter-base @solana/wallet-adapter-react @solana/wallet-adapter-wallets @solana/web3.js
```

| Package | Purpose |
| :--- | :--- |
| `ethereum-cryptography` | `keccak256`, `bytesToHex`, `hexToBytes` for Solana message construction |
| `ethers` | `AbiCoder`, `solidityPackedKeccak256` for ABI encoding + field hashing |
| `bs58` | Base58 encode/decode for Solana addresses and keys |
| `@noble/ed25519` | ed25519 Orderly key generation (same as EVM) |
| `@solana/wallet-adapter-*` | Solana wallet connection and `signMessage()` |
| `@solana/web3.js` | Solana connection, transactions (for Ledger workaround + deposits) |

### Python — Solana Wallet Signing

```bash
pip install solders solana base58 pynacl eth-abi pysha3
```

| Package | Purpose |
| :--- | :--- |
| `solders` / `solana` | Solana keypair and transaction handling |
| `pynacl` | ed25519 signing for Solana native wallet operations |
| `eth-abi` | ABI encoding (equivalent to ethers `AbiCoder`) |
| `pysha3` | keccak256 hashing |
| `base58` | Base58 encoding |

### Orderly SDK (React — EVM + Solana)

```bash
npm install @orderly.network/react-app @orderly.network/wallet-connector
```

No manual auth code required — the SDK handles both EVM and Solana authentication automatically via its polymorphic wallet adapter system.

---

## Quick Reference: Auth Decision Tree

```
Are you using the Orderly JS SDK (@orderly.network/react-app or hooks)?
├── YES → Authentication is automatic for BOTH EVM and Solana.
│         Just wrap your app with providers. No manual signing needed.
│
└── NO → Manual authentication required:
    │
    ├── Which wallet type?
    │   ├── EVM (MetaMask, WalletConnect, etc.):
    │   │   ├── Register: EIP-712 "Registration" → POST /v1/register_account
    │   │   └── Add Key:  EIP-712 "AddOrderlyKey" → POST /v1/orderly_key
    │   │
    │   └── Solana (Phantom, Solflare, etc.):
    │       ├── Register: keccak256(ABI-encode) + signMessage → POST /v1/register_account
    │       └── Add Key:  keccak256(ABI-encode) + signMessage → POST /v1/orderly_key (+ chainType: "SOL")
    │
    ├── After key creation (IDENTICAL for both EVM and Solana):
    │   ├── Every REST API call:
    │   │   └── Sign: timestamp + method + path + body → ed25519 → base64url headers
    │   │
    │   ├── WebSocket:
    │   │   └── Sign: timestamp only → ed25519 → query string or auth event
    │   │
    │   └── Withdrawals / Settle PnL:
    │       ├── EVM:    EIP-712 "Withdraw" or "SettlePnl" with on-chain domain
    │       └── Solana:  keccak256(ABI-encode) + signMessage (with salt for withdraw)
```
