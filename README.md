NIP-XXX
=======

Chain-Agnostic Zaps
-------------------

`draft` `optional`

This NIP extends [NIP-57](https://github.com/nostr-protocol/nips/blob/master/57.md) (Lightning Zaps) to support **chain-agnostic payments** using [CAIP-358](https://standards.chainagnostic.org/CAIPs/caip-358) (`wallet_pay`) payment requests, in addition to BOLT-11 Lightning invoices.

This enables zaps settled via on-chain tokens (e.g., sBTC on Stacks, USDC on Ethereum/Solana, or any asset addressable via [CAIP-19](https://standards.chainagnostic.org/CAIPs/caip-19)) while preserving full backward compatibility with existing Lightning zaps.

## Motivation

NIP-57 zaps are limited to Lightning Network payments. This creates barriers for:

- **AI agents** that hold on-chain wallets but lack Lightning infrastructure
- **Users on non-Lightning chains** (Stacks, Ethereum, Solana) who want to zap with assets they already hold
- **sBTC holders** who want to zap with bitcoin-backed tokens without bridging back to Lightning
- **Cross-chain payments** where the sender and recipient prefer different settlement layers

CAIP-358 (`wallet_pay`) provides a chain-agnostic payment request standard that maps cleanly onto the existing zap flow. By supporting CAIP-358 alongside BOLT-11, clients and wallets can offer multi-chain zaps without breaking existing implementations.

## Specification

### New Tags

This NIP introduces the following optional tags on zap request (kind `9734`) and zap receipt (kind `9735`) events:

| Tag | Description | Required |
|-----|-------------|----------|
| `payment_type` | `"bolt11"` (default) or `"caip358"` | Optional on 9734, REQUIRED on 9735 if non-Lightning |
| `caip358_request` | JSON-encoded CAIP-358 `wallet_pay` request params | On 9734 when `payment_type` = `"caip358"` |
| `caip358_receipt` | JSON-encoded CAIP-358 `wallet_pay` response result | On 9735 when `payment_type` = `"caip358"` |
| `network` | [CAIP-2](https://standards.chainagnostic.org/CAIPs/caip-2) chain ID (e.g., `"stacks:1"`, `"eip155:1"`) | On 9735 when `payment_type` = `"caip358"` |
| `txid` | On-chain transaction ID (for PUSH payments) | On 9735 when settlement is on-chain |
| `asset` | [CAIP-19](https://standards.chainagnostic.org/CAIPs/caip-19) asset ID | On 9735 when `payment_type` = `"caip358"` |

### Protocol Flow (CAIP-358 Zaps)

The flow mirrors NIP-57 but replaces LNURL/BOLT-11 with CAIP-358:

1. Client reads the recipient's **zap endpoint**. If the endpoint advertises `caip358` support (via a `caip358` field in the lnurl-pay or `.well-known` response), the client MAY use CAIP-358.

2. Client creates a **zap request** (kind `9734`) with:
   - Standard NIP-57 tags (`p`, `relays`, `amount`, optionally `e`, `a`)
   - `["payment_type", "caip358"]`
   - `["caip358_request", "<JSON>"]` containing a CAIP-358 `wallet_pay` request

3. Client sends the zap request to the recipient's zap endpoint (same as NIP-57 Appendix B).

4. The zap server validates the request and returns **payment options** (CAIP-358 `paymentOptions` array) instead of a BOLT-11 invoice.

5. The client (or wallet) selects a payment option, executes the payment, and receives a **transfer receipt** (CAIP-358 `TransferReceipt`).

6. The client sends the receipt back to the zap server.

7. The zap server **verifies the payment on-chain** (for PUSH payments) or **settles the payment** (for PULL payments), then publishes a **zap receipt** (kind `9735`) with the CAIP-358 receipt data.

### Zap Request (kind 9734) — CAIP-358

Example zap request for 100 sats sBTC on Stacks mainnet:

```json
{
  "kind": 9734,
  "content": "sBTC zap!",
  "tags": [
    ["relays", "wss://relay.damus.io", "wss://nos.lol"],
    ["amount", "100"],
    ["p", "04c915daefee38317fa734444acee390a8269fe5810b2241e5e6dd343dfbecc9"],
    ["e", "9ae37aa68f48645127299e9453eb5d908a0cbb6058ff340d528ed4d37c8994fb"],
    ["payment_type", "caip358"],
    ["caip358_request", "{\"version\":1,\"orderId\":\"zap-9ae37aa6-1740000000\",\"expiry\":1740000300,\"paymentOptions\":[{\"asset\":\"stacks:1/sip10:SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token\",\"amount\":\"0x64\",\"recipient\":\"SP1JBH94STS4MHD61H3HA1ZN2R4G41EZGFG9SXP66\",\"types\":[\"sip10-transfer\"]}]}"]
  ],
  "pubkey": "2b4603d231d15f771ded3e6c1ee250d79bd9a8950dbaf2e76015d5bb5c65e198",
  "created_at": 1740000000
}
```

### Zap Receipt (kind 9735) — CAIP-358

Example zap receipt after on-chain sBTC payment:

```json
{
  "kind": 9735,
  "content": "",
  "tags": [
    ["p", "04c915daefee38317fa734444acee390a8269fe5810b2241e5e6dd343dfbecc9"],
    ["e", "9ae37aa68f48645127299e9453eb5d908a0cbb6058ff340d528ed4d37c8994fb"],
    ["bolt11", ""],
    ["description", "<JSON-encoded kind 9734 zap request>"],
    ["payment_type", "caip358"],
    ["network", "stacks:1"],
    ["txid", "5e2422845c5e0908e3ae2d5a424b19a625d602cf82b1f781f6e97e21abbb1f27"],
    ["asset", "stacks:1/sip10:SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token"],
    ["caip358_receipt", "{\"version\":1,\"orderId\":\"zap-9ae37aa6-1740000000\",\"payment\":{\"asset\":\"stacks:1/sip10:SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token\",\"amount\":\"0x64\",\"recipient\":\"SP1JBH94STS4MHD61H3HA1ZN2R4G41EZGFG9SXP66\",\"types\":[\"sip10-transfer\"]},\"receipt\":{\"type\":\"sip10-transfer\",\"hash\":\"5e2422845c5e0908e3ae2d5a424b19a625d602cf82b1f781f6e97e21abbb1f27\",\"data\":{\"from\":\"SP16H0KE0BPR4XNQ64115V5Y1V3XTPGMWG5YPC9TR\",\"to\":\"SP1JBH94STS4MHD61H3HA1ZN2R4G41EZGFG9SXP66\",\"value\":\"0x64\"}}}"]
  ],
  "pubkey": "<zap_server_pubkey>",
  "created_at": 1740000030
}
```

### CAIP-358 Extensions for Stacks

CAIP-358 v1 defines transfer types for EVM and Solana. This NIP proposes the following additional types for Stacks:

| Transfer Type | Description |
|---------------|-------------|
| `sip10-transfer` | SIP-010 fungible token transfer (PUSH payment). Equivalent to `erc20-transfer` for Stacks. |
| `stx-transfer` | Native STX transfer (PUSH payment). Equivalent to `native-transfer`. |

### CAIP-19 Asset IDs for Stacks

Following [CAIP-19](https://standards.chainagnostic.org/CAIPs/caip-19) conventions:

| Asset | CAIP-19 ID |
|-------|-----------|
| STX (native) | `stacks:1/slip44:5757` |
| sBTC | `stacks:1/sip10:SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token` |

The `sip10` namespace is proposed for SIP-010 fungible tokens on Stacks, analogous to `erc20` for ERC-20 tokens.

## Backward Compatibility

This NIP is **fully backward compatible** with NIP-57:

- If `payment_type` is absent, clients MUST assume `"bolt11"` (standard Lightning zap).
- Existing zap receipts without `payment_type` continue to work unchanged.
- Clients that don't support CAIP-358 simply ignore the new tags.
- The `bolt11` tag is retained (set to empty string `""`) on CAIP-358 zap receipts for compatibility with clients that expect it.

## Verification

### For Lightning Zaps (unchanged)
Follow NIP-57 Appendix F.

### For CAIP-358 Zaps

Clients MUST verify CAIP-358 zap receipts by:

1. The `zap receipt` event MUST have a valid signature from the recipient's zap server pubkey.
2. The `description` tag MUST contain a valid kind `9734` zap request.
3. The `caip358_receipt` MUST contain a valid CAIP-358 response with matching `orderId`.
4. For PUSH payments (e.g., `sip10-transfer`, `erc20-transfer`):
   - The `txid` MUST reference a confirmed transaction on the specified `network`.
   - The transaction MUST transfer the correct `asset`, `amount`, and `recipient` as specified in the zap request's `caip358_request`.
   - The transaction sender SHOULD match the zap request's `pubkey` (but MAY differ for sponsored transactions).
5. For PULL payments (e.g., `erc20-approve`, `erc2612-permit`):
   - The `hash` in the receipt MUST be a valid signature/approval.
   - The zap server MUST have settled the payment before publishing the receipt.

## Security Considerations

- **On-chain verification**: Unlike Lightning (where only the zap server can verify payment), CAIP-358 PUSH payments are publicly verifiable on-chain. This is a privacy tradeoff but a security improvement.
- **Double-spend**: Zap servers MUST verify transaction finality before publishing receipts. For Stacks, this means waiting for the transaction to be included in an anchored block (~30 seconds).
- **Replay protection**: The `orderId` field in CAIP-358 prevents replay attacks. Zap servers MUST reject duplicate `orderId` values.
- **Asset validation**: Clients MUST verify that the `asset` in the receipt matches one of the `paymentOptions` in the original request.

## Multi-Chain Payment Options

A key advantage of CAIP-358 is offering **multiple payment options** in a single request. A zap server can advertise acceptance of sBTC, USDC, STX, or Lightning — and the wallet picks the best option:

```json
{
  "paymentOptions": [
    {
      "asset": "stacks:1/sip10:SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token",
      "amount": "0x64",
      "recipient": "SP1JBH94STS4MHD61H3HA1ZN2R4G41EZGFG9SXP66",
      "types": ["sip10-transfer"]
    },
    {
      "asset": "stacks:1/slip44:5757",
      "amount": "0x1312D00",
      "recipient": "SP1JBH94STS4MHD61H3HA1ZN2R4G41EZGFG9SXP66",
      "types": ["stx-transfer"]
    },
    {
      "asset": "eip155:1/erc20:0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
      "amount": "0x5F5E100",
      "recipient": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
      "types": ["erc20-transfer"]
    }
  ]
}
```

## References

- [NIP-57](https://github.com/nostr-protocol/nips/blob/master/57.md) — Lightning Zaps
- [CAIP-358](https://standards.chainagnostic.org/CAIPs/caip-358) — Universal Payment Request Method
- [CAIP-2](https://standards.chainagnostic.org/CAIPs/caip-2) — Blockchain ID Specification
- [CAIP-19](https://standards.chainagnostic.org/CAIPs/caip-19) — Asset Type and Asset ID Specification
- [SIP-010](https://github.com/stacksgov/sips/blob/main/sips/sip-010/sip-010-fungible-token-standard.md) — Stacks Fungible Token Standard
- [SIP-029](https://github.com/stacksgov/sips/pull/202) — Stacks Pay Payment Request Standard
- [BOLT-11](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md) — Lightning Invoice Protocol

## Copyright

This NIP is placed in the public domain.

## Real-World Example: sBTC Zap on Stacks Mainnet

The following is a real sBTC zap performed on February 20, 2026 by cocoa007 (npub19drq8533690hw80d8ekpacjs67dan2y4pka09emqzh2mkhr9uxvqd4k3nn) to Stark Comet on Stacks mainnet.

### On-Chain Transactions

Two sBTC transfers of 100 sats each were sent to `SP1JBH94STS4MHD61H3HA1ZN2R4G41EZGFG9SXP66`:

- **tx1**: [`5e2422845c5e0908e3ae2d5a424b19a625d602cf82b1f781f6e97e21abbb1f27`](https://explorer.hiro.so/txid/5e2422845c5e0908e3ae2d5a424b19a625d602cf82b1f781f6e97e21abbb1f27?chain=mainnet)
- **tx2**: [`6a3dd5787dd1f828c5ffe42bda90f086d7d771f42d9af6c0c16baded2e3f41b0`](https://explorer.hiro.so/txid/6a3dd5787dd1f828c5ffe42bda90f086d7d771f42d9af6c0c16baded2e3f41b0?chain=mainnet)

### Nostr Zap Proof (kind 1 — current)

Published as event [`59a3588f3f9dca87dac921aaaab651ed5fcc7f829c36b2ba6f20f6ac8837fae4`](https://njump.me/59a3588f3f9dca87dac921aaaab651ed5fcc7f829c36b2ba6f20f6ac8837fae4):

```json
{
  "kind": 1,
  "pubkey": "2b4603d231d15f771ded3e6c1ee250d79bd9a8950dbaf2e76015d5bb5c65e198",
  "created_at": 1740038508,
  "tags": [
    ["t", "bitcoin"],
    ["t", "sbtc"],
    ["t", "stacks"],
    ["t", "zap"],
    ["t", "x402"],
    ["r", "https://explorer.hiro.so/txid/5e2422845c5e0908e3ae2d5a424b19a625d602cf82b1f781f6e97e21abbb1f27?chain=mainnet"],
    ["r", "https://explorer.hiro.so/txid/6a3dd5787dd1f828c5ffe42bda90f086d7d771f42d9af6c0c16baded2e3f41b0?chain=mainnet"],
    ["r", "https://github.com/cocoa007/x402-nostr-relay/issues/3"]
  ],
  "content": "⚡ sBTC zap to Stark Comet\n\nSent 200 sats sBTC (2x 100 sats) on Stacks mainnet — bitcoin-backed micropayments, no Lightning needed.\n\ntx1: https://explorer.hiro.so/txid/5e2422845c5e0908e3ae2d5a424b19a625d602cf82b1f781f6e97e21abbb1f27?chain=mainnet\ntx2: https://explorer.hiro.so/txid/6a3dd5787dd1f828c5ffe42bda90f086d7d771f42d9af6c0c16baded2e3f41b0?chain=mainnet\n\nRecipient: SP1JBH94STS4MHD61H3HA1ZN2R4G41EZGFG9SXP66 (Stark Comet)\nToken: SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token\n\nThis is what sBTC zaps could look like — on-chain verifiable bitcoin payments instead of Lightning invoices. Anyone can verify the txids on Stacks explorer.\n\nSpec for Nostr sBTC zaps: https://github.com/cocoa007/x402-nostr-relay/issues/3\n\n#bitcoin #sbtc #stacks #nostr #zap #x402",
  "id": "59a3588f3f9dca87dac921aaaab651ed5fcc7f829c36b2ba6f20f6ac8837fae4"
}
```

### How This Would Look With NIP-XXX (kind 9734 + 9735)

**Zap Request (kind 9734)** — sent by cocoa007 to the zap server:

```json
{
  "kind": 9734,
  "pubkey": "2b4603d231d15f771ded3e6c1ee250d79bd9a8950dbaf2e76015d5bb5c65e198",
  "created_at": 1740038500,
  "content": "sBTC zap for collab on portfolio risk scorer",
  "tags": [
    ["relays", "wss://relay.damus.io", "wss://nos.lol"],
    ["amount", "200"],
    ["p", "<stark_comet_nostr_pubkey>"],
    ["payment_type", "caip358"],
    ["caip358_request", "{\"version\":1,\"orderId\":\"zap-cocoa007-starkcomet-1740038500\",\"expiry\":1740038800,\"paymentOptions\":[{\"asset\":\"stacks:1/sip10:SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token\",\"amount\":\"0xC8\",\"recipient\":\"SP1JBH94STS4MHD61H3HA1ZN2R4G41EZGFG9SXP66\",\"types\":[\"sip10-transfer\"]}]}"]
  ]
}
```

**Zap Receipt (kind 9735)** — published by the zap server after verifying on-chain payment:

```json
{
  "kind": 9735,
  "pubkey": "<zap_server_nostr_pubkey>",
  "created_at": 1740038540,
  "content": "",
  "tags": [
    ["p", "<stark_comet_nostr_pubkey>"],
    ["P", "2b4603d231d15f771ded3e6c1ee250d79bd9a8950dbaf2e76015d5bb5c65e198"],
    ["bolt11", ""],
    ["description", "<JSON of the kind 9734 zap request above>"],
    ["payment_type", "caip358"],
    ["network", "stacks:1"],
    ["txid", "5e2422845c5e0908e3ae2d5a424b19a625d602cf82b1f781f6e97e21abbb1f27"],
    ["txid", "6a3dd5787dd1f828c5ffe42bda90f086d7d771f42d9af6c0c16baded2e3f41b0"],
    ["asset", "stacks:1/sip10:SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token"],
    ["caip358_receipt", "{\"version\":1,\"orderId\":\"zap-cocoa007-starkcomet-1740038500\",\"payment\":{\"asset\":\"stacks:1/sip10:SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token\",\"amount\":\"0xC8\",\"recipient\":\"SP1JBH94STS4MHD61H3HA1ZN2R4G41EZGFG9SXP66\",\"types\":[\"sip10-transfer\"]},\"receipt\":{\"type\":\"sip10-transfer\",\"hash\":\"5e2422845c5e0908e3ae2d5a424b19a625d602cf82b1f781f6e97e21abbb1f27\",\"data\":{\"from\":\"SP16H0KE0BPR4XNQ64115V5Y1V3XTPGMWG5YPC9TR\",\"to\":\"SP1JBH94STS4MHD61H3HA1ZN2R4G41EZGFG9SXP66\",\"value\":\"0xC8\"}}}"]
  ]
}
```

**Verification steps for any client:**

1. Decode the `description` tag → get the original kind 9734 zap request
2. Read `txid` tags → [`5e24228...`](https://explorer.hiro.so/txid/5e2422845c5e0908e3ae2d5a424b19a625d602cf82b1f781f6e97e21abbb1f27?chain=mainnet), [`6a3dd57...`](https://explorer.hiro.so/txid/6a3dd5787dd1f828c5ffe42bda90f086d7d771f42d9af6c0c16baded2e3f41b0?chain=mainnet)
3. Query Stacks API: `GET https://api.hiro.so/extended/v1/tx/{txid}` for each
4. Verify: sender = `SP16H0KE…` (cocoa007), recipient = `SP1JBH94…` (Stark Comet), amount = 100 sats sBTC each
5. Confirm txs are in anchored blocks (finalized)
