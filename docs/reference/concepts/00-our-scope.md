# 00. UCP 적용 범위 가이드 (Registry-first)

이 문서는 `concepts/`의 원본 UCP 설명을 우리 프로젝트 관점에서 해석한 적용 범위를 정의한다.

---

## 독해 가이드

- 이 문서의 목표:
concepts 문서를 읽을 때의 `해석 룰`을 먼저 고정한다.
- 지금 몰라도 되는 것:
개별 결제/보안 프로토콜 세부
- 여기서 꼭 잡을 것:
무엇이 지금 범위이고, 무엇이 later scope인지

## 목표 재정의

우리의 1차 목표는 결제 최적화가 아니라 아래 3가지다.

1. Permissionless Listing: 누구나 상점/상품 등록 가능
2. Verifiable Discovery: UCP 기반 표준 discovery + 검증 가능한 원본 데이터
3. Trust by Transparency: 온체인 이력으로 신뢰 확보

결제(`sol.usdc`, `sol.native`)는 분리된 모듈로 점진 도입한다.

---

## concepts 문서 해석 규칙

| 문서 | 프로젝트 적용 우선순위 | 적용 방식 |
|------|------------------------|-----------|
| `01-overview.md` | 높음 | UCP 전체 구조 참조, 버전/스키마 호환성 기준으로 사용 |
| `02-payment-flow.md` | 중간 | checkout/order 흐름은 Phase 4+에서 순차 도입 |
| `03-ap2-security.md` | 중간 | Option A/B에서 유지, Full on-chain 범위에서는 선택적 |
| `04-rest-vs-mcp.md` | 높음 | MCP 바인딩 규칙, meta/idempotency 매핑 기준으로 사용 |

---

## 현재 범위 (In Scope)

1. MerchantProfile/ProductListing 온체인 등록/조회
2. UCP discovery 응답 정합성 유지
3. 검색/인덱싱/메타데이터 무결성 검증
4. 스팸/시빌 방어를 위한 정책 설계(프로토콜 중립 유지)
5. **Buyer PII 암호화 전달** (주문별 ephemeral key + EncryptedBuyerInfo PDA + PDA closure)
6. **MCP Server Agent-local 실행** (중앙 서버 불필요 구조)

## 차후 범위 (Later Scope)

1. Checkout full replacement 오케스트레이션 고도화
2. Escrow/환불/분쟁 자동화
3. 결제 모듈 확장(`sol.usdc` 외 추가 핸들러)
4. PII Relay 고도화 (복수 Curator 경쟁, 위임 키 프로토콜 표준화)

---

## 범위 경계 원칙

1. 프로토콜은 중립 유지, 정책은 Curator/UI 계층에서 적용
2. PII 원본은 Agent-local DB, on-chain은 해시(영구) + 암호화된 PII(임시, PDA closure로 제거)
3. 모듈 간 결합 최소화: registry/discovery와 payment는 독립적으로 진화
4. PII 암호문의 on-chain 노출 기간을 최소화 (ephemeral key + PDA closure)
