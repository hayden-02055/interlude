# Market

## Market이란?

Market은 Interlude 에코시스템의 온체인 커머스 계층이다. **Solana Program**과 **MCP Server** 두 컴포넌트로 구성되며, AI Agent가 상점 탐색부터 결제/주문까지 수행할 수 있는 신뢰 기반 커머스 인프라를 제공한다.

- **Solana Program** (`ucp_commerce`): 상점/상품/주문의 신뢰 데이터를 on-chain에 기록한다.
- **MCP Server**: AI Agent 측에서 실행되는 도구 인터페이스. AI 요청을 on-chain 조회/트랜잭션으로 변환한다.

핵심 가치는 **permissionless listing + verifiable discovery**이며, 결제는 이와 분리된 선택형 모듈이다.

---

## 기술 스택

| 계층 | 기술 | 역할 |
|------|------|------|
| AI 에이전트 | Claude, GPT 등 | 사용자 대신 쇼핑 수행 |
| MCP Server | TypeScript / Node.js | MCP 도구 -> Solana 트랜잭션 변환 (Agent-local) |
| Off-chain DB | Redis / SQLite | 개인정보 원본, 세션 캐시, idempotency 저장 (Agent-local) |
| Blockchain | Solana + Anchor (Rust) | 커머스 로직 on-chain 실행 |
| 결제 모듈 | USDC/SOL (SPL Token) | discovery/등록 계층과 분리된 선택형 settlement |
| 메타데이터 | Arweave | 상품 이미지, 상세 설명 영구 저장 |
| 크랭커 | Clockwork / 자체 Keeper | 에스크로 자동 릴리스 트리거 |

---

## 3분 개념 지도

이 시스템의 본체는 Solana Program이다. 상점/상품/주문의 "신뢰 데이터"는 여기에 기록된다.
MCP Server는 AI Agent 측에서 실행되는 도구 인터페이스로, AI 요청을 받아 on-chain 조회/트랜잭션으로 변환한다 (중앙 서버 아님).
Off-chain DB는 Agent 로컬에서 개인정보 원본/세션/멱등성 같은 운영 데이터를 저장한다.
Arweave는 이미지/상세설명 같은 대용량 메타데이터를 저장한다.
Payment는 붙일 수 있는 모듈이며, 핵심 가치(등록/탐색 신뢰)와 분리된다.

```mermaid
flowchart TB
  U["사용자"] --> A["AI 에이전트"]
  A --> M["MCP Server (Agent-local 툴 집합)"]
  M --> S["Solana Program (신뢰 데이터)"]
  M --> D["Off-chain DB (Agent-local: PII/세션/멱등성)"]
  M --> W["Arweave (이미지/상세설명)"]
  S --> P["결제 모듈 (선택)"]
  S --> E["EncryptedBuyerInfo PDA (임시 암호화 PII)"]
  M -.-> I["PII Curator (선택: Merchant 편의)"]
```

---

## 사용자 여정

```mermaid
flowchart TB
  R1["1 discover_merchant / search_products"] --> R2["2 create_checkout"]
  R2 --> R3["3 complete_checkout (선택 결제 모듈)"]
  R3 --> R4["4 get_order / fulfillment 추적"]
```

---

## 데이터 3계층 원칙

1. 상점/상품 discovery에 필요한 데이터 정합성을 최우선으로 유지한다.
2. 신뢰가 필요한 최소 데이터만 on-chain에 두고, 대용량/민감 데이터는 off-chain에 둔다.
3. payment 필드는 선택형 모듈로 취급하며 registry/discovery 계층과 분리한다.
4. PII는 주문별 임시 키로 암호화하여 on-chain에 임시 저장하고, Merchant 수신 확인 후 PDA를 닫는다.
5. PII 암호문의 on-chain 노출 기간을 최소화하여 미래 키 유출 위험을 줄인다.
6. Curator Badge(attestation)는 on-chain에 저장하여 AI Agent가 즉시 검증 가능하게 한다. Curator가 직접 TX에 서명하므로 Merchant 위조 불가.

