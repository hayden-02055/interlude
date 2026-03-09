# sol.usdc Payment Handler 정의

> [05-solana-program.md](05-solana-program.md) ← 이전 | 다음 → [07-implementation-plan.md](07-implementation-plan.md)

---

## 독해 가이드

- 이 문서의 목표:
`sol.usdc`를 UCP payment handler로 어떻게 선언/교환하는지 고정한다.
- 지금 몰라도 되는 것:
추가 핸들러(`sol.native` 등) 확장 전략
- 여기서 꼭 잡을 것:
discovery 선언, checkout `payment_handlers`, complete 시 instrument 스키마

UCP payment handler로서 `sol.usdc`를 정의한다. 기존 `com.google.pay`, `com.stripe` 등과 동일한 레벨에서 작동한다.

> **이 문서가 sol.usdc config의 단일 기준(canonical source)입니다.**
> 다른 문서(04-mcp-server.md, design-vision.md 등)의 config 예시는 이 문서의 스키마를 따릅니다.

## Discovery에서의 선언

```json
{
  "services": {
    "dev.ucp.shopping": [{
      "transport": "mcp",
      "version": "2026-01-11",
      "endpoint": "https://ucp-solana-mcp.example.com",
      "spec": "https://ucp-solana-mcp.example.com/openrpc.json",
      "schema": "https://ucp-solana-mcp.example.com/schema.json"
    }]
  }
}
```

## Checkout 응답의 payment_handlers

```json
{
  "payment_handlers": {
    "sol.usdc": [{
      "id": "usdc_escrow",
      "version": "2026-01-11",
      "config": {
        "network": "mainnet-beta",
        "program_id": "UCPxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
        "usdc_mint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
        "escrow_type": "program_controlled",
        "auto_release_days": 14
      }
    }]
  }
}
```

## Payment Instrument 구조 (complete_checkout 시)

```json
{
  "payment": {
    "instruments": [{
      "id": "instr_usdc_1",
      "handler_id": "usdc_escrow",
      "type": "crypto_wallet",
      "selected": true,
      "credential": {
        "type": "solana_wallet",
        "wallet_address": "BuyerWallet111...xxx"
      },
      "display": {
        "brand": "solana",
        "label": "USDC Wallet",
        "last_digits": "...xxx"
      }
    }]
  }
}
```

**Tokenization 불필요**: 기존 UCP에서는 카드번호를 토큰화하지만, Solana에서는 지갑 주소가 공개 정보이므로 토큰화 과정이 필요 없다. 지갑 서명 = 결제 승인.
