# Phase 2 Roadmap: Product Details + Blockchain Payments + Anti-Scam

## Executive summary

Phase 2 expands the marketplace MVP with three strategic pillars:
1. **Product detail screens** (better buyer experience)
2. **Blockchain payment option** (payment flexibility, crypto users)
3. **Anti-scam protections** (trust and safety)

**Timeline:** 8–12 weeks  
**Team:** 1 senior Flutter developer + backend support + smart contract developer (for escrow)

---

## Feature 1: Product detail screen

### Goal
Give buyers confidence by showing full product information before purchase.

### User story
As a buyer, I want to see detailed product information (images, description, seller info, stock) so I can make an informed purchase decision.

### Requirements

#### Must-have
- Full product title and description
- Image gallery (swipeable, zoomable on mobile)
- Price, currency, stock availability
- Seller name and verification badge
- Add-to-cart with quantity selector
- "Out of stock" state
- Back navigation to catalog

#### Nice-to-have (Phase 3)
- Reviews and ratings
- Related products
- Share product link

### Technical design

#### Backend changes
- Existing `GET /products/{id}` already returns full product data
- No backend changes needed for MVP

#### Flutter changes
- Add route `/product/:id` in `go_router`
- Create `ProductDetailScreen` widget
- Create reusable `ImageGallery` widget (horizontal `PageView` with indicators)
- Add quantity stepper to `cart_state.dart`
- Navigation: tap product in catalog → `context.go('/product/${product.id}')`

#### State management
- Fetch product by ID using `FutureProvider` or `AsyncNotifier`
- Cart state already exists (`CartController` in Riverpod)

### Acceptance criteria
- [ ] User can tap product in catalog and see detail screen
- [ ] Image gallery shows all product images, swipeable
- [ ] User can select quantity (1–stock_qty)
- [ ] Add-to-cart updates cart state
- [ ] Out-of-stock products show clear disabled state
- [ ] Works on mobile and web (responsive)

### Estimated effort
**1–2 weeks**

---

## Feature 2: Blockchain payment option

### Goal
Offer crypto payments as an alternative to Stripe for users who prefer blockchain transactions.

### User story
As a buyer, I want to pay with cryptocurrency (USDC, ETH) so I don't need to use a credit card.

### Requirements

#### Must-have (Phase 2A)
- Payment method selector: Stripe or Blockchain
- Supported tokens: USDC (Ethereum mainnet or Polygon)
- Display payment details: amount, wallet address, QR code
- Detect on-chain payment and mark order 'paid'
- Clear error messages (wrong amount, timeout)

#### Nice-to-have (Phase 2B)
- Support more tokens (ETH, DAI, USDT)
- Support multiple chains (Polygon, Arbitrum, Base)
- Blockchain escrow for buyer protection

### Technical design

#### Architecture (big picture)

```
[Flutter App]
   ↓ POST /orders/{id}/checkout_crypto
[Backend API]
   ↓ Generate unique address or monitor specific tx
[Blockchain Indexer / Webhook Service]
   ↓ Detect payment on-chain
[Backend API]
   ↓ POST /webhooks/blockchain (mark order PAID)
[Database]
```

#### Backend changes

**New table: `crypto_payment_requests`**
```sql
id, order_id, token, amount, recipient_address, tx_hash, status, created_at
```

**New endpoints:**
- `POST /orders/{id}/checkout_crypto`
  - Input: `{ token: "USDC", chain: "ethereum" }`
  - Output: `{ payment_address, amount, qr_code_data, expiry }`
- `POST /webhooks/blockchain`
  - Receives webhook from indexer when payment detected
  - Marks order paid, stores tx_hash

**Blockchain indexer options:**
1. **Self-hosted indexer**: Poll blockchain for incoming txs to our address
2. **Third-party webhook**: Alchemy Notify, QuickNode, Moralis, etc.
3. **Simple polling**: App polls backend, backend checks on-chain balance

**Recommended approach for MVP:**  
Use **Alchemy Notify** or **QuickNode webhook** to detect USDC transfers to a specific address.

#### Flutter changes

