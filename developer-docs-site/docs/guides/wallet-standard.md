---
title: "Adhere to Wallet Standard"
slug: "wallet-standard"
---
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

# Aptos Wallet Standard

The wallet standard provides guidelines to have interoperability between multiple wallets.  This ensures that dapp developers do not need to change
their applications to handle different wallets.  A single interface for all of them together, allowing easy additions of new wallets and more users
to each application.  This allows users to choose which wallet they want without worrying about whether apps support them.

In order to allow for Aptos' wallet interoperability, the following is required:
1. Mnemonics - a set of words that can derive account private keys
2. dApp API - entry points into the wallet to support access to identity managed by the wallet
3. Key rotation - handling both the relationship around mnemonics and the recovery of accounts in different wallets

## Mnemonics Phrases

A mnemonic phrase is a multiple word phrase that can be used to generate account addresses.
We recommend 1 mnemonic per account, in order to handle key rotation better.  For example, [Petra wallet](../guides/install-petra-wallet.md) uses a 1 to 1 mnemonic to account system.
However, some wallets may want to support 1 mnemonic to many accounts coming from other chains. To support both of these use cases the Aptos wallet standard uses a [BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki) derive path for mnemonics to accounts.

### Creating an Aptos Account

Aptos account creation can be supported across wallets in the following manner:

