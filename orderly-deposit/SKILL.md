---
name: orderly-deposit
description: Complete guide to depositing funds into an Orderly Network trading account — EVM Vault contract interaction, token approval, cross-chain fees, Solana deposits, SDK hooks and UI components, swap deposits, and working examples in TypeScript, Python, and React.
argument-hint: <deposit-topic> [tech-stack]
---

# Orderly Deposit Skill

Use this skill for anything related to depositing tokens into an Orderly Network trading account — the on-chain Vault contract interaction, ERC-20 token approvals, cross-chain deposit fees, Solana SPL deposits, the SDK `useDeposit` hook, pre-built deposit UI components, and swap-and-deposit flows.

## Overview

Orderly Network uses a **cross-chain settlement architecture**. Users deposit tokens into a **Vault smart contract** on their chosen chain, and the funds are relayed via **LayerZero** to the **Orderly L2 settlement chain** (chain ID 291) where the trading account balance is credited.

### Architecture Layers

| Layer | Role in Deposits |
| :--- | :--- |
| **Asset Layer** | Vault contract on each supported chain where users deposit tokens |
| **Cross-Chain Layer** | LayerZero relays the deposit message from source chain → Orderly L2 |
| **Settlement Layer** | Orderly L2 (chain 291) — Ledger contract credits the user's balance |
| **Engine Layer** | Orderbook sees updated balance; user can trade |

### Deposit Flow Summary

```
EVM:
1. Approve token spending   →  token.approve(vaultAddress, amount)
2. Query cross-chain fee    →  vault.getDepositFee(user, depositData)  →  returns fee in WEI
3. Call deposit              →  vault.deposit(depositData, { value: fee })
4. Wait for finalization     →  LayerZero relays to Orderly L2  →  balance credited

Solana:
1. No token approval needed  (SPL tokens don't use ERC-20 allowance)
2. Query cross-chain fee     →  simulate depositToVault instruction
3. Call deposit              →  vault program deposit (versioned transaction via LayerZero)
4. Wait for finalization     →  LayerZero relays to Orderly L2  →  balance credited
```

> **Key Point**: After a deposit, funds are NOT immediately available. The cross-chain message must be delivered and confirmed on Orderly L2. This typically takes 1–5 minutes depending on the source chain, but can take longer (e.g., ~20 minutes for chains requiring many block confirmations like Polygon).

---

## Configuration

### Vault Contract Addresses

The Vault contract is deployed on every supported chain. **Most EVM chains share the same address.**

#### Mainnet

| Chain | Vault Address |
| :--- | :--- |
| Arbitrum One | `0x816f722424B49Cf1275cc86DA9840Fbd5a6167e9` |
| Optimism | `0x816f722424B49Cf1275cc86DA9840Fbd5a6167e9` |
| Base | `0x816f722424B49Cf1275cc86DA9840Fbd5a6167e9` |
| Ethereum | `0x816f722424B49Cf1275cc86DA9840Fbd5a6167e9` |
| Mantle | `0x816f722424B49Cf1275cc86DA9840Fbd5a6167e9` |
| Avalanche | `0x816f722424B49Cf1275cc86DA9840Fbd5a6167e9` |
| SEI | `0x816f722424B49Cf1275cc86DA9840Fbd5a6167e9` |
| Morph | `0x816f722424B49Cf1275cc86DA9840Fbd5a6167e9` |
| Sonic | `0x816f722424B49Cf1275cc86DA9840Fbd5a6167e9` |
| Monad | `0x816f722424B49Cf1275cc86DA9840Fbd5a6167e9` |
| Bera | `0x816f722424B49Cf1275cc86DA9840Fbd5a6167e9` |
| Story | `0x816f722424B49Cf1275cc86DA9840Fbd5a6167e9` |
| Mode | `0x816f722424B49Cf1275cc86DA9840Fbd5a6167e9` |
| Plume | `0x816f722424B49Cf1275cc86DA9840Fbd5a6167e9` |
| BNB Smart Chain | `0x816f722424B49Cf1275cc86DA9840Fbd5a6167e9` |
| Abstract | `0xE80F2396A266e898FBbD251b89CFE65B3e41fD18` |

#### Testnet

| Chain | Vault Address |
| :--- | :--- |
| Arbitrum Sepolia | `0x0EaC556c0C2321BA25b9DC01e4e3c95aD5CDCd2f` |
| Optimism Sepolia | `0xEfF2896077B6ff95379EfA89Ff903598190805EC` |
| Base Sepolia | `0xdc7348975aE9334DbdcB944DDa9163Ba8406a0ec` |
| Ethereum Sepolia | `0x0EaC556c0C2321BA25b9DC01e4e3c95aD5CDCd2f` |

#### Solana

| Environment | Vault Program |
| :--- | :--- |
| Mainnet | `ErBmAD61mGFKvrFNaTJuxoPwqrS8GgtwtqJTJVjFWx9Q` |
| Devnet | `9shwxWDUNhtwkHocsUAmrNAQfBH2DHh4njdAEdHZZkF2` |

### Testnet USDC (Arbitrum Sepolia)