```mermaid
flowchart TB
  subgraph OC["On-chain (Solana)"]
    direction TB
    OC1["신뢰 데이터<br/>가격/재고, 체크아웃 상태/금액<br/>에스크로, 주문/배송 이벤트<br/>환불/조정, 상점 프로필, 판매 통계<br/>Curator Badge(attestation)"]
    OC2["PII 해시 (영구)<br/>email_hash, name_hash 등<br/>→ 검증용, 복호화 불가"]
    OC3["암호화된 PII (임시)<br/>EncryptedBuyerInfo PDA<br/>→ Merchant 수신 확인 후 PDA 닫기"]
    OC4["저장 형태: PDA Account<br/>비용: 계정당 ~0.002-0.007 SOL"]
    OC1 --- OC2 --- OC3 --- OC4
  end

  OC4 -->|"Arweave URI 참조"| AR1

  subgraph AR["Off-chain (Arweave)"]
    direction TB
    AR1["대용량/저변경 데이터<br/>상품 이미지, 상세 설명<br/>로고/배너, 약관 전문"]
    AR2["참조: on-chain에 URI 저장<br/>비용: 1MB당 ~$0.005 (1회 영구)"]
    AR1 --- AR2
  end

  OC2 -.->|"해시 무결성 검증"| DB3

  subgraph DB["Off-chain (Agent 로컬 Redis / SQLite)"]
    direction TB
    DB1["PII 원본 (Agent 측)<br/>이메일/전화/실명/배송 주소"]
    DB2["세션/임시 데이터<br/>Cart, continue_url<br/>세션키/서명 캐시<br/>멱등성 키 → 응답 캐시"]
    DB3["데이터 연결<br/>checkout_id → buyer_data"]
    DB4["영속성: Redis(AOF) 또는 SQLite<br/>GDPR 삭제 대응 가능<br/>(on-chain 해시만 잔존)"]
    DB1 --- DB2 --- DB3 --- DB4
  end
```

---

## MCP Server 운영 주체

MCP Server는 AI Agent 측에서 실행되며, Off-chain DB도 Agent 로컬에 위치한다.
따라서 **별도 중앙 서버 운영자가 불필요**하다.

```
AI Agent (각자 자기 것)
├── MCP Server        ← Agent가 직접 실행
├── Off-chain DB      ← Agent 로컬 (Redis/SQLite)
└── Arweave Client    ← 조회만

공유 인프라 (운영자 없음)
├── Solana            ← 퍼블릭 블록체인
├── Arweave           ← 퍼블릭 영구 저장
└── PII Curator (선택) ← 복수 운영 가능, Merchant 편의 제공
```

---

## PII 핵심 흐름: Ephemeral Key + PDA Closure

### 설계 동기

| 문제 | 설명 |
|------|------|
| MCP Server가 Agent 측에 있음 | Merchant가 Agent의 Off-chain DB에 직접 접근 불가 |
| Agent 오프라인 가능성 | Buyer가 주문 후 에이전트를 종료할 수 있음 |
| 탈중앙화 유지 | Merchant에게 별도 서버 운영을 강제하지 않음 |
| 보안 | on-chain에 PII 평문 저장 불가, 영구 암호문도 위험 |

### 핵심 원칙

1. **on-chain 암호문은 임시**: Merchant 수신 확인 후 PDA를 닫아 active state에서 제거
2. **주문별 임시 키**: Merchant 장기 키 유출 시에도 이미 폐기된 ephemeral key로 과거 PII 복호화 불가
3. **이중 방어**: ephemeral key + PDA closure 두 계층으로 보안 확보

### PII 전달 상세 흐름

