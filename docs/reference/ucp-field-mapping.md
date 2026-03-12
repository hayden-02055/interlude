# UCP 필드 매핑 레퍼런스

Market MCP Server가 Solana 데이터를 UCP 형식으로 변환할 때의 필드 매핑 레퍼런스.

---

## Checkout 필드 매핑

| UCP 필드 | Solana 소스 | 변환 |
|----------|------------|------|
| `ucp.version` | 하드코딩 | `"2026-01-11"` |
| `ucp.capabilities` | MerchantProfile.supports_* | bool → capability 배열 |
| `ucp.payment_handlers` | 하드코딩 + MerchantProfile | 결제 모듈 정보 (예: sol.usdc) |
| `id` | CheckoutSession.checkout_id | bytes → hex string `"chk_"` prefix |
| `status` | CheckoutSession.status | enum → string (6개 상태) |
| `currency` | `"USD"` (고정) | USDC = USD 기반 스테이블코인, ISO 4217 |
| `line_items[].id` | CheckoutLineItem.index | `"li_"` + index |
| `line_items[].item.id` | CheckoutLineItem.product_id | 직접 매핑 |
| `line_items[].item.title` | CheckoutLineItem.title | 직접 매핑 |
| `line_items[].item.price` | CheckoutLineItem.unit_price | 직접 매핑 |
| `line_items[].item.image_url` | ProductListing.image_uri | Arweave URI |
| `line_items[].quantity` | CheckoutLineItem.quantity | u32 → int |
| `line_items[].totals` | 계산 | unit_price × quantity |
| `totals[].type` | 개별 필드 매핑 | subtotal/discount/fulfillment/tax/fee/total |
| `totals[].amount` | CheckoutSession.subtotal 등 | 직접 매핑 (0인 항목은 생략) |
| `totals[].display_text` | MerchantProfile or 하드코딩 | 선택적 |
| `buyer.email` | Off-chain DB (원본) | 해시 검증 후 반환 |
| `buyer.first_name` | Off-chain DB (원본) | 해시 검증 후 반환 |
| `buyer.last_name` | Off-chain DB (원본) | 해시 검증 후 반환 |
| `buyer.phone_number` | Off-chain DB (원본) | 해시 검증 후 반환 |
| `context.country` | CheckoutSession.buyer_country | 2-byte → string |
| `context.language` | CheckoutSession.buyer_language | 2-byte → string |
| `messages` | CheckoutSession.error_flags | 비트 플래그 → Message[] |
| `links` | MerchantProfile.*_url | URL 필드 → Link[] |
| `expires_at` | CheckoutSession.expires_at | i64 → RFC 3339 |
| `continue_url` | Off-chain DB | status == RequiresEscalation 시만 포함 |
| `payment` | sol.usdc 핸들러 정보 | 자동 구성 |
| `order.id` | Order.order_id | bytes → hex string `"ord_"` prefix |
| `order.permalink_url` | 생성 | Solana explorer 또는 커스텀 URL |

---

## Order 필드 매핑

| UCP 필드 | Solana 소스 | 변환 |
|----------|------------|------|
| `id` | Order.order_id | bytes → hex string |
| `checkout_id` | Order.checkout_id | bytes → hex string |
| `permalink_url` | 생성 | Solana explorer URL |
| `line_items[].quantity` | CheckoutLineItem.quantity + FulfillmentEvent 집계 | `{ total: u32, fulfilled: u32 }` 객체로 변환 |
| `line_items[].status` | FulfillmentEvent 집계 | `"processing"` / `"partial"` / `"fulfilled"` |
| `line_items` | CheckoutLineItem[] PDA | 불변 스냅샷 |
| `totals` | Order.subtotal/tax/... | 개별 필드 → Total[] |
| `fulfillment.expectations` | FulfillmentExpectation[] PDA + EncryptedBuyerInfo 또는 Curator | 배열, destination은 Merchant가 복호화한 PII에서 복원 |
| `fulfillment.events` | FulfillmentEvent[] PDA | append-only 이벤트 목록 |
| `adjustments` | Adjustment[] PDA | append-only 조정 목록 |
