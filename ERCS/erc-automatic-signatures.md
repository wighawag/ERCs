---
title: Automatic Signature Request (`wallet_autoSign`)
author: Ronan Sandford (@wighawag)
discussions-to: https://ethereum-magicians.org/t/automatic-authentication-signature/2429
status: Draft
type: Standards Track
category: ERC
created: 2019-05-15
requires: 1102, 191
---

## Simple Summary
Applications with backend components often need to authenticate wallet owners using their private keys. Currently, wallets provide applications a way to fetch user addresses, but applications cannot verify that users actually own the corresponding private keys. With `wallet_autoSign`, applications can request signatures for specific payloads to ensure users have access to the private keys behind their Ethereum addresses. This can also be used to communicate authentically with a backend without session tracking.

## Abstract
Applications can request wallets to sign specific payloads on behalf of users without requiring explicit user confirmation for each signature.

## Motivation
Currently, applications with backend components that require Ethereum address authentication ask users to sign messages. In some cases, these messages are static (perhaps for better appearance), making them vulnerable to replay attacks.

Such user interaction is less than ideal and technically unnecessary. By restricting the type of payload that automatic signatures can sign, these requests can be processed behind the scenes without involving the user. There is no risk for the user as the payload is prepended with a unique string that prevents misuse.

## Specification
The JSON RPC method `wallet_autoSign` requires 2 parameters:

### Required Parameters
- `account` (string): the Ethereum address expected to sign the payload
- `payload` (string): any string the application wants to authenticate

The wallet must prepend "Automatic Signature" to the payload before signing via [EIP-191](https://eips.ethereum.org/EIPS/eip-191)to prevent the application from requesting signatures for other purposes.

### Example
A JSON-RPC request to authenticate via a specific challenge:

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "wallet_autoSign",
  "params": ["0x50545c559a4110c2a1216976b3667b3208a71c0e", "any string"]
}
```

This would actually sign the string "Automatic Signature: any string" with "0x50545c559a4110c2a1216976b3667b3208a71c0e"'s private key.

This requires that [EIP-1102]'s `eth_requestAccounts` was approved by the user first, to preserve privacy.

## Rationale
Several design considerations informed the current specification:

1. **Signature Prefix**: The decision to prepend "Automatic Signature" to all payloads ensures that applications cannot trick users into signing arbitrary content. This prefix distinguishes automatic signatures from other types of signatures and helps prevent misuse.

2. **Signature Mechanism**: We considered using a non-hardened derived key for signing instead of the primary private key. This approach could add an extra layer of security by isolating the automatic signing capability from the main account keys. However, for simplicity and compatibility with existing wallet implementations, the current specification uses the primary private key.

3. **Authentication Flow**: The requirement that [EIP-1102]'s `eth_requestAccounts` must be approved first preserves user privacy by ensuring users have already consented to sharing their address with the application.

4. **Rate Limiting**: While not mandated in the specification, implementers are encouraged to consider rate limiting requests to prevent potential misuse or denial-of-service attacks.

Further discussion on these design decisions can be found in the referenced discussions-to link.

## Implementation
Below is a sample implementation

```javascript
/**
 * Implementation of wallet_autoSign using EIP-712 style signing
 * @param {string} account - Ethereum address to sign with
 * @param {string} payload - Application-provided payload to sign
 * @returns {Promise<string>} - Signature
 */
async function wallet_autoSign(account, payload) {
  // Verify account is valid and controlled by this wallet
  if (!isAccountOwned(account)) {
    throw new Error('Requested account not found');
  }

  // Verify eth_requestAccounts was previously approved for this domain
  if (!hasApprovedAccounts()) {
    throw new Error('eth_requestAccounts must be approved first');
  }

  // Prepend the required prefix
  const messageToSign = `Automatic Signature: ${payload}`;

  // Convert to hex encoded UTF-8 message for signing
  const messageHex = '0x' + Buffer.from(messageToSign, 'utf8').toString('hex');

  // Create the Ethereum-specific message format
  // The "\x19Ethereum Signed Message:\n" prefix is added to prevent signing raw transaction data
  const ethMessage = ethUtil.hashPersonalMessage(ethUtil.toBuffer(messageHex));

  // Sign the message with the private key corresponding to the account
  const privateKey = getPrivateKeyForAccount(account);
  const signature = ethUtil.ecsign(ethMessage, privateKey);

  // Convert to the standard signature format
  const signatureHex = ethUtil.toRpcSig(signature.v, signature.r, signature.s);

  // Rate limiting logic could be implemented here
  incrementRateLimitCounter(account, getDomain());

  return signatureHex;
}
```


## Test Cases
Basic test scenarios for wallet implementations:

1. **Basic Functionality Test**:
   - Request `wallet_autoSign` with a valid Ethereum address and payload
   - Verify the returned signature corresponds to "Automatic Signature: [payload]" signed by the specified address

2. **Privacy Protection Test**:
   - Attempt to call `wallet_autoSign` without prior approval of `eth_requestAccounts`
   - Verify the wallet rejects the request

3. **Address Mismatch Test**:
   - Request `wallet_autoSign` with an address that doesn't match any account controlled by the wallet
   - Verify the wallet rejects the request

4. **Signature Validation Test**:
   - Generate a signature using `wallet_autoSign`
   - Verify that the signature can be correctly recovered using standard Ethereum signature recovery methods
   - Confirm that the recovered address matches the requested signing address

## Backwards Compatibility
No backwards compatibility issues have been identified. This proposal introduces a new method and does not modify the behavior of existing JSON-RPC methods. Wallets that do not implement this method would simply reject the `wallet_autoSign` request, requiring applications to fall back to manual signature requests.

## Security Considerations
Implementing this EIP raises several security considerations:

1. **Automatic Signing Risks**: Allowing applications to request signatures without user confirmation introduces potential risks. The required prefix "Automatic Signature" mitigates this by ensuring applications cannot trick users into signing arbitrary data that could be used for other purposes.

2. **Replay Attack Protection**: Applications should incorporate additional measures such as timestamps, nonces, or application-specific identifiers in their payloads to prevent replay attacks across different contexts.

3. **Rate Limiting**: Wallet implementations should consider implementing rate limiting to prevent abuse of the automatic signing capability.

4. **Scope Limitation**: The scope of what can be automatically signed is deliberately limited to authentication purposes only. Any extension of this capability should be carefully considered.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

[EIP-191]: https://eips.ethereum.org/EIPS/eip-191
[EIP-1102]: https://eips.ethereum.org/EIPS/eip-1102