**New screen: `BlockchainCheckoutScreen`**
- Display wallet address (copyable)
- Display QR code (scannable by wallet app)
- Display amount and token
- Show "Waiting for payment..." state
- Poll backend `/orders/{id}` to check payment status
- Navigate to "Payment confirmed" or timeout after 15 minutes

**Payment flow:**
1. Buyer reaches checkout, selects "Pay with Blockchain"
2. App calls `POST /orders/{id}/checkout_crypto`
3. Backend returns address, amount, QR code
4. Flutter shows `BlockchainCheckoutScreen`
5. Buyer opens wallet app, scans QR or copies address, sends crypto
6. Indexer detects payment → webhook → backend marks order paid
7. Flutter polling detects status change → shows success

#### State management
- Add `PaymentMethod` enum: `stripe | blockchain`
- Add `BlockchainPaymentRequest` model
- Use `AsyncNotifier` to poll order status

### Security considerations
- Validate payment amount and token on-chain (not just trust webhook)
- Store recipient address securely
- Rate-limit checkout to prevent spam address generation

### Acceptance criteria
- [ ] Buyer can select "Blockchain" payment method at checkout
- [ ] Blockchain checkout screen shows address, QR, amount
- [ ] System detects on-chain payment and marks order paid
- [ ] Buyer sees "Payment confirmed" after successful tx
- [ ] Stripe flow still works unchanged
- [ ] Clear error handling (timeout, wrong amount, wrong token)

### Estimated effort
**3–5 weeks**
- Backend + indexer integration: 1–2 weeks
- Flutter UI + payment flow: 1–2 weeks
- Testing (testnet, mainnet): 1 week

---

## Feature 3: Anti-scam protections

### Goal
Build trust and safety mechanisms to protect buyers and sellers from fraud.

### User stories
- As a buyer, I want to know if a seller is verified so I can trust them.
- As a buyer, I want to open a dispute if the product doesn't arrive.
- As a seller, I want to respond to disputes with proof of shipment.
- As a platform, I want to detect and block spam/fake accounts.

### Requirements

#### Phase 2A: Seller verification (must-have)
- Sellers upload verification documents (ID, business license)
- Admin manually approves/rejects
- Verified sellers get a badge
- Unverified sellers have listing limits (e.g., max 3 products)

#### Phase 2B: Dispute system (must-have)
- Buyer can open dispute for order (status: `disputed`)
- Seller can respond with proof of shipment
- Admin resolves dispute: refund buyer or release payment to seller
- Order transitions: `PAID` → `DISPUTED` → `RESOLVED` or `REFUNDED`

#### Phase 2C: Rate limiting and spam detection (must-have)
- Backend rate-limits: max 10 products/hour, max 5 orders/hour per user
- Flag suspicious patterns: duplicate product titles, rapid account creation
- Admin dashboard to review flagged accounts

#### Phase 2D: Blockchain escrow (nice-to-have)
- For blockchain payments, hold funds in escrow smart contract
- Release to seller when buyer confirms delivery or after timeout (7 days)
- Buyer can trigger refund if seller doesn't ship

### Technical design

#### Backend changes

**New table: `seller_verifications`**
```sql
id, user_id, status (pending/approved/rejected), 
documents_s3_keys, admin_notes, created_at, reviewed_at
```

**New table: `disputes`**
```sql
id, order_id, opened_by_user_id, reason, description, 
seller_response, status (open/resolved/refunded), 
resolved_by_admin_id, resolved_at, created_at
```

**New endpoints:**
- `POST /seller/verification` (upload docs)
- `GET /seller/verification` (check status)
- `POST /orders/{id}/dispute` (buyer opens dispute)
- `POST /disputes/{id}/respond` (seller responds)
- `POST /admin/disputes/{id}/resolve` (admin resolves)

**Rate limiting:**
- Add middleware to track requests per user per time window
- Return `429 Too Many Requests` if exceeded

**Smart contract (optional, for escrow):**
```solidity
contract Escrow {
  function createEscrow(uint orderId, address seller) payable;
  function confirmDelivery(uint orderId); // buyer calls
  function releaseFunds(uint orderId); // after confirm or timeout
  function refund(uint orderId); // admin or buyer in dispute
}
```