| Token | Address | Decimals |
| :--- | :--- | :--- |
| USDC | `0x75faf114eafb1BDbe2F0316DF893fd58CE46AA4d` | 6 |

> **Testnet faucet**: `POST https://testnet-operator-evm.orderly.org/v1/faucet/usdc` — mints 1,000 testnet USDC to the caller's address.

### Supported Deposit Tokens by Chain

| Chain(s) | Supported Tokens |
| :--- | :--- |
| Most EVM chains | USDC |
| Arbitrum, Ethereum, BNB | USDC, USDT |
| Ethereum, BNB | USDC, USDT, YUSD, USD1 |
| Ethereum | USDC, USDT, YUSD, USD1, WBTC |
| Mantle | USDC.e (bridged USDC) |
| Solana | USDC (SPL) |

---

## Part 1: The VaultDepositFE Struct

Every deposit call requires a `VaultDepositFE` struct with four fields:

```solidity
struct VaultDepositFE {
    bytes32 accountId;    // Orderly account ID
    bytes32 brokerHash;   // keccak256 of broker ID string
    bytes32 tokenHash;    // keccak256 of token symbol string
    uint128 tokenAmount;  // Amount in token's smallest unit (e.g., USDC has 6 decimals)
}
```

### Field Derivation

#### accountId

The Orderly account ID is a deterministic keccak256 hash of the wallet address and broker:

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

// Example:
const accountId = getAccountId('0xYourWallet', 'woofi_dex');
```

#### brokerHash

```typescript
import { keccak256 } from 'ethers';
const brokerHash = keccak256(new TextEncoder().encode('woofi_dex'));
```

#### tokenHash

```typescript
import { keccak256 } from 'ethers';
const tokenHash = keccak256(new TextEncoder().encode('USDC'));
```

> **Common token hashes**: These are `keccak256` of the token **symbol string** (not the address). `keccak256("USDC")`, `keccak256("USDT")`, `keccak256("WBTC")`, etc.

#### tokenAmount

The amount in the token's smallest unit. For USDC (6 decimals):

```typescript
import { parseUnits } from 'ethers';
const tokenAmount = parseUnits('100', 6); // 100 USDC → 100000000n
```

---

## Part 2: EVM Deposit — Step by Step

### Prerequisites

- User has a connected EVM wallet
- User has a registered Orderly account (see `orderly-authentication` skill)
- User has an active Orderly key (see `orderly-authentication` skill)
- User has the deposit token (e.g., USDC) in their wallet

### Step 1: Approve Token Spending

Before depositing, you must authorize the Vault contract to transfer your tokens via the ERC-20 `approve()` function:

```typescript
import { ethers } from 'ethers';

const ERC20_ABI = [
  'function approve(address spender, uint256 amount) returns (bool)',
  'function allowance(address owner, address spender) view returns (uint256)',
];

const provider = new ethers.BrowserProvider(window.ethereum);
const signer = await provider.getSigner();
const usdcContract = new ethers.Contract(USDC_ADDRESS, ERC20_ABI, signer);

// Check current allowance
const currentAllowance = await usdcContract.allowance(signer.address, VAULT_ADDRESS);

// Approve if insufficient
if (currentAllowance < depositAmount) {
  // Option A: Approve exact amount
  const tx = await usdcContract.approve(VAULT_ADDRESS, depositAmount);
  await tx.wait();

  // Option B: Approve unlimited (common pattern to avoid repeated approvals)
  // const tx = await usdcContract.approve(VAULT_ADDRESS, ethers.MaxUint256);
  // await tx.wait();
}
```

> **USDT special case**: USDT on Ethereum mainnet requires setting the allowance to 0 before setting a new value (if the current allowance is > 0). The SDK handles this automatically.

### Step 2: Get the Deposit Fee

The deposit fee covers the LayerZero cross-chain message from the source chain to Orderly L2. It's paid in the source chain's **native token** (e.g., ETH on Arbitrum):

```typescript
const VAULT_ABI = [
  'function deposit((bytes32 accountId, bytes32 brokerHash, bytes32 tokenHash, uint128 tokenAmount) data) payable',
  'function depositTo(address receiver, (bytes32 accountId, bytes32 brokerHash, bytes32 tokenHash, uint128 tokenAmount) data) payable',
  'function getDepositFee(address receiver, (bytes32 accountId, bytes32 brokerHash, bytes32 tokenHash, uint128 tokenAmount) data) view returns (uint256)',
  'function depositFeeEnabled() view returns (bool)',
];

const vaultContract = new ethers.Contract(VAULT_ADDRESS, VAULT_ABI, signer);

const depositData = {
  accountId: getAccountId(signer.address, BROKER_ID),
  brokerHash: keccak256(new TextEncoder().encode(BROKER_ID)),
  tokenHash: keccak256(new TextEncoder().encode('USDC')),
  tokenAmount: parseUnits('100', 6),
};

