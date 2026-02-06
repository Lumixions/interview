# Phase 2 Architecture Diagrams

## 1) Blockchain payment flow (sequence diagram)

```mermaid
sequenceDiagram
  participant Buyer as BuyerApp
  participant Api as BackendAPI
  participant Indexer as BlockchainIndexer
  participant Chain as Blockchain

  Note over Buyer,Chain: Buyer selects Pay with Blockchain
  
  Buyer->>Api: POST /orders/ID/checkout_crypto
  Api->>Api: Generate payment address
  Api->>Api: Create crypto_payment_request
  Api-->>Buyer: payment_address, amount, qr_code, expiry
  
  Buyer->>Buyer: Display address, QR code, amount
  Note over Buyer: Buyer opens wallet, sends crypto
  
  Buyer->>Chain: Send USDC to payment_address
  Chain-->>Indexer: Transaction detected
  
  Indexer->>Api: POST /webhooks/blockchain tx_hash
  Api->>Api: Verify tx on-chain
  Api->>Api: Mark order PAID, store tx_hash
  Api-->>Indexer: 200 OK
  
  Buyer->>Api: GET /orders/ID polling
  Api-->>Buyer: Order status PAID
  Buyer->>Buyer: Show Payment confirmed
```

---

## 2) Blockchain escrow flow (with smart contract)

```mermaid
sequenceDiagram
  participant Buyer as BuyerApp
  participant Api as BackendAPI
  participant Contract as EscrowSmartContract
  participant Seller as SellerApp
  participant Chain as Blockchain

  Note over Buyer,Chain: Buyer pays via blockchain with escrow

  Buyer->>Api: POST /orders/ID/checkout_crypto escrow true
  Api->>Contract: Deploy escrow orderId seller_address
  Contract-->>Api: Escrow address
  Api-->>Buyer: escrow_address amount qr_code

  Buyer->>Buyer: Send USDC to escrow_address
  Buyer->>Chain: Transfer USDC
  Chain->>Contract: Funds locked in escrow

  Api->>Api: Detect escrow funded mark order PAID_ESCROWED

  Note over Seller: Seller ships product
  Seller->>Api: Update order SHIPPED

  Note over Buyer: Buyer receives product
  Buyer->>Api: POST /orders/ID/confirm_delivery
  Api->>Contract: confirmDelivery orderId
  Contract->>Chain: Release funds to seller
  Chain-->>Seller: Funds transferred

  Alt Buyer disputes
    Buyer->>Api: POST /orders/ID/dispute
    Api->>Api: Order status DISPUTED
    Note over Api: Admin reviews dispute
    Api->>Contract: refund orderId or releaseFunds orderId
    Contract->>Chain: Refund buyer or pay seller
  End
```

---

## 3) Seller verification workflow

```mermaid
flowchart TD
  A[Seller signs in] --> B{Has seller profile?}
  B -->|No| C[Create seller profile]
  B -->|Yes| D{Verified?}
  
  C --> D
  D -->|No| E[Show: Upload verification docs]
  D -->|Yes| F[Full access: unlimited products]
  
  E --> G[Seller uploads ID, business docs]
  G --> H[POST /seller/verification]
  H --> I[Backend stores docs in S3]
  I --> J[Admin reviews in dashboard]
  
  J --> K{Approve?}
  K -->|Yes| L[Update status: APPROVED]
  K -->|No| M[Update status: REJECTED, add reason]
  
  L --> N[Seller sees: Verified badge]
  M --> O[Seller sees: Rejected, re-upload]
  
  N --> F
  O --> E
  
  D -->|Unverified| P[Limited: max 3 products]
```

---

## 4) Dispute resolution workflow

```mermaid
flowchart TD
  A[Order status: PAID] --> B[Buyer waits for delivery]
  B --> C{Product arrived?}
  
  C -->|Yes| D[Order complete]
  C -->|No, after 7 days| E[Buyer opens dispute]
  
  E --> F[POST /orders/id/dispute]
  F --> G[Order status: DISPUTED]
  G --> H[Seller notified]
  
  H --> I[Seller responds with proof]
  I --> J[POST /disputes/id/respond]
  J --> K[Admin reviews evidence]
  
  K --> L{Decision?}
  L -->|Refund buyer| M["Admin: POST /admin/disputes/id/resolve refund=true"]
  L -->|Release to seller| N["Admin: POST /admin/disputes/id/resolve refund=false"]
  
  M --> O[Refund buyer, order: REFUNDED]
  N --> P[Seller keeps payment, order: RESOLVED]
  
  O --> Q[Dispute closed]
  P --> Q
```

---

## 5) Phase 2 system architecture (components)

```mermaid
flowchart TB
  subgraph Client [Client Layer]
    FlutterApp[Flutter App Mobile+Web]
  end
  
  subgraph Backend [Backend Layer]
    API[FastAPI Service]
    DB[(Postgres DB)]
  end
  
  subgraph Storage [Storage Layer]
    S3[(S3 Bucket)]
  end
  
  subgraph Payments [Payment Systems]
    Stripe[Stripe Checkout]
    Indexer[Blockchain Indexer]
    Contract[Escrow Smart Contract]
  end
  
  subgraph Blockchain [Blockchain]
    Chain[Ethereum / Polygon]
  end
  
  FlutterApp -->|"GET /products/:id"| API
  FlutterApp -->|"POST /orders/ID/checkout"| API
  FlutterApp -->|"POST /orders/ID/checkout_crypto"| API
  FlutterApp -->|"POST /seller/verification"| API
  FlutterApp -->|"POST /orders/ID/dispute"| API
  
  API --> DB
  API -->|Presign URLs| S3
  FlutterApp -->|Upload images| S3
  
  API -->|Create CheckoutSession| Stripe
  Stripe -->|Webhook| API
  
  API -->|Generate payment address| Indexer
  Indexer -->|Monitor txs| Chain
  Indexer -->|Webhook payment detected| API
  
  API -->|Deploy/interact| Contract
  Contract -->|On-chain| Chain
  
  style FlutterApp fill:#e1f5ff
  style API fill:#fff4e1
  style DB fill:#f0f0f0
  style Stripe fill:#e8f5e9
  style Chain fill:#fce4ec
```

---

## 6) Anti-scam protection layers

```mermaid
flowchart LR
  subgraph Prevention [Prevention Layer]
    A1[Seller Verification]
    A2[Rate Limiting]
    A3[Spam Detection]
  end
  
  subgraph Detection [Detection Layer]
    B1[Duplicate Listings]
    B2[Fake Orders]
    B3[Suspicious Behavior]
  end
  
  subgraph Resolution [Resolution Layer]
    C1[Dispute System]
    C2[Blockchain Escrow]
    C3[Admin Review]
  end
  
  Prevention --> Detection
  Detection --> Resolution
  
  A1 -->|Verified badge| Buyer
  A2 -->|Block spam| Buyer
  B1 -->|Flag| Admin
  C1 -->|Refund/resolve| Buyer
  C2 -->|Trustless release| Seller
  
  style Prevention fill:#e8f5e9
  style Detection fill:#fff9c4
  style Resolution fill:#ffccbc
```

---

## Notes

- **Blockchain indexer options**: Alchemy Notify, QuickNode, Moralis, or self-hosted polling
- **Smart contract audit**: Required before mainnet deployment for escrow
- **Admin dashboard**: Can be simple Flutter web app or separate React/Next.js admin portal
- **Verification KYC**: Consider Stripe Identity, Persona, or Sumsub for automated verification (Phase 3)