```
complete_checkout 시 (Agent 측):
  1. ephemeral_keypair = Keypair::generate()
  2. encrypted_pii = encrypt(pii_json, ephemeral_keypair.pubkey)
  3. encrypted_ephemeral_privkey = encrypt(ephemeral_keypair.privkey, merchant.pubkey)
  4. Solana TX: EncryptedBuyerInfo PDA 생성
     {
       order: order_pda,
       encrypted_pii,
       encrypted_ephemeral_key: encrypted_ephemeral_privkey,
       ephemeral_pubkey: ephemeral_keypair.pubkey,
     }

Merchant 수신:
  5. EncryptedBuyerInfo PDA 조회 (on-chain)
  6. ephemeral_privkey = decrypt(encrypted_ephemeral_key, merchant.privkey)
  7. pii_json = decrypt(encrypted_pii, ephemeral_privkey)
  8. 배송 정보 확보 → 자체 DB에 저장

PDA 닫기:
  9. close_buyer_info instruction 호출 (Merchant 서명)
  10. PDA 삭제, rent → Merchant에게 환수
  11. Merchant가 ephemeral_privkey 폐기
```

### 시간축과 노출 구간

```
주문 ──── 배송준비 ──── 배송완료 ──── PDA 닫기
  │                                    │
  ├─ EncryptedBuyerInfo on-chain ──────┤
  │      (노출 구간: 수일~수주)          │
  │                                    └─ PDA 닫힘, active state에서 제거
  │
  └─ CheckoutSession.buyer_*_hash ──────────────── 영구 잔존 (해시만, 복호화 불가)
```

### 위협 분석

| 공격 시나리오 | ephemeral key만 | PDA closure만 | **둘 다 적용** |
|---------------|-----------------|---------------|----------------|
| Merchant 장기 키 유출 | ephemeral key 이미 폐기 -> **안전** | PDA 닫혀있으면 암호문 없음 -> **안전** | **이중 방어** |
| Ledger 히스토리 스캔 | ephemeral key 없이 복호화 불가 -> **안전** | 닫힌 PDA는 active state에 없음 | **이중 방어** |
| PDA 닫기 전 탈취 | Merchant 장기 키 + ephemeral key 동시 필요 -> **난이도 높음** | 아직 열려있어 조회 가능 | ephemeral key 보호 |
| Curator 해킹 | 위임 키 범위만 노출 | Curator 캐시만 위험 | on-chain과 분리 |

### 잔존 위험과 수용 범위

- **PDA가 열려 있는 기간** (수일~수주): ephemeral privkey + Merchant 장기 키가 동시에 유출되어야 복호화 가능. 현실적 위험도 매우 낮음.
- **과거 ledger 히스토리**: Validator pruning 대상. Archival node에 남을 수 있지만 ephemeral key 이미 폐기되어 복호화 불가.
- **Merchant가 PDA를 안 닫는 경우**: 인센티브 설계로 대응 (rent 환수 = 경제적 동기). 필요 시 TTL 강제 (keeper가 일정 기간 후 자동 close).

### PII Curator (선택적 보조 계층)

Merchant가 직접 on-chain PDA를 읽고 복호화할 수 있지만,
운영 편의를 위해 **PII Curator**를 선택적으로 활용할 수 있다.

```
Curator 역할:
├── on-chain EncryptedBuyerInfo 이벤트 감지
├── Merchant 위임 키로 복호화 → 운영 DB에 캐시
├── Merchant에게 조회 API 제공 (배송지, 고객 목록, 검색)
├── GDPR 삭제 요청 시 캐시에서 삭제
└── Webhook 브릿지 역할 병행 가능

Curator 장애 시 fallback:
├── Merchant가 Solana PDA 직접 조회 → 자기 키로 복호화
└── on-chain이 원본이므로 데이터 유실 없음

탈중앙화:
├── 복수 Curator 운영 가능 (경쟁/선택)
├── Merchant가 Curator를 자유롭게 선택/변경
├── 프로토콜은 Curator에 중립
└── 특정 Curator에 종속되지 않음
```

---

## Buyer PII 관리 흐름

### Checkout 단계 (Agent 측)