const depositFee = await vaultContract.getDepositFee(signer.address, depositData);
console.log('Deposit fee (WEI):', depositFee.toString());
```

> **Tip**: The SDK applies a 5% buffer to the fee (`fee * 1.05`) to account for gas price fluctuations. You should do the same for raw API usage.

### Step 3: Execute the Deposit

Call `deposit()` on the Vault contract, passing the fee as `msg.value`:

```typescript
const tx = await vaultContract.deposit(depositData, { value: depositFee });
const receipt = await tx.wait();
console.log('Deposit tx:', receipt.hash);
```

### Step 4: Wait for Cross-Chain Confirmation

After the transaction confirms on the source chain, the deposit enters the cross-chain relay pipeline:

```
Source chain tx confirmed
    → LayerZero picks up the message
    → Message relayed to Orderly L2
    → VaultManager processes the deposit
    → Ledger credits user balance
```

You can monitor the status via:

1. **LayerZero Explorer**: Search the user's wallet address at `https://layerzeroscan.com/`
   - **In-flight** = processing
   - **Delivered** = confirmed
   - **No message found** = not yet received by LayerZero

2. **Orderly API**: `GET /v1/asset/history?side=DEPOSIT` (requires authentication)

---

## Part 3: Complete EVM Deposit Example — TypeScript

```typescript
import { AbiCoder, ethers, keccak256, parseUnits, solidityPackedKeccak256 } from 'ethers';

// === Configuration ===
const BROKER_ID = 'woofi_dex';
const USDC_ADDRESS = '0x75faf114eafb1BDbe2F0316DF893fd58CE46AA4d'; // Testnet USDC
const VAULT_ADDRESS = '0x0EaC556c0C2321BA25b9DC01e4e3c95aD5CDCd2f'; // Testnet Vault
const RPC_URL = 'https://arbitrum-sepolia.publicnode.com';

// === ABIs ===
const ERC20_ABI = [
  'function approve(address spender, uint256 amount) returns (bool)',
  'function allowance(address owner, address spender) view returns (uint256)',
  'function balanceOf(address account) view returns (uint256)',
];

const VAULT_ABI = [
  'function deposit((bytes32 accountId, bytes32 brokerHash, bytes32 tokenHash, uint128 tokenAmount) data) payable',
  'function getDepositFee(address receiver, (bytes32 accountId, bytes32 brokerHash, bytes32 tokenHash, uint128 tokenAmount) data) view returns (uint256)',
];

// === Helpers ===
function getAccountId(address: string, brokerId: string): string {
  const abicoder = AbiCoder.defaultAbiCoder();
  return keccak256(
    abicoder.encode(
      ['address', 'bytes32'],
      [address, solidityPackedKeccak256(['string'], [brokerId])]
    )
  );
}

// === Main Deposit Function ===
async function deposit(privateKey: string, amountUsdc: string): Promise<string> {
  const provider = new ethers.JsonRpcProvider(RPC_URL);
  const wallet = new ethers.Wallet(privateKey, provider);

  const usdc = new ethers.Contract(USDC_ADDRESS, ERC20_ABI, wallet);
  const vault = new ethers.Contract(VAULT_ADDRESS, VAULT_ABI, wallet);

  const amount = parseUnits(amountUsdc, 6); // USDC has 6 decimals

  // 1. Check balance
  const balance = await usdc.balanceOf(wallet.address);
  if (balance < amount) {
    throw new Error(`Insufficient USDC: have ${balance}, need ${amount}`);
  }

  // 2. Approve
  const allowance = await usdc.allowance(wallet.address, VAULT_ADDRESS);
  if (allowance < amount) {
    console.log('Approving USDC...');
    const approveTx = await usdc.approve(VAULT_ADDRESS, amount);
    await approveTx.wait();
    console.log('Approved');
  }

  // 3. Build deposit data
  const encoder = new TextEncoder();
  const depositData = {
    accountId: getAccountId(wallet.address, BROKER_ID),
    brokerHash: keccak256(encoder.encode(BROKER_ID)),
    tokenHash: keccak256(encoder.encode('USDC')),
    tokenAmount: amount,
  };

  // 4. Get deposit fee
  const fee = await vault.getDepositFee(wallet.address, depositData);
  console.log('Deposit fee:', ethers.formatEther(fee), 'ETH');

  // 5. Execute deposit
  const tx = await vault.deposit(depositData, { value: fee });
  const receipt = await tx.wait();
  console.log('Deposit tx confirmed:', receipt.hash);
  return receipt.hash;
}

// Usage
deposit(process.env.PRIVATE_KEY!, '100').catch(console.error);
```

---

## Part 4: Complete EVM Deposit Example — Python