#### Flutter changes

**Seller verification flow:**
- Screen: `SellerVerificationScreen`
- Upload ID/docs using image picker + S3 presign flow (same as product images)
- Show status: pending / approved / rejected
- If unverified, show banner "Verify your account to list more products"

**Dispute flow:**
- In buyer order detail, add "Open dispute" button (if order is `PAID` and >3 days old)
- Screen: `DisputeFormScreen` (reason dropdown, description text)
- In seller orders (future), show dispute notification, add "Respond" button

**Admin dashboard (web-only for MVP):**
- Simple Flutter web app or separate admin portal
- List pending verifications, disputes
- Approve/reject buttons

#### State management
- Add `SellerVerification` model and `AsyncNotifier`
- Add `Dispute` model
- Update `Order` model to include dispute status

### Security considerations
- Encrypt/redact sensitive verification docs in storage
- Admin-only endpoints must verify admin role (add `is_admin` flag to users table)
- Log all dispute actions for audit trail

### Acceptance criteria

**Seller verification:**
- [ ] Seller can upload verification docs
- [ ] Admin can approve/reject
- [ ] Verified sellers show badge on profile
- [ ] Unverified sellers limited to 3 products

**Dispute system:**
- [ ] Buyer can open dispute for paid order
- [ ] Seller sees dispute notification and can respond
- [ ] Admin can resolve dispute (refund or close)
- [ ] Order status reflects dispute state

**Rate limiting:**
- [ ] Spamming product creation is blocked
- [ ] Clear error message shown to user

**Blockchain escrow (if implemented):**
- [ ] Funds held in smart contract until delivery confirmed
- [ ] Buyer can confirm delivery → funds released to seller
- [ ] If disputed, admin can trigger refund

### Estimated effort
**4–6 weeks**
- Seller verification: 1–2 weeks
- Dispute system: 2–3 weeks
- Rate limiting: 1 week
- Blockchain escrow (optional): 2–3 weeks

---

## Phase 2 combined timeline

| Week | Milestone | Deliverables |
|------|-----------|--------------|
| 1–2 | Product detail screen | Detail UI, image gallery, navigation |
| 3 | Blockchain backend foundation | DB schema, indexer integration, webhooks |
| 4–5 | Blockchain Flutter UI | Payment method selector, crypto checkout screen, polling |
| 6 | Blockchain testing | Testnet, mainnet, edge cases |
| 7–8 | Seller verification | Upload flow, admin approval, badges |
| 9–11 | Dispute system | Buyer/seller dispute forms, admin resolution |
| 12 | Rate limiting + polish | Spam detection, final QA, docs |

**Total:** 12 weeks (3 months)  
**Can be parallelized:** Product detail + blockchain backend can run concurrently if team has 2 developers.

---

## Architecture diagrams

See `docs/diagrams/phase2_architecture.md` for:
- Blockchain payment sequence diagram
- Dispute workflow diagram
- Escrow smart contract flow

---

## Success metrics (Phase 2)

| Metric | Target |
|--------|--------|
| % buyers viewing product details before purchase | >80% |
| % orders using blockchain payment | >10% (if marketed to crypto users) |
| Seller verification completion rate | >60% |
| Dispute resolution time | <3 days (manual admin review) |
| Spam/fake listings blocked | >95% |

---

## Risks and mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Blockchain indexer downtime | High | Use reliable provider (Alchemy, QuickNode), add retry logic |
| Smart contract bugs (escrow) | Critical | Audit contract, test on testnet thoroughly, start with small limits |
| Seller verification backlog | Medium | Hire manual reviewers or integrate KYC API (Stripe Identity, Persona) |
| Dispute abuse (fake disputes) | Medium | Require evidence, flag repeat offenders, manual review |
| Crypto price volatility | Medium | Use stablecoins (USDC) primarily, show USD-equivalent amounts |

---

## Phase 3 preview (future)

After Phase 2, consider:
- Multi-seller cart and split orders
- Automatic dispute resolution with ML
- Reviews and ratings
- Seller analytics dashboard
- Push notifications for payment confirmations
- Multi-chain support (Solana, Polygon, Arbitrum)