1. Generate a mnemonic phrase, for example with BIP39
2. Get the master seed from that mnemonic phrase
3. Use the BIP44 derive path to retrieve an account address (e.g. `m/44'/637'/0'/0'/0'`)
    - See the [Aptos typescript SDK's implementation for the derive path](https://github.com/aptos-labs/aptos-core/blob/1bc5fd1f5eeaebd2ef291ac741c0f5d6f75ddaef/ecosystem/typescript/sdk/src/aptos_account.ts#L49-L69))
    - For example Petra Wallet, always uses the path `m/44'/637'/0'/0'/0'` since there is 1 mnemonic per 1 account


```typescript
/**
   * Creates new account with bip44 path and mnemonics,
   * @param path. (e.g. m/44'/637'/0'/0'/0')
   * Detailed description: {@link https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki}
   * @param mnemonics.
   * @returns AptosAccount
   */
  static fromDerivePath(path: string, mnemonics: string): AptosAccount {
    if (!AptosAccount.isValidPath(path)) {
      throw new Error("Invalid derivation path");
    }

    const normalizeMnemonics = mnemonics
      .trim()
      .split(/\s+/)
      .map((part) => part.toLowerCase())
      .join(" ");

    const { key } = derivePath(path, bytesToHex(bip39.mnemonicToSeedSync(normalizeMnemonics)));

    return new AptosAccount(new Uint8Array(key));
  }
```

### Supporting 1 Mnemonic per Multiple Account Wallets

This is not recommended because the 1 mnemonic to many accounts paradigm makes it harder to handle rotated keys (mnemonic changes for one account but not others).
However, many wallets from other ecosystems use this paradigm, and take these steps to generate accounts

1. Generate a mnemonic phrase, for example with BIP39
2. Get the master seed from that mnemonic phrase
4. Use the BIP44 derive path to retrieve private keys (e.g. `m/44'/637'/i'/0'/0'`) where `i` is the account index
    - See the [Aptos typescript SDK's implementation for the derive path](https://github.com/aptos-labs/aptos-core/blob/1bc5fd1f5eeaebd2ef291ac741c0f5d6f75ddaef/ecosystem/typescript/sdk/src/aptos_account.ts#L49-L69))
6. Increase `i` until all of the accounts the user wants to import are found
    - Note: The iteration should be limited, if an account doesn't exist during iteration, keep iterating for a constant `address_gap_limit` (10 for now) to see if there are any other accounts. If an account is found we will continue to iterate as normal.

ie.
```typescript
const gapLimit = 10;
let currentGap = 0;

for (let i = 0; currentGap < gapLimit; i += 1) {
    const derivationPath = `m/44'/637'/${i}'/0'/0'`;
    const account = fromDerivePath(derivationPath, mnemonic);
    const response = account.getResources();
    if (response.status !== 404) {
        wallet.addAccount(account);
        currentGap = 0;
    } else {
        currentGap += 1;
    }
}
```

## dApp API

More important than account creation, is how wallets connect to dapps.  Additionally, following these APIs will allow for the wallet developer to integrate with the [Aptos Wallet adapter standard](../concepts/wallet-adapter-concept.md).  The APIs are as follows:

- `connect()`, `disconnect()`
- `account()`
- `network()`
- `signAndSubmitTransaction(transaction: EntryFunctionPayload)`
- `signMessage(payload: SignMessagePayload)`
- Event listening (`onAccountChanged(listener)`, `onNetworkChanged(listener)`)

```typescript
// Common Args and Responses

// For single-signer account, there is one publicKey and minKeysRequired is null.
// For multi-signer account, there are multiple publicKeys and minKeysRequired value.
type AccountInfo {
    address: string;
    publicKey: string | string[];
    minKeysRequired?: number; // for multi-signer account
}

type NetworkInfo = {
  name: string;
  chainId: string;
  url: string;
};

// The important thing to return here is the transaction hash the dApp can wait for it
type [PendingTransaction](https://github.com/aptos-labs/aptos-core/blob/1bc5fd1f5eeaebd2ef291ac741c0f5d6f75ddaef/ecosystem/typescript/sdk/src/generated/models/PendingTransaction.ts)

type [EntryFunctionPayload](https://github.com/aptos-labs/aptos-core/blob/1bc5fd1f5eeaebd2ef291ac741c0f5d6f75ddaef/ecosystem/typescript/sdk/src/generated/models/EntryFunctionPayload.ts)


```

### Connection APIs

The connection APIs ensure that wallets don't accept requests until the user acknowledges that they want to see the requests.  This keeps
the user state clean, and prevents the user from unknowingly having prompts.

- `connect()` will prompt the user for a connection
    - return `Promise<AccountInfo>`
- `disconnect()` allows the user to stop giving access to a dApp and also helps the dApp with state management
    - return `Promise<void>`

### State APIs
#### Get Account
**Connection required**

Allows a dapp to query for the current connected account address and public key

- `account()` no prompt to the user
    - returns `Promise<AccountInfo>`

#### Get Network
**Connection required**

Allows a dapp to query for the current connected network name, chain id, and url

- `network()` no prompt to the user
    - returns `Promise<NetworkInfo>`

### Signing APIs
#### Sign and submit transaction
**Connection required**

Allows a dapp to send a simple JSON payload using the [SDK](https://github.com/aptos-labs/aptos-core/blob/1bc5fd1f5eeaebd2ef291ac741c0f5d6f75ddaef/ecosystem/typescript/sdk/src/aptos_client.ts#L217-L221)
for signing and submission to the current network.  The user should be prompted for approval.

- `signAndSubmitTransaction(transaction: EntryFunctionPayload)` will prompt the user with the transaction they are signing
    - returns `Promise<PendingTransaction>`

#### Sign message
**Connection required**

Allows a dapp to sign a message with their private key.  The most common use case is to verify identity, but there are a few other possible use
cases.  The user should be prompted for approval You may notice some wallets from other chains just provide an interface to sign arbitrary strings. This can be susceptible to man-in-the-middle attacks, signing string transactions, etc.

Types:
```typescript
export interface SignMessagePayload {
  address?: boolean; // Should we include the address of the account in the message
  application?: boolean; // Should we include the domain of the dApp
  chainId?: boolean; // Should we include the current chain id the wallet is connected to
  message: string; // The message to be signed and displayed to the user
  nonce: string; // A nonce the dApp should generate
}

export interface SignMessageResponse {
  address?: string;
  application?: string;
  chainId?: number;
  fullMessage: string; // The message that was generated to sign
  message: string; // The message passed in by the user
  nonce: string,
  prefix: string, // Should always be APTOS
  signature: string | string[]; // The signed full message
  bitmap?: Uint8Array; // a 4-byte (32 bits) bit-vector of length N
}
```

- `signMessage(payload: SignMessagePayload)` prompts the user with the `payload.message` to be signed
    - returns `Promise<SignMessageResponse>`

An example:
`signMessage({nonce: 1234034, message: "Welcome to dApp!", address: true, application: true, chainId: true })`

This would generate the `fullMessage` to be signed and returned as the `signature`:
```yaml
APTOS
address: 0x000001
chain_id: 7
application: badsite.firebase.google.com
nonce: 1234034
message: Welcome to dApp!
```

Aptos has support for both single-signer, and multi-signer accounts. If the wallet is single-signer account, there is exactly one signature and `bitmap` is null. If the wallet is multi-signers account, there are multiple `signature` and `bitmap` values. `bitmap` masks which public key has signed the message.

### Event listening

To be added in the future:
- Event listening (`onAccountChanged(listener)`, `onNetworkChanged(listener)`)

## Key Rotation

Key rotation is currently not implemented in any wallets. Mapping of rotated keys has been [implemented](https://github.com/aptos-labs/aptos-core/pull/2972) but SDK integration is in progress.

Wallets that import a private key will have to do the following:
1. Derive the authentication key
2. Lookup the authentication key onchain in the Account origination table
  - If the account doesn't exist, it's a new account.  The address to be used is the authentication key
  - If the account does exist, it's a rotated key account, and the address to be used will come from the table.

## Appendix
- **[Forum post with discussion](https://forum.aptoslabs.com/t/wallet-dapp-api-standards/11765/33)** about the dapp API