```python
import json
import os
from eth_account import Account
from eth_abi import encode
from web3 import Web3

# === Configuration ===
BROKER_ID = "woofi_dex"
TOKEN_ID = "USDC"
USDC_ADDRESS = Web3.to_checksum_address("0x75faf114eafb1BDbe2F0316DF893fd58CE46AA4d")
VAULT_ADDRESS = Web3.to_checksum_address("0x0EaC556c0C2321BA25b9DC01e4e3c95aD5CDCd2f")
RPC_URL = "https://arbitrum-sepolia.publicnode.com"

# === USDC ABI (minimal) ===
ERC20_ABI = json.loads("""[
  {"inputs":[{"name":"spender","type":"address"},{"name":"amount","type":"uint256"}],
   "name":"approve","outputs":[{"name":"","type":"bool"}],"stateMutability":"nonpayable","type":"function"},
  {"inputs":[{"name":"owner","type":"address"},{"name":"spender","type":"address"}],
   "name":"allowance","outputs":[{"name":"","type":"uint256"}],"stateMutability":"view","type":"function"},
  {"inputs":[{"name":"account","type":"address"}],
   "name":"balanceOf","outputs":[{"name":"","type":"uint256"}],"stateMutability":"view","type":"function"}
]""")

# === Vault ABI (minimal — deposit + getDepositFee) ===
VAULT_ABI = json.loads("""[
  {"inputs":[{"components":[
    {"name":"accountId","type":"bytes32"},{"name":"brokerHash","type":"bytes32"},
    {"name":"tokenHash","type":"bytes32"},{"name":"tokenAmount","type":"uint128"}
  ],"name":"data","type":"tuple"}],
  "name":"deposit","outputs":[],"stateMutability":"payable","type":"function"},
  {"inputs":[{"name":"receiver","type":"address"},{"components":[
    {"name":"accountId","type":"bytes32"},{"name":"brokerHash","type":"bytes32"},
    {"name":"tokenHash","type":"bytes32"},{"name":"tokenAmount","type":"uint128"}
  ],"name":"data","type":"tuple"}],
  "name":"getDepositFee","outputs":[{"name":"","type":"uint256"}],"stateMutability":"view","type":"function"}
]""")


def get_account_id(address: str, broker_id: str) -> bytes:
    """Derive the Orderly account ID from wallet address + broker."""
    broker_hash = Web3.solidity_keccak(["string"], [broker_id])
    encoded = encode(["address", "bytes32"], [address, broker_hash])
    return Web3.keccak(encoded)


def deposit(private_key: str, amount_usdc: float):
    """Deposit USDC into the Orderly Vault on Arbitrum Sepolia."""
    w3 = Web3(Web3.HTTPProvider(RPC_URL))
    account = Account.from_key(private_key)
    w3.middleware_onion.add(
        Web3.middleware.construct_sign_and_send_raw_middleware(account)
    )

    usdc = w3.eth.contract(address=USDC_ADDRESS, abi=ERC20_ABI)
    vault = w3.eth.contract(address=VAULT_ADDRESS, abi=VAULT_ABI)

    amount = int(amount_usdc * 10**6)  # USDC has 6 decimals

    # 1. Check balance
    balance = usdc.functions.balanceOf(account.address).call()
    assert balance >= amount, f"Insufficient USDC: have {balance}, need {amount}"

    # 2. Approve
    allowance = usdc.functions.allowance(account.address, VAULT_ADDRESS).call()
    if allowance < amount:
        print("Approving USDC...")
        tx = usdc.functions.approve(VAULT_ADDRESS, amount).transact(
            {"from": account.address}
        )
        w3.eth.wait_for_transaction_receipt(tx)
        print("Approved")

    # 3. Build deposit data
    account_id = get_account_id(account.address, BROKER_ID)
    broker_hash = Web3.keccak(text=BROKER_ID)
    token_hash = Web3.keccak(text=TOKEN_ID)

    deposit_input = (
        account_id,   # accountId (bytes32)
        broker_hash,  # brokerHash (bytes32)
        token_hash,   # tokenHash (bytes32)
        amount,       # tokenAmount (uint128)
    )

    # 4. Get deposit fee (in WEI)
    deposit_fee = vault.functions.getDepositFee(
        Web3.to_checksum_address(account.address), deposit_input
    ).call()
    print(f"Deposit fee: {Web3.from_wei(deposit_fee, 'ether')} ETH")

    # 5. Execute deposit
    tx = vault.functions.deposit(deposit_input).transact(
        {"from": account.address, "value": deposit_fee}
    )
    receipt = w3.eth.wait_for_transaction_receipt(tx)
    print(f"Deposit tx confirmed: {receipt['transactionHash'].hex()}")
    return receipt["transactionHash"].hex()


if __name__ == "__main__":
    deposit(os.environ["PRIVATE_KEY"], 100.0)
```

---

## Part 5: Solana Deposit

Solana deposits follow the same conceptual flow but use the Solana Vault program instead of an EVM smart contract. SPL tokens (like USDC) do not need an `approve()` step.

### Key Differences from EVM

| Aspect | EVM | Solana |
| :--- | :--- | :--- |
| **Token approval** | Required (`ERC-20 approve()`) | Not needed (SPL tokens) |
| **Fee calculation** | `vault.getDepositFee()` view call | Simulated on-chain transaction |
| **Transaction format** | Standard EVM transaction | Versioned Transaction with Address Lookup Tables |
| **Cross-chain relay** | LayerZero EVM endpoint | LayerZero Solana endpoint |
| **Deposit methods** | `deposit()` or `depositTo()` | `deposit()` for SPL, `depositNative()` for SOL |
| **Compute budget** | Gas limit auto-estimated | Explicit 400,000 compute units |
| **Vault program** | Solidity contract | Anchor program (Rust) |

