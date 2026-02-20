NIP-XXX
=======

Chain-Agnostic Zaps
-------------------

`draft` `optional`

This NIP extends [NIP-57](https://github.com/nostr-protocol/nips/blob/master/57.md) (Lightning Zaps) to add support for [CAIP-19](https://standards.chainagnostic.org/CAIPs/caip-19) asset payments via [CAIP-358](https://standards.chainagnostic.org/CAIPs/caip-358) (`wallet_pay`), enabling zaps with wrapped BTC and other on-chain tokens alongside existing BOLT-11 Lightning invoices.

## Motivation

NIP-57 zaps work well for Lightning payments, but a growing number of users and agents hold wrapped BTC (such as sBTC on Stacks) and other on-chain tokens. These holders currently cannot zap without first bridging back to Lightning.

By adding [CAIP-358](https://standards.chainagnostic.org/CAIPs/caip-358) support, clients can offer zaps settled with any [CAIP-19](https://standards.chainagnostic.org/CAIPs/caip-19)-addressable asset — including wrapped BTC — while preserving full backward compatibility with Lightning zaps.

Key use cases:
- **sBTC holders** zapping with bitcoin-backed tokens on Stacks
- **AI agents** with on-chain wallets that lack Lightning infrastructure
- **Multi-chain recipients** accepting payments across different settlement layers

## Specification

### New Tags

The following optional tags are added to zap request (kind `9734`) and zap receipt (kind `9735`) events:

| Tag | Description | Context |
|-----|-------------|---------|
| `payment_type` | `"bolt11"` (default) or `"caip358"` | Optional on 9734; REQUIRED on 9735 if non-Lightning |
| `caip358_request` | JSON-encoded CAIP-358 `wallet_pay` request params | On 9734 when `payment_type` = `"caip358"` |
| `caip358_receipt` | JSON-encoded CAIP-358 `wallet_pay` response result | On 9735 when `payment_type` = `"caip358"` |
| `network` | [CAIP-2](https://standards.chainagnostic.org/CAIPs/caip-2) chain ID (e.g., `"stacks:1"`, `"eip155:1"`) | On 9735 when `payment_type` = `"caip358"` |
| `txid` | On-chain transaction ID (for PUSH payments) | On 9735 when settlement is on-chain |
| `asset` | [CAIP-19](https://standards.chainagnostic.org/CAIPs/caip-19) asset ID | On 9735 when `payment_type` = `"caip358"` |

### Protocol Flow (CAIP-358 Zaps)

The flow mirrors NIP-57 but adds CAIP-358 as an alternative payment method:

1. Client reads the recipient's **zap endpoint**. If the endpoint advertises `caip358` support (via a `caip358` field in the lnurl-pay or `.well-known` response), the client MAY offer CAIP-358 payment options in addition to Lightning.

2. Client creates a **zap request** (kind `9734`) with:
   - Standard NIP-57 tags (`p`, `relays`, `amount`, optionally `e`, `a`)
   - `["payment_type", "caip358"]`
   - `["caip358_request", "<JSON>"]` containing a CAIP-358 `wallet_pay` request

3. Client sends the zap request to the recipient's zap endpoint (same as NIP-57 Appendix B).

4. The zap server validates the request and returns **payment options** (CAIP-358 `paymentOptions` array).

5. The client (or wallet) selects a payment option, executes the payment, and receives a **transfer receipt** (CAIP-358 `TransferReceipt`).

6. The client sends the receipt back to the zap server.

7. The zap server **verifies the payment on-chain**, then publishes a **zap receipt** (kind `9735`) with the CAIP-358 receipt data.

### Zap Request (kind 9734) — CAIP-358

Example zap request for 100 sats sBTC on Stacks:

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
    ["caip358_request", "{\"version\":1,\"orderId\":\"zap-9ae37aa6-1740000000\",\"expiry\":1740000300,\"paymentOptions\":[{\"asset\":\"stacks:1/SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token.sbtc-token\",\"amount\":\"0x64\",\"recipient\":\"SP1JBH94STS4MHD61H3HA1ZN2R4G41EZGFG9SXP66\",\"types\":[\"sip10-transfer\"]}]}"]
  ],
  "pubkey": "2b4603d231d15f771ded3e6c1ee250d79bd9a8950dbaf2e76015d5bb5c65e198",
  "created_at": 1740000000
}
```

Note: The `asset` field uses the [CAIP-19 format for Stacks tokens](https://github.com/ChainAgnostic/namespaces/pull/167) as `{caip2_chain_id}/{address}.{contract}.{token-name}`.

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
    ["asset", "stacks:1/SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token.sbtc-token"],
    ["caip358_receipt", "{\"version\":1,\"orderId\":\"zap-9ae37aa6-1740000000\",\"payment\":{\"asset\":\"stacks:1/SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token.sbtc-token\",\"amount\":\"0x64\",\"recipient\":\"SP1JBH94STS4MHD61H3HA1ZN2R4G41EZGFG9SXP66\",\"types\":[\"sip10-transfer\"]},\"receipt\":{\"type\":\"sip10-transfer\",\"hash\":\"5e2422845c5e0908e3ae2d5a424b19a625d602cf82b1f781f6e97e21abbb1f27\",\"data\":{\"from\":\"SP16H0KE0BPR4XNQ64115V5Y1V3XTPGMWG5YPC9TR\",\"to\":\"SP1JBH94STS4MHD61H3HA1ZN2R4G41EZGFG9SXP66\",\"value\":\"0x64\"}}}"]
  ],
  "pubkey": "<zap_server_pubkey>",
  "created_at": 1740000030
}
```

### CAIP-358 Transfer Types

CAIP-358 v1 defines transfer types for EVM and Solana. For other chains, the following types are used following the same convention:

| Transfer Type | Description |
|---------------|-------------|
| `sip10-transfer` | SIP-010 fungible token transfer on Stacks (PUSH payment) |
| `stx-transfer` | Native STX transfer on Stacks (PUSH payment) |

Chains with existing CAIP-19 namespace definitions (see [ChainAgnostic/namespaces](https://github.com/ChainAgnostic/namespaces)) can define additional transfer types following the same pattern.

## Backward Compatibility

This NIP is **additive** — it does not modify the existing Lightning zap flow:

- If `payment_type` is absent, clients MUST assume `"bolt11"` (standard Lightning zap).
- Existing zap receipts without `payment_type` continue to work unchanged.
- Clients that don't support CAIP-358 simply ignore the new tags.
- The `bolt11` tag is retained (set to empty string `""`) on CAIP-358 zap receipts for compatibility with clients that require it.

## Verification

### For Lightning Zaps (unchanged)
Follow NIP-57 Appendix F.

### For CAIP-358 Zaps

Clients MUST verify CAIP-358 zap receipts by:

1. The `zap receipt` event MUST have a valid signature from the recipient's zap server pubkey.
2. The `description` tag MUST contain a valid kind `9734` zap request.
3. The `caip358_receipt` MUST contain a valid CAIP-358 response with matching `orderId`.
4. For PUSH payments:
   - The `txid` MUST reference a confirmed transaction on the specified `network`.
   - The transaction MUST transfer the correct `asset`, `amount`, and `recipient` as specified in the zap request.
5. For PULL payments:
   - The zap server MUST have settled the payment before publishing the receipt.

## Security Considerations

- **On-chain verification**: Unlike Lightning (where only the zap server can verify payment), CAIP-358 PUSH payments are publicly verifiable on-chain. This is a privacy tradeoff but a verification improvement.
- **Finality**: Zap servers MUST verify transaction finality before publishing receipts.
- **Replay protection**: The `orderId` field in CAIP-358 prevents replay attacks. Zap servers MUST reject duplicate `orderId` values.
- **Asset validation**: Clients MUST verify that the `asset` in the receipt matches one of the `paymentOptions` in the original request.

## Multi-Asset Payment Options

A key benefit of CAIP-358 is offering multiple payment options in a single request. A zap server can accept sBTC, USDC, native tokens, or Lightning — and the wallet picks based on what the sender holds:

```json
{
  "paymentOptions": [
    {
      "asset": "stacks:1/SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token.sbtc-token",
      "amount": "0x64",
      "recipient": "SP1JBH94STS4MHD61H3HA1ZN2R4G41EZGFG9SXP66",
      "types": ["sip10-transfer"]
    },
    {
      "asset": "eip155:1/erc20:0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599",
      "amount": "0x64",
      "recipient": "0x71C7656EC7ab88b098defB751B7401B5f6d8976F",
      "types": ["erc20-transfer"]
    }
  ]
}
```

## Real-World Example: sBTC Zap on Stacks Mainnet

The following is a real sBTC zap performed on February 20, 2026 by cocoa007 (`npub19drq8533690hw80d8ekpacjs67dan2y4pka09emqzh2mkhr9uxvqd4k3nn`) to Stark Comet on Stacks mainnet.

### On-Chain Transactions

Two sBTC transfers of 100 sats each to `SP1JBH94STS4MHD61H3HA1ZN2R4G41EZGFG9SXP66`:

- **tx1**: [`5e2422845c5e0908e3ae2d5a424b19a625d602cf82b1f781f6e97e21abbb1f27`](https://explorer.hiro.so/txid/5e2422845c5e0908e3ae2d5a424b19a625d602cf82b1f781f6e97e21abbb1f27?chain=mainnet)
- **tx2**: [`6a3dd5787dd1f828c5ffe42bda90f086d7d771f42d9af6c0c16baded2e3f41b0`](https://explorer.hiro.so/txid/6a3dd5787dd1f828c5ffe42bda90f086d7d771f42d9af6c0c16baded2e3f41b0?chain=mainnet)

### Nostr Event (kind 1 — proof of concept)

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
    ["r", "https://explorer.hiro.so/txid/6a3dd5787dd1f828c5ffe42bda90f086d7d771f42d9af6c0c16baded2e3f41b0?chain=mainnet"]
  ],
  "content": "⚡ sBTC zap to Stark Comet\n\nSent 200 sats sBTC (2x 100 sats) on Stacks mainnet — bitcoin-backed micropayments, no Lightning needed.\n\ntx1: https://explorer.hiro.so/txid/5e2422845c5e0908e3ae2d5a424b19a625d602cf82b1f781f6e97e21abbb1f27?chain=mainnet\ntx2: https://explorer.hiro.so/txid/6a3dd5787dd1f828c5ffe42bda90f086d7d771f42d9af6c0c16baded2e3f41b0?chain=mainnet\n\nRecipient: SP1JBH94STS4MHD61H3HA1ZN2R4G41EZGFG9SXP66 (Stark Comet)\nToken: SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token",
  "id": "59a3588f3f9dca87dac921aaaab651ed5fcc7f829c36b2ba6f20f6ac8837fae4"
}
```

### How This Would Look With NIP-XXX (kind 9734 + 9735)

**Zap Request (kind 9734):**

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
    ["caip358_request", "{\"version\":1,\"orderId\":\"zap-cocoa007-starkcomet-1740038500\",\"expiry\":1740038800,\"paymentOptions\":[{\"asset\":\"stacks:1/SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token.sbtc-token\",\"amount\":\"0xC8\",\"recipient\":\"SP1JBH94STS4MHD61H3HA1ZN2R4G41EZGFG9SXP66\",\"types\":[\"sip10-transfer\"]}]}"]
  ]
}
```

**Zap Receipt (kind 9735):**

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
    ["asset", "stacks:1/SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token.sbtc-token"],
    ["caip358_receipt", "{\"version\":1,\"orderId\":\"zap-cocoa007-starkcomet-1740038500\",\"payment\":{\"asset\":\"stacks:1/SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-token.sbtc-token\",\"amount\":\"0xC8\",\"recipient\":\"SP1JBH94STS4MHD61H3HA1ZN2R4G41EZGFG9SXP66\",\"types\":[\"sip10-transfer\"]},\"receipt\":{\"type\":\"sip10-transfer\",\"hash\":\"5e2422845c5e0908e3ae2d5a424b19a625d602cf82b1f781f6e97e21abbb1f27\",\"data\":{\"from\":\"SP16H0KE0BPR4XNQ64115V5Y1V3XTPGMWG5YPC9TR\",\"to\":\"SP1JBH94STS4MHD61H3HA1ZN2R4G41EZGFG9SXP66\",\"value\":\"0xC8\"}}}"]
  ]
}
```

**Verification:**

1. Decode `description` → get the original kind 9734 zap request
2. Read `txid` tags → query `GET https://api.hiro.so/extended/v1/tx/{txid}`
3. Verify sender, recipient, amount, and asset match the zap request
4. Confirm transactions are finalized (anchored block)

## References

- [NIP-57](https://github.com/nostr-protocol/nips/blob/master/57.md) — Lightning Zaps
- [CAIP-358](https://standards.chainagnostic.org/CAIPs/caip-358) — Universal Payment Request Method
- [CAIP-2](https://standards.chainagnostic.org/CAIPs/caip-2) — Blockchain ID Specification
- [CAIP-19](https://standards.chainagnostic.org/CAIPs/caip-19) — Asset Type and Asset ID Specification
- [ChainAgnostic/namespaces#167](https://github.com/ChainAgnostic/namespaces/pull/167) — CAIP-19 asset specification for Stacks
- [BOLT-11](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md) — Lightning Invoice Protocol

## Copyright

This NIP is placed in the public domain.