```mermaid
flowchart TB
  subgraph C["create_checkout"]
    C1["buyer.email/name/phone 수신"]
    C2["Agent Off-chain DB 저장: checkout_id -> buyer_data"]
    C3["On-chain에는 해시만 저장<br/>buyer_email_hash, buyer_name_hash"]
    C4["프로그램이 해시 존재로 필수 필드 검증"]
    C1 --> C2 --> C3 --> C4
  end

  subgraph G["get_checkout"]
    G1["On-chain CheckoutSession 조회"]
    G2["Agent Off-chain DB에서 buyer 원본 조회"]
    G3["SHA-256(원본) == on-chain 해시 검증"]
    G4["UCP 응답에 원본 buyer 데이터 포함"]
    G1 --> G2 --> G3 --> G4
  end
```

### 주문 완료 단계 (PII를 Merchant에게 전달)

```mermaid
sequenceDiagram
  participant A as Buyer Agent (MCP Server)
  participant S as Solana Program
  participant I as PII Curator (선택)
  participant M as Merchant

  Note over A,S: complete_checkout 시
  A->>A: 1. 주문별 ephemeral keypair 생성
  A->>A: 2. PII를 ephemeral pubkey로 암호화
  A->>A: 3. ephemeral privkey를 Merchant pubkey로 암호화
  A->>S: 4. EncryptedBuyerInfo PDA 생성<br/>(암호화된 PII + 암호화된 ephemeral privkey)

  Note over M,S: Merchant가 PII 수신
  M->>S: 5. EncryptedBuyerInfo PDA 조회
  M->>M: 6. 자기 privkey로 ephemeral privkey 복호화
  M->>M: 7. ephemeral privkey로 PII 복호화
  M->>M: 8. 배송 정보 확보

  Note over M,S: PDA 닫기 (암호문 제거)
  M->>S: 9. close_buyer_info 호출 (Merchant 서명)
  S-->>M: 10. PDA 삭제, rent 환수
  M->>M: 11. ephemeral privkey 폐기

  Note over I,S: Curator 경유 (선택적)
  I->>S: EncryptedBuyerInfo 이벤트 감지
  I->>I: Merchant 위임 키로 복호화 → 캐시
  M->>I: 편의 API로 PII 조회 (대안 경로)
```

### GDPR 삭제

```mermaid
flowchart TB
  subgraph R["GDPR 삭제 요청"]
    R1["Agent Off-chain DB에서 buyer 원본 삭제"]
    R2["Merchant에게 삭제 요청 전달"]
    R3["Curator에게 캐시 삭제 요청"]
    R4["On-chain 해시는 잔존 (복호화 불가)"]
    R5["EncryptedBuyerInfo PDA는 이미 닫혀있음<br/>(close_buyer_info로 삭제 완료)"]
    R6["ephemeral key 이미 폐기됨"]
    R1 --> R2 --> R3 --> R4 --> R5 --> R6
  end
```

---

## Order Webhook 브릿지

UCP는 상점이 `webhook_url`로 주문 이벤트를 Platform에 push하는 패턴을 사용한다.
Solana에서는 on-chain 이벤트를 감지하여 webhook으로 변환하는 브릿지가 필요하다.

```mermaid
sequenceDiagram
  participant P as Platform
  participant D as Off-chain DB
  participant M as MCP Server (Webhook Bridge)
  participant S as Solana Program
  participant Q as DLQ

  P->>D: order_id -> webhook_url 등록
  M->>S: WebSocket 이벤트 구독
  S-->>M: FulfillmentEvent / EscrowReleased / RefundProcessed
  M->>S: Order + 관련 PDA 조회
  M->>P: POST webhook_url (UCP Order JSON)
  alt 전송 실패
    M->>M: 최대 5회 재시도 (지수 백오프)
    M->>Q: 최종 실패 이벤트 기록
  end
```

---

## 관련 문서

- [Solana Program 설계](solana-program.md) -- Account 구조체, Instructions, 상태 전이
- [MCP Server 설계](mcp-server.md) -- 도구 목록, create_checkout/complete_checkout/get_order 상세
- [Payment 설계](payment.md) -- sol.usdc Payment Handler, 에스크로, 크랭커