### Solana Deposit Parameters

The Solana Vault program uses the same logical fields as EVM, encoded as Anchor instruction parameters:

```typescript
// Anchor program parameters for Solana deposit
interface SolanaDepositParams {
  accountId: number[];    // bytes32 as byte array
  brokerHash: number[];   // keccak256 of broker ID
  tokenHash: number[];    // keccak256 of token symbol
  tokenAmount: BN;        // amount in token decimals (BN from @coral-xyz/anchor)
}
```

### Solana Deposit Flow (SDK Handles Automatically)

When using the Orderly SDK, Solana deposits are handled transparently by the `DefaultSolanaWalletAdapter`:

1. The SDK detects the Solana wallet connection
2. The `useDeposit` hook works identically for Solana — same API, same UI
3. Internally, the adapter:
   - Derives all required PDAs (Program Derived Addresses) for broker/token allowlists
   - Gets the cross-chain fee by simulating a `getDepositFee` transaction
   - Builds a Versioned Transaction with compute budget + address lookup tables
   - Calls `deposit()` (for SPL tokens) or `depositNative()` (for SOL) on the Vault program
   - Sends via the connected Solana wallet

> **For raw Solana deposits without the SDK**, refer to the `DefaultSolanaWalletAdapter` source code in `@orderly.network/default-solana-adapter`. The complexity of PDA derivation and LayerZero endpoint integration makes the SDK strongly recommended for Solana deposits.

---

## Part 6: Orderly SDK — useDeposit Hook

The `useDeposit` hook from `@orderly.network/hooks` handles the entire deposit flow automatically, including token approval, fee calculation, and cross-chain deposit.

### Import and Setup

```tsx
import { useDeposit } from '@orderly.network/hooks';
```

### Hook Parameters

```typescript
interface UseDepositOptions {
  address?: string;     // Token contract address (e.g., USDC address on source chain)
  decimals?: number;    // Token decimals (e.g., 6 for USDC)
  srcToken?: string;    // Source token symbol (e.g., "USDC")
  srcChainId?: number;  // Source chain ID (e.g., 421614 for Arb Sepolia)
}
```

### Hook Return Values

```typescript
const {
  // State
  balance,               // string — Wallet balance of the selected token
  allowance,             // string — Current ERC-20 allowance for the Vault
  quantity,              // string — User-entered deposit amount
  depositFee,            // bigint — Cross-chain fee in WEI (with 5% buffer)
  isNativeToken,         // boolean — Whether the selected token is the native token (ETH, SOL, etc.)
  dst,                   // object — Destination chain info (USDC address, chainId, decimals, network)

  // Loading states
  balanceRevalidating,   // boolean — Loading state for balance fetch
  allowanceRevalidating, // boolean — Loading state for allowance fetch
  depositFeeRevalidating,// boolean — Loading state for fee query

  // Actions
  setQuantity,           // (amount: string) => void — Set the deposit amount
  approve,               // (amount?: string) => Promise — Approve token spending
  deposit,               // () => Promise — Execute the deposit
  fetchBalance,          // (address: string, decimals: number) => Promise — Fetch balance for a specific token
  fetchBalances,         // (tokens: TokenInfo[]) => Promise — Batch-fetch balances
} = useDeposit(options);
```

### Basic Usage

```tsx
import { useDeposit } from '@orderly.network/hooks';
import { useState } from 'react';

function DepositForm() {
  const [amount, setAmount] = useState('');

  const {
    balance,
    allowance,
    deposit,
    approve,
    setQuantity,
    depositFee,
    isNativeToken,
  } = useDeposit({
    address: '0x75faf114eafb1BDbe2F0316DF893fd58CE46AA4d', // USDC
    decimals: 6,
    srcToken: 'USDC',
    srcChainId: 421614, // Arbitrum Sepolia
  });

  const handleDeposit = async () => {
    try {
      // Check if approval is needed (not for native tokens)
      if (!isNativeToken && Number(allowance) < Number(amount)) {
        await approve(amount);
      }

      // Set quantity and execute deposit
      setQuantity(amount);
      await deposit();
      alert('Deposit submitted! Waiting for cross-chain confirmation...');
    } catch (error) {
      console.error('Deposit failed:', error);
    }
  };

  return (
    <div>
      <p>Wallet Balance: {balance} USDC</p>
      <p>Current Allowance: {allowance}</p>
      <p>Deposit Fee: {depositFee ? `${Number(depositFee) / 1e18} ETH` : 'Loading...'}</p>
      <input
        type="number"
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
        placeholder="Amount to deposit"
      />
      <button onClick={handleDeposit}>
        {!isNativeToken && Number(allowance) < Number(amount) ? 'Approve & Deposit' : 'Deposit'}
      </button>
    </div>
  );
}
```

### Advanced: Approve + Deposit with Loading States

