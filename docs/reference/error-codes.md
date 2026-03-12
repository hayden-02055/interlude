# 에러 코드 레퍼런스

Interlude 시스템의 에러 코드 레퍼런스. 프로토콜 에러, 비즈니스 에러, on-chain 에러를 한 곳에서 관리한다.

---

## MCP 프로토콜 에러 (JSON-RPC)

프로토콜 에러는 JSON-RPC error 객체로 반환한다. 비즈니스 에러와 구분된다.

```typescript
// 프로토콜 에러 → JSON-RPC error 객체로 반환
// 비즈니스 에러 → JSON-RPC result + messages 배열로 반환 (정상 응답)

function buildProtocolError(code: number, message: string) {
  return {
    jsonrpc: "2.0",
    id: requestId,
    error: { code, message }
  };
}
```

### 에러 코드 매핑

| 코드 | 이름 | 설명 | 사용 예시 |
|------|------|------|-----------|
| `-32001` | DISCOVERY_ERROR | 상점 프로필 조회 실패, 프로필 형식 오류 | MerchantProfile PDA 없음 |
| `-32000` | PROTOCOL_ERROR | 인증 실패, 권한 없음, 멱등성 충돌, 속도 제한 | Buyer 서명 누락, 같은 idempotency-key로 다른 checkout에 complete |
| `-32603` | INTERNAL_ERROR | Solana RPC 실패, 예상치 못한 오류 | Solana RPC 타임아웃 |

> 비즈니스 에러 (예: 재고 부족)는 프로토콜 에러가 아니라 200 OK + `messages[{ code: "out_of_stock" }]`로 반환한다.

---

## error_flags 비트 레이아웃 및 messages 생성

CheckoutSession의 `error_flags: u32` 필드를 UCP `messages[]` 배열로 변환하는 로직이다.

### 비트 영역

| 비트 범위 | 용도 | UCP message type |
|-----------|------|------------------|
| bit 0-15 | Errors | `"error"` |
| bit 16-23 | Warnings | `"warning"` |
| bit 24-31 | Info | `"info"` |

### Error 비트 (bit 0-15)

| 비트 | 플래그 | code | path | severity | 메시지 |
|------|--------|------|------|----------|--------|
| 0 | `0x0001` | `missing` | `$.buyer.email` | `recoverable` | Buyer email is required |
| 1 | `0x0002` | `missing` | `$.buyer.first_name` | `recoverable` | Buyer name is required |
| 2 | `0x0004` | `missing` | `$.fulfillment` | `recoverable` | Shipping address is required |
| 3 | `0x0008` | `missing` | `$.line_items` | `recoverable` | At least one line item is required |
| 4 | `0x0010` | `missing` | `$.buyer.phone_number` | `recoverable` | Phone number is required |
| 5 | `0x0020` | `payment_declined` | `$.payment` | `recoverable` | Insufficient USDC balance |
| 6 | `0x0040` | `requires_sign_in` | `$.buyer` | `requires_buyer_input` | Buyer must connect their Solana wallet |
| 7 | `0x0080` | `out_of_stock` | `$.line_items` | `recoverable` | One or more items are out of stock |

### Warning 비트 (bit 16-23)

| 비트 | 플래그 | code | path | 메시지 |
|------|--------|------|------|--------|
| 16 | `0x010000` | `final_sale` | `$.line_items` | This item is final sale and cannot be returned |
| 17 | `0x020000` | `fulfillment_changed` | `$.fulfillment` | Shipping method has been updated |
| 18 | `0x040000` | `price_changed` | `$.line_items` | Product price has changed since cart was created |

### Info 비트 (bit 24-31)

| 비트 | 플래그 | code | 메시지 |
|------|--------|------|--------|
| 24 | `0x01000000` | — | Free shipping on orders over $50! |
| 25 | `0x02000000` | `loyalty_points` | You will earn 100 loyalty points with this purchase |

### errorFlagsToMessages 변환 함수

