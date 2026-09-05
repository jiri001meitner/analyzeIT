# Wasm Marketplace Licensing and Bitcoin Payments

This document outlines the commercial licensing model for analyzeIT WebAssembly
plugins using Bitcoin and the Lightning Network.

## 1. Rationale: Why Bitcoin Over Traditional Gateways

Traditional fiat payment gateways (Stripe, PayPal, Adyen) introduce significant
friction and architectural drawbacks for an open-source, privacy-first tool:

- **KYC & Corporate Overhead**: Requires merchant entity verification, banking
  integrations, and exposure to arbitrary account freezes.
- **Privacy Mismatch**: Collecting personal names, billing addresses, and credit
  card data directly contradicts the zero-tracking ethos of analyzeIT.
- **Chargeback Risks**: Digital software licenses are vulnerable to fraudulent
  card chargebacks. Bitcoin settlements are final and irreversible.
- **Global Availability**: Bitcoin and Lightning operate uniformly across all
  jurisdictions without cross-border credit card fees or country restrictions.

## 2. Payment Architecture: Lightning Network First

For digital goods and micro-transactions, the **Lightning Network** is the
primary payment transport:

1. **Near-Instant Settlement**: Invoices settle in seconds (< 2s), enabling
   immediate module activation.
1. **Sub-Cent Fees**: Ultra-low routing fees allow flexible licensing models,
   including flat lifetime licenses, annual seats, or pay-per-run micro-fees.
1. **L402 / LSAT Support**: Compatibility with the HTTP `402 Payment Required`
   standard for programmatic module invocations.

## 3. Cryptographic Offline License Verification

To prevent phone-home telemetry and ensure air-gapped security, licenses are
verified cryptographically via asymmetric keys (Ed25519):

```text
[ analyzeIT Dashboard ] ──(Request Invoice)──► [ Marketplace API ]
           │                                            │
   (Pay LN Invoice)                             (Generate Token)
           ▼                                            ▼
[ Lightning Network ] ───(Settlement Event)────► [ Sign License ]
                                                        │
[ SQLite Storage ] ◄─────(Return Signed Token)──────────┘
```

### License Token Format

```json
{
  "module_id": "attribution-advanced",
  "instance_id": "inst-a98f12",
  "issued_at": 1772755200,
  "expires_at": 0,
  "signature": "ed25519_signature_hex"
}
```

The self-hosted instance verifies the token using the marketplace public key
baked into the binary or configured in `.env`. No continuous network connection
to the marketplace is required after token delivery.

## 4. Trustless Revenue Splitting (Developer Royalties)

A core requirement is ensuring developers receive their revenue share
automatically without manual accounting or centralized payout delays.

### A. On-Chain Bitcoin: Native Multi-Output Transactions

On the base Bitcoin layer, splitting is enforced directly by the ledger:

1. **Atomic Multi-Output Transaction**: When purchasing via on-chain Bitcoin,
   the payment transaction defines multiple outputs simultaneously:
   - Output 0: Developer's Bitcoin address (e.g., 80% of price).
   - Output 1: Marketplace platform address (e.g., 20% platform fee).
1. **Trustless Execution**: The transaction is mined atomically in a single
   block. The marketplace never takes custody of the developer's funds; the
   buyer pays both parties directly in one cryptographic operation.

### B. Lightning Network: Real-Time Payment Splitting

For micro-transactions, the platform utilizes Lightning split mechanisms:

1. **Automated KeySend / Split Routing**: When the customer settles a Lightning
   invoice, the marketplace routing node instantly streams the developer's
   allotted percentage directly to their Lightning Address (e.g.,
   `alice@getalby.com`) via programmatic KeySend or LNURL-Pay splits.
1. **Zero Custody Friction**: Developers receive funds in seconds into their
   self-custodial wallet without waiting for monthly billing cycles or minimum
   withdrawal thresholds.

### Module Royalty Specification

Wasm modules declare payment destinations in their manifest (`manifest.json`):

```json
{
  "pricing": {
    "price_sats": 50000,
    "developer_lightning_address": "dev@ln.provider.com",
    "developer_onchain_address": "bc1q...",
    "developer_royalty_percent": 85
  }
}
```