```tsx
import { useDeposit, useAccount } from '@orderly.network/hooks';
import { AccountStatusEnum } from '@orderly.network/types';

function AdvancedDepositForm({ tokenAddress, tokenDecimals, chainId }: {
  tokenAddress: string;
  tokenDecimals: number;
  chainId: number;
}) {
  const { state } = useAccount();
  const [amount, setAmount] = useState('');
  const [loading, setLoading] = useState(false);

  const {
    balance,
    allowance,
    deposit,
    approve,
    setQuantity,
    depositFee,
    depositFeeRevalidating,
    isNativeToken,
  } = useDeposit({
    address: tokenAddress,
    decimals: tokenDecimals,
    srcToken: 'USDC',
    srcChainId: chainId,
  });

  // Must be fully authenticated to deposit
  if (state.status < AccountStatusEnum.EnableTrading) {
    return <p>Please connect wallet and enable trading first.</p>;
  }

  const needsApproval = !isNativeToken && Number(allowance) < Number(amount);

  const handleDeposit = async () => {
    setLoading(true);
    try {
      if (needsApproval) {
        // Approve first (SDK defaults to MaxUint256 if no amount specified)
        await approve(amount);
      }
      setQuantity(amount);
      await deposit();
    } catch (err: any) {
      console.error('Deposit error:', err.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div>
      <p>Balance: {balance} USDC</p>
      <p>Fee: {depositFeeRevalidating ? 'Calculating...' : `${Number(depositFee) / 1e18} ETH`}</p>
      <input
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
        max={balance}
      />
      <button onClick={handleDeposit} disabled={loading || !amount}>
        {loading ? 'Processing...' : needsApproval ? 'Approve & Deposit' : 'Deposit'}
      </button>
    </div>
  );
}
```

---

## Part 7: Orderly SDK — Pre-built Deposit UI

The SDK provides ready-to-use deposit UI components in `@orderly.network/ui-transfer`.

### DepositForm Widget

```tsx
import { DepositForm } from '@orderly.network/ui-transfer';

function DepositPage() {
  return (
    <DepositForm
      onOk={() => {
        console.log('Deposit successful!');
      }}
      onCancel={() => {
        console.log('Deposit cancelled');
      }}
    />
  );
}
```

The `DepositForm` widget includes:
- **Chain selector** — dropdown to pick the source chain
- **Token picker** — select which token to deposit
- **Amount input** — with a "Max" button
- **Balance display** — shows wallet balance
- **Fee display** — shows the cross-chain deposit fee
- **Action button** — dynamically shows "Approve", "Deposit", or "Approve & Deposit"
- **Swap support** — can auto-swap non-USDC tokens into USDC before depositing (if enabled)

### DepositAndWithdrawWidget (Tabbed)

```tsx
import { DepositAndWithdrawWidget } from '@orderly.network/ui-transfer';

function TransferPage() {
  return (
    <DepositAndWithdrawWidget
      depositConfig={{
        onOk: () => console.log('Deposited!'),
      }}
      withdrawConfig={{
        onOk: () => console.log('Withdrawn!'),
      }}
    />
  );
}
```

This renders a tabbed interface with "Deposit" and "Withdraw" tabs.

### Enabling Swap Deposits

To allow users to deposit from non-USDC tokens (the SDK swaps via WOOFi DEX before depositing):

```tsx
<OrderlyAppProvider
  brokerId="your_broker_id"
  networkId="mainnet"
  enableSwapDeposit  // Enable swap + deposit flow
>
  <WalletConnectorProvider enableSwapDeposit>
    <YourApp />
  </WalletConnectorProvider>
</OrderlyAppProvider>
```

With swap deposits enabled, the `DepositForm` will show additional tokens beyond USDC and handle the swap atomically.

---

## Part 8: Vault Contract ABI Reference

### Key Functions for Deposits

#### deposit

Deposits tokens into the user's Orderly trading account.

```json
{
  "type": "function",
  "name": "deposit",
  "inputs": [{
    "name": "data",
    "type": "tuple",
    "internalType": "struct VaultTypes.VaultDepositFE",
    "components": [
      { "name": "accountId", "type": "bytes32" },
      { "name": "brokerHash", "type": "bytes32" },
      { "name": "tokenHash", "type": "bytes32" },
      { "name": "tokenAmount", "type": "uint128" }
    ]
  }],
  "stateMutability": "payable"
}
```

- **`msg.value`**: Must include the cross-chain fee returned by `getDepositFee()`
- **Emits**: `AccountDeposit(accountId, userAddress, depositNonce, tokenHash, tokenAmount)`

#### depositTo

Deposits tokens on behalf of another address (used for delegate signer / smart contract accounts):

```json
{
  "type": "function",
  "name": "depositTo",
  "inputs": [
    { "name": "receiver", "type": "address" },
    {
      "name": "data",
      "type": "tuple",
      "components": [
        { "name": "accountId", "type": "bytes32" },
        { "name": "brokerHash", "type": "bytes32" },
        { "name": "tokenHash", "type": "bytes32" },
        { "name": "tokenAmount", "type": "uint128" }
      ]
    }
  ],
  "stateMutability": "payable"
}
```