```typescript
interface UcpMessage {
  type: "error" | "warning" | "info";
  code?: string;                    // error/warning: 필수, info: 선택
  path?: string;
  severity?: "recoverable" | "requires_buyer_input" | "requires_buyer_review";
                                    // error: 필수, warning/info: 없음
  content: string;
  content_type?: "plain" | "markdown";
}

function errorFlagsToMessages(flags: number): UcpMessage[] {
  const messages: UcpMessage[] = [];

  // === Errors (bit 0-15) ===
  if (flags & 0x0001) messages.push({
    type: "error", code: "missing",
    path: "$.buyer.email", severity: "recoverable",
    content: "Buyer email is required"
  });

  if (flags & 0x0002) messages.push({
    type: "error", code: "missing",
    path: "$.buyer.first_name", severity: "recoverable",
    content: "Buyer name is required"
  });

  if (flags & 0x0004) messages.push({
    type: "error", code: "missing",
    path: "$.fulfillment", severity: "recoverable",
    content: "Shipping address is required"
  });

  if (flags & 0x0008) messages.push({
    type: "error", code: "missing",
    path: "$.line_items", severity: "recoverable",
    content: "At least one line item is required"
  });

  if (flags & 0x0010) messages.push({
    type: "error", code: "missing",
    path: "$.buyer.phone_number", severity: "recoverable",
    content: "Phone number is required"
  });

  if (flags & 0x0020) messages.push({
    type: "error", code: "payment_declined",
    path: "$.payment", severity: "recoverable",
    content: "Insufficient USDC balance"
  });

  if (flags & 0x0040) messages.push({
    type: "error", code: "requires_sign_in",
    path: "$.buyer", severity: "requires_buyer_input",
    content: "Buyer must connect their Solana wallet"
  });

  if (flags & 0x0080) messages.push({
    type: "error", code: "out_of_stock",
    path: "$.line_items", severity: "recoverable",
    content: "One or more items are out of stock"
  });

  // === Warnings (bit 16-23) ===
  if (flags & 0x010000) messages.push({
    type: "warning", code: "final_sale",
    path: "$.line_items",
    content: "This item is final sale and cannot be returned"
  });

  if (flags & 0x020000) messages.push({
    type: "warning", code: "fulfillment_changed",
    path: "$.fulfillment",
    content: "Shipping method has been updated"
  });

  if (flags & 0x040000) messages.push({
    type: "warning", code: "price_changed",
    path: "$.line_items",
    content: "Product price has changed since cart was created"
  });

  // === Info (bit 24-31) ===
  if (flags & 0x01000000) messages.push({
    type: "info",
    content: "Free shipping on orders over $50!"
  });

  if (flags & 0x02000000) messages.push({
    type: "info", code: "loyalty_points",
    content: "You will earn 100 loyalty points with this purchase"
  });

  return messages;
}
```

---

## CommerceError (On-chain, Rust)

Solana Program에서 사용하는 커스텀 에러 코드. UCP messages와 매핑된다.

```rust
#[error_code]
pub enum CommerceError {
    // === Checkout 에러 (UCP message.code에 매핑) ===
    #[msg("Buyer email is required")]
    BuyerEmailMissing,           // → error, missing, $.buyer.email, recoverable

    #[msg("Buyer name is required")]
    BuyerNameMissing,            // → error, missing, $.buyer.first_name, recoverable

    #[msg("Shipping address is required")]
    ShippingAddressMissing,      // → error, missing, $.fulfillment, recoverable

    #[msg("Phone number is required")]
    PhoneMissing,                // → error, missing, $.buyer.phone_number, recoverable

    #[msg("Product not found")]
    ProductNotFound,             // → error, invalid, $.line_items, recoverable

    #[msg("Insufficient stock")]
    InsufficientStock,           // → error, out_of_stock, $.line_items, recoverable

    #[msg("Checkout session expired")]
    CheckoutExpired,             // → status = canceled

    #[msg("Invalid checkout status for this operation")]
    InvalidCheckoutStatus,

    #[msg("Requires escalation to merchant UI")]
    RequiresEscalation,          // → status = requires_escalation

    // === 결제 에러 ===
    #[msg("Insufficient USDC balance")]
    InsufficientBalance,         // → error, payment_declined, $.payment, recoverable

    #[msg("Escrow transfer failed")]
    EscrowTransferFailed,

    // === 권한 에러 ===
    #[msg("Unauthorized: not the merchant authority")]
    UnauthorizedMerchant,

    #[msg("Unauthorized: not the buyer")]
    UnauthorizedBuyer,

    // === 주문 에러 ===
    #[msg("Order already fulfilled")]
    AlreadyFulfilled,

    #[msg("Refund exceeds remaining escrow")]
    RefundExceedsEscrow,

    #[msg("Escrow already released")]
    EscrowAlreadyReleased,

    #[msg("Escrow release time not reached")]
    EscrowReleaseTimeNotReached,

    // === 멱등성 ===
    #[msg("Idempotency key already used for a different operation")]
    IdempotencyConflict,

    // === Line Item ===
    #[msg("Maximum line items (10) exceeded")]
    MaxLineItemsExceeded,

    // === Buyer PII ===
    #[msg("Buyer info PDA already exists for this order")]
    BuyerInfoAlreadyExists,

    #[msg("Buyer info PDA not found")]
    BuyerInfoNotFound,

    #[msg("Buyer info TTL not reached for auto-close")]
    BuyerInfoTtlNotReached,

    // === Curator Badge ===
    #[msg("Maximum attestations (5) exceeded")]
    MaxAttestationsExceeded,

    #[msg("Attestation not found for this curator and type")]
    AttestationNotFound,

    #[msg("Attestation has expired")]
    AttestationExpired,
}
```

### CommerceError → UCP message 매핑 요약

| CommerceError | UCP code | path | severity |
|---------------|----------|------|----------|
| BuyerEmailMissing | `missing` | `$.buyer.email` | `recoverable` |
| BuyerNameMissing | `missing` | `$.buyer.first_name` | `recoverable` |
| ShippingAddressMissing | `missing` | `$.fulfillment` | `recoverable` |
| PhoneMissing | `missing` | `$.buyer.phone_number` | `recoverable` |
| ProductNotFound | `invalid` | `$.line_items` | `recoverable` |
| InsufficientStock | `out_of_stock` | `$.line_items` | `recoverable` |
| InsufficientBalance | `payment_declined` | `$.payment` | `recoverable` |
| CheckoutExpired | — | — | status → `canceled` |
| RequiresEscalation | — | — | status → `requires_escalation` |
