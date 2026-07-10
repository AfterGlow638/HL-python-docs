# Brief Handoff Contract

> 버전: 2026-07-10  
> 목적: Morning → Afternoon → Night Brief 사이에서 사전 가설과 사후 결과를 비교하기 위한 최소 상태 계약

## 1. 기본 원칙

Scheduled Task가 다른 Task의 출력을 자동으로 기억하거나 검색할 수 있다고 가정하지 않는다.

이 계약은 다음 중 하나로 이전 출력이 전달될 때 사용한다.

- 같은 대화에 이전 브리핑이 존재
- 연결된 Drive/Slack/DB/Streamlit API에서 이전 JSON을 읽음
- 사용자가 이전 브리핑 JSON을 프롬프트에 첨부
- 외부 수집기가 `previous_brief_context`를 제공

접근할 수 없으면 다음처럼 기록한다.

```json
{
  "previous_brief_access": "unavailable",
  "comparison_mode": "not_performed"
}
```

이전 결과를 현재 데이터로 재구성하면 다음처럼 기록한다.

```json
{
  "previous_brief_access": "unavailable",
  "comparison_mode": "reconstructed",
  "warning": "This is not a direct audit of the original brief."
}
```

## 2. 비교 상태 표준

| 상태 | 의미 |
|---|---|
| `confirmed` | 사전에 선언된 trigger와 confirmation이 데이터로 충족됨 |
| `partially_confirmed` | 일부 조건만 충족되거나 세션 중 일시적으로 충족됨 |
| `invalidated` | 사전에 선언된 invalidation이 충족됨 |
| `not_triggered` | trigger 자체가 발생하지 않음 |
| `not_testable` | 필요한 가격·수급·시간대 데이터가 없음 |
| `data_conflict` | 출처 또는 세션이 충돌해 판정 불가 |
| `reconstructed_only` | 원본 브리핑 없이 현재 데이터로만 사후 재구성 |

사후 수익률만 보고 `confirmed`로 판정하지 않는다. 반드시 사전 trigger·confirmation·invalidation과 비교한다.

## 3. Morning → Afternoon 입력

권장 최소 JSON:

```json
{
  "brief_id": "morning-YYYY-MM-DD",
  "as_of_kst": "",
  "brief_type": "morning",
  "market_bias": "",
  "confidence": 0.0,
  "sessions_used": {
    "korea": "",
    "united_states": ""
  },
  "top_drivers": [],
  "opening_expectations": {
    "sk_hynix": "",
    "samsung_electronics": "",
    "kospi200": ""
  },
  "key_levels": {
    "sk_hynix": [],
    "samsung_electronics": [],
    "kospi200": []
  },
  "long_scenario": {
    "trigger": "",
    "confirmation": "",
    "invalidation": ""
  },
  "short_scenario": {
    "trigger": "",
    "confirmation": "",
    "invalidation": ""
  },
  "no_trade_conditions": [],
  "missing_data": []
}
```

Afternoon Brief는 다음만 평가한다.

- 갭 방향이 예상과 일치했는가
- 초기 30분과 종가 구조가 trigger/confirmation을 충족했는가
- invalidation이 발생했는가
- 오전에 누락된 데이터가 본장 중 확인됐는가
- 시나리오 방향은 맞았지만 진입 조건은 나빴는가

## 4. Afternoon → Night 입력

권장 최소 JSON:

```json
{
  "brief_id": "afternoon-YYYY-MM-DD",
  "as_of_kst": "",
  "brief_type": "afternoon",
  "market_bias_into_evening": "",
  "confidence": 0.0,
  "morning_scorecard": [],
  "korea_close": {
    "sk_hynix": "",
    "samsung_electronics": "",
    "kospi200": "",
    "close_quality": ""
  },
  "current_flow_status": {
    "core_stocks": "final/provisional/unavailable",
    "single_stock_etfs": "final/provisional/unavailable",
    "kospi200_derivatives": "final/provisional/unavailable"
  },
  "key_evening_levels": {
    "krx_close": [],
    "nxt": [],
    "krx_night_futures": [],
    "us_premarket": []
  },
  "long_scenario": {
    "trigger": "",
    "confirmation": "",
    "invalidation": ""
  },
  "short_scenario": {
    "trigger": "",
    "confirmation": "",
    "invalidation": ""
  },
  "no_trade_conditions": [],
  "missing_data": []
}
```

Night Brief는 다음을 평가한다.

- NXT 최종값이 KRX 종가 해석을 확인했는가
- 18시 이후 KRX 야간파생이 Afternoon 시나리오를 확인했는가
- 미국 프리마켓과 SK hynix ADS/ADR가 같은 방향인가
- Afternoon 시점 잠정 수급이 최종값에서 바뀌었는가
- 다음날 가설을 바꿀 만큼 새로운 정보가 발생했는가

## 5. Night → Next Morning 입력

Night Brief는 다음날 Morning Brief가 읽을 수 있도록 최소한 다음을 남긴다.

```json
{
  "brief_id": "night-YYYY-MM-DD",
  "as_of_kst": "",
  "next_krx_bias": "",
  "confidence": 0.0,
  "nxt_final": "",
  "krx_night_derivatives_at_brief": "",
  "skhy_adr_premarket": "",
  "us_premarket_memory_group": "",
  "key_events_before_us_open": [],
  "next_morning_checks": [],
  "conditions_that_would_override_this_brief": []
}
```

다음날 Morning Brief는 미국 정규장 종가가 Night Brief의 프리마켓 가설을 확인했는지 별도로 평가한다.

## 6. Hindsight-bias 방지

금지:

- 결과가 상승했으므로 Morning long scenario가 맞았다고 판정
- 장중 한 번 trigger를 통과했지만 즉시 invalidation된 사실을 생략
- 이전 브리핑에 없던 수준을 사후에 핵심 기준으로 추가
- 원본 브리핑을 읽지 못했는데 정확히 비교한 것처럼 작성

권장:

> Morning long trigger는 장 초반 충족됐지만 10시 12분에 invalidation이 발생해 `partially_confirmed`로 판정한다.

> 원본 Morning Brief 출력에 접근할 수 없어 직접 성과평가는 수행하지 않았다. 현재 데이터로 재구성한 가설과 실제 장 흐름만 비교한다.

## 7. 소스 감사

이전 브리핑의 숫자를 현재 소스에서 다시 확인할 때는 다음을 별도로 기록한다.

- original brief value
- current re-fetched value
- source and timestamp
- mismatch reason
- whether comparison remains valid