- **`receiver`**: The address that owns the Orderly account being deposited into
- **Emits**: `AccountDepositTo(accountId, userAddress, depositNonce, tokenHash, tokenAmount)`

#### getDepositFee

Returns the cross-chain fee (in WEI of the native token) required for a deposit:

```json
{
  "type": "function",
  "name": "getDepositFee",
  "inputs": [
    { "name": "receiver", "type": "address" },
    {
      "name": "data",
      "type": "tuple",
      "components": [
        { "name": "accountId", "type": "bytes32" },
        { "name": "brokerHash", "type": "bytes32" },
        { "name": "tokenHash", "type": "bytes32" },
        { "name": "tokenAmount", "type": "uint128" }
      ]
    }
  ],
  "outputs": [{ "name": "", "type": "uint256" }],
  "stateMutability": "view"
}
```

#### depositFeeEnabled

Check if the deposit fee is currently active:

```json
{
  "type": "function",
  "name": "depositFeeEnabled",
  "outputs": [{ "name": "", "type": "bool" }],
  "stateMutability": "view"
}
```

### Events

#### AccountDeposit

Emitted on successful `deposit()`:

```solidity
event AccountDeposit(
    bytes32 indexed accountId,
    address indexed userAddress,
    uint64  indexed depositNonce,
    bytes32 tokenHash,
    uint128 tokenAmount
);
```

#### AccountDepositTo

Emitted on successful `depositTo()`:

```solidity
event AccountDepositTo(
    bytes32 indexed accountId,
    address indexed userAddress,
    uint64  indexed depositNonce,
    bytes32 tokenHash,
    uint128 tokenAmount
);
```

### Common Contract Errors

| Error | Cause |
| :--- | :--- |
| `ZeroDeposit` | Deposit amount is zero |
| `ZeroDepositFee` | Deposit fee is enabled but `msg.value` is zero |
| `DepositExceedLimit` | Deposit exceeds the per-token limit set by admin |
| `TokenNotAllowed` | Token hash is not whitelisted in the Vault |
| `BrokerNotAllowed` | Broker hash is not whitelisted in the Vault |
| `AccountIdInvalid` | The `accountId` doesn't match `keccak256(msg.sender, brokerHash)` |

---

## Part 9: Checking Deposit Status

### API Endpoint

```
GET /v1/asset/history?side=DEPOSIT
```

**Requires authentication** (ed25519 signed headers — see `orderly-authentication` skill).

| Parameter | Type | Description |
| :--- | :--- | :--- |
| `side` | string | `"DEPOSIT"` |
| `token` | string | Filter by token (e.g., `"USDC"`) |
| `status` | string | `"NEW"` / `"CONFIRM"` / `"PROCESSING"` / `"COMPLETED"` / `"FAILED"` |
| `start_t` / `end_t` | number | Timestamp range (ms) |
| `page` / `size` | number | Pagination |

**Rate limit**: 10 requests per 60 seconds.

### Response

```json
{
  "success": true,
  "data": {
    "rows": [
      {
        "id": "123456",
        "tx_id": "0xabc123...",
        "side": "DEPOSIT",
        "token": "USDC",
        "amount": 100.0,
        "fee": 0,
        "trans_status": "COMPLETED",
        "created_time": 1700000000000,
        "updated_time": 1700000060000,
        "chain_id": 421614
      }
    ]
  }
}
```

### Deposit Status Flow

```
NEW → CONFIRM → PROCESSING → COMPLETED
                           → FAILED
```

### TypeScript Status Check

```typescript
async function checkDepositStatus(baseUrl: string, headers: Record<string, string>) {
  const res = await fetch(`${baseUrl}/v1/asset/history?side=DEPOSIT&size=5`, {
    headers,
  });
  const { data } = await res.json();
  for (const row of data.rows) {
    console.log(`${row.token}: ${row.amount} — ${row.trans_status} (tx: ${row.tx_id})`);
  }
}
```

### LayerZero Explorer

For cross-chain message tracking, search the user's wallet address on the LayerZero Explorer:

```
https://layerzeroscan.com/address/{userAddress}
```

---

## Part 10: Swap Deposits (Advanced)

The Orderly SDK supports **swap deposits** — depositing from any supported token, not just USDC. The SDK uses **WOOFi DEX** to swap the source token to USDC before depositing into the Vault.

### Same-Chain Swap Deposit

If the user has a non-USDC token on the same chain where Orderly has a Vault:

```
User has WETH on Arbitrum
    → SDK swaps WETH → USDC via WOOFi DEX depositor contract
    → USDC deposited into Orderly Vault
    → Single transaction (atomic)
```

### Cross-Chain Swap Deposit

If the user has tokens on a different chain:

```
User has USDC on Ethereum
    → SDK queries WOOFi cross-chain router for a quote
    → Calls crossSwap() on the router contract
    → LayerZero bridges tokens to the destination chain
    → USDC arrives at Orderly Vault on settlement chain
```

### Enabling in SDK

Swap deposits require the `enableSwapDeposit` flag on both the provider and wallet connector:

```tsx
<WalletConnectorProvider enableSwapDeposit>
  <OrderlyAppProvider enableSwapDeposit brokerId="..." networkId="mainnet">
    {/* DepositForm will show all supported tokens + chains */}
  </OrderlyAppProvider>
</WalletConnectorProvider>
```

> **Note**: Swap deposits are only available on **mainnet**. Testnet only supports direct USDC deposits.

---

## Part 11: Troubleshooting Deposits

### Common Issues

| Problem | Cause | Solution |
| :--- | :--- | :--- |
| **"TokenNotAllowed"** revert | Token hash not whitelisted | Verify `tokenHash = keccak256("USDC")` — use the token **symbol string**, not address |
| **"BrokerNotAllowed"** revert | Broker hash not whitelisted | Verify `brokerHash = keccak256("your_broker_id")` |
| **"AccountIdInvalid"** revert | accountId doesn't match `msg.sender` + `brokerHash` | Recalculate: `keccak256(abi.encode(address, solidityPackedKeccak256("string", brokerId)))` |
| **"ZeroDepositFee"** revert | `msg.value` is 0 but fee is required | Call `getDepositFee()` first and pass the result as `msg.value` |
| **"DepositExceedLimit"** revert | Amount exceeds per-token limit | Check `vault.tokenAddress2DepositLimit(tokenAddress)` |
| **Deposit confirmed but balance not updated** | Cross-chain relay pending | Check LayerZero Explorer — wait for "Delivered" status (1–20 min) |
| **USDT approval fails** | USDT requires reset to 0 first | Call `approve(vault, 0)` before `approve(vault, amount)` (Ethereum mainnet only) |
| **Insufficient native token** | Not enough ETH/SOL for deposit fee | Ensure wallet has native tokens for `msg.value` (deposit fee) + gas |
| **"BalanceNotEnough"** | Wallet doesn't have enough of the deposit token | Check `token.balanceOf(user)` before depositing |
| **Deposit not showing in Orderly** | LayerZero message lost or delayed | Search wallet on https://layerzeroscan.com/ — if "No message found", contact Orderly support |

### Debugging Checklist

1. **Is the account registered?** — `GET /v1/get_account?address={addr}&broker_id={broker}`
2. **Does the wallet have the token?** — `token.balanceOf(wallet)` ≥ deposit amount
3. **Is the Vault approved?** — `token.allowance(wallet, vaultAddress)` ≥ deposit amount
4. **Is the token allowed?** — `vault.getAllowedToken(tokenHash)` returns a valid address
5. **Is the broker allowed?** — `vault.getAllowedBroker(brokerHash)` returns `true`
6. **Is the deposit fee included?** — `vault.getDepositFee(...)` and pass as `msg.value`
7. **Did the tx succeed on-chain?** — Check source chain block explorer
8. **Is the cross-chain message delivered?** — Check LayerZero Explorer

---

## Part 12: Required Libraries

### TypeScript / JavaScript (Direct Contract Interaction)

```bash
npm install ethers
```

| Package | Purpose |
| :--- | :--- |
| `ethers` | Contract interaction, ABI encoding, keccak256, parseUnits |

### Python (Direct Contract Interaction)

```bash
pip install web3 eth-account eth-abi
```

| Package | Purpose |
| :--- | :--- |
| `web3` | Web3.py — contract interaction, keccak256 |
| `eth-account` | Wallet/key management |
| `eth-abi` | ABI encoding |

### Orderly SDK (React — Handles Everything Automatically)

```bash
npm install @orderly.network/hooks @orderly.network/react-app @orderly.network/wallet-connector @orderly.network/ui-transfer
```

| Package | Purpose |
| :--- | :--- |
| `@orderly.network/hooks` | `useDeposit` hook |
| `@orderly.network/ui-transfer` | `DepositForm`, `DepositAndWithdrawWidget` UI components |
| `@orderly.network/react-app` | App provider (handles auth + config) |
| `@orderly.network/wallet-connector` | Wallet connection (EVM + Solana) |

---

## Quick Reference: Deposit Decision Tree

```
Are you using the Orderly JS SDK?
├── YES → Use the pre-built deposit UI or useDeposit hook:
│   ├── Quickest: <DepositForm /> from @orderly.network/ui-transfer
│   ├── Custom UI: useDeposit() from @orderly.network/hooks
│   └── Both handle EVM + Solana, approval, fees, and cross-chain automatically
│
└── NO → Manual deposit via smart contract:
    │
    ├── Which chain?
    │   ├── EVM:
    │   │   1. token.approve(vaultAddress, amount)
    │   │   2. vault.getDepositFee(user, depositData) → fee in WEI
    │   │   3. vault.deposit(depositData, { value: fee })
    │   │   4. Monitor via LayerZero Explorer + GET /v1/asset/history
    │   │
    │   └── Solana:
    │       → Complex PDA derivation + LayerZero endpoint integration
    │       → Strongly recommended to use the SDK for Solana deposits
    │
    └── Token not USDC?
        ├── Swap deposit via WOOFi DEX (mainnet only)
        └── Or swap externally to USDC first, then deposit
```
