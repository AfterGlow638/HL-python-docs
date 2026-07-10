# SK hynix ADS/ADR 및 미국 세션 모니터링 가이드

> 버전: 2026-07-10  
> 목적: SK hynix 미국 상장 이후 Afternoon/Night/Morning Brief에서 미국 가격발견을 한국 원주와 연결해 해석한다.

## 1. 용어

- **Common share**: KRX KOSPI에 상장된 SK하이닉스 보통주 `000660`
- **ADS**: 미국 시장에서 거래되는 American Depositary Share
- **ADR**: ADS를 증명하는 예탁증서. 시장 대화에서는 ADS와 ADR이 혼용되지만 수치 계산에서는 공식 문서의 ADS 조건을 사용한다.
- **Regular-way symbol**: 공식 Nasdaq 거래 심볼
- **When-issued symbol**: 신규 상장·기업행사 과정에서 일시적으로 사용될 수 있는 조건부 심볼. 공식 현재 상태가 확인될 때만 사용한다.

## 2. 현재 기준과 매회 재검증

2026-07-10 기준 공식 Nasdaq Trader 공지는 다음 전환을 예상한다.

| 단계 | 예상 미국 거래일 | 심볼 | 주의사항 |
|---|---|---|---|
| when-issued | 2026-07-10 | `SKHYV` | 신규상장 개시 절차·halt cross 때문에 프리마켓 체결이 없을 수 있음 |
| regular-way | 2026-07-13 | `SKHY` | 공식 심볼 변경과 이력 승계 여부를 재확인 |
| expected settlement for when-issued trades | 2026-07-14 | 해당 거래 기준 | 결제일과 거래일을 혼동하지 않음 |

공식 공지:

- Nasdaq Data Technical News #2026-11:
  https://www.nasdaqtrader.com/TraderNews.aspx?id=DTN2026-11
- Nasdaq Equity Trader Alert #2026-37:
  https://www.nasdaqtrader.com/TraderNews.aspx?id=ETA2026-37

매회 확인할 공식 기준점:

- Nasdaq/Nasdaq Trader current symbol and trading status
- SEC F-1/F-1A offering registration
- SEC F-6 ADR registration
- SK hynix IR disclosure
- KRX common share: `000660`

예정일이 지났다는 이유만으로 심볼 전환이나 거래 개시를 완료 상태로 간주하지 않는다. 상장 첫날이나 전환 기간에는 다음을 매회 확인한다.

```text
listing_phase:
- pre_listing
- when_issued
- regular_way
- halted
- unavailable
```

일부 보도나 데이터 사업자가 when-issued 심볼을 별도로 표시할 수 있다. 공식 Nasdaq/SEC/거래상태에서 확인되지 않으면 심볼을 추정하지 않는다.

## 3. Tier 1 공식 링크

- Nasdaq Trader transition notice:
  https://www.nasdaqtrader.com/TraderNews.aspx?id=DTN2026-11
- Nasdaq Trader equity alert:
  https://www.nasdaqtrader.com/TraderNews.aspx?id=ETA2026-37
- Nasdaq regular-way quote page:
  https://www.nasdaq.com/market-activity/stocks/skhy
- Nasdaq symbol directory:
  https://www.nasdaqtrader.com/Trader.aspx?id=symbollookup
- SEC F-1/A filing index, 2026-07-06:
  https://www.sec.gov/Archives/edgar/data/2120882/000119312526295501/0001193125-26-295501-index.htm
- SEC F-6 filing index, 2026-07-01:
  https://www.sec.gov/Archives/edgar/data/2120882/000119380526000898/0001193805-26-000898-index.htm
- SK hynix disclosures:
  https://www.skhynix.com/ir/UI-FR-IR12_T1/
- SK hynix KRX share-price page:
  https://www.skhynix.com/ir/UI-FR-IR02/
- KRX/KIND:
  https://kind.krx.co.kr/

Nasdaq 페이지에 `currently not trading`, `halted`, 데이터 없음, 또는 신규종목 개시 대기가 표시되면 가격을 추정하지 않는다. `SKHYV`가 when-issued로 거래되는 동안 `SKHY` 페이지가 비거래 상태인 것은 오류가 아닐 수 있다.

## 4. ADS 비율과 환산

현재 공모 문서의 비율을 공식 최신 prospectus에서 확인한 뒤 저장한다.

2026-07-10 기준 공모 조건으로 알려진 구조:

```text
1 ADS = 0.1 KRX common share
10 ADS = 1 KRX common share
```

비율은 기업행사로 변경될 수 있으므로 영구 상수로 취급하지 않는다.

### 4.1 KRX 종가 기반 이론 ADS 환산값

```text
krx_implied_ads_usd
= krx_common_price_krw
  × common_shares_per_ads
  ÷ usd_krw
```

현재 비율이 `0.1 common share per ADS`로 공식 검증됐다면:

```text
krx_implied_ads_usd
= krx_common_price_krw × 0.1 ÷ usd_krw
```

### 4.2 표시 괴리

```text
adr_vs_krx_indicative_gap_pct
= adr_price_usd / krx_implied_ads_usd - 1
```

이 값은 **표시상 괴리**다. 다음 이유로 무위험 차익거래 프리미엄으로 표현하지 않는다.

- 한국과 미국의 거래시간 불일치
- FX timestamp 불일치
- 발행·취소와 예탁은행 절차
- 결제주기와 시장 접근 제한
- 세금·수수료·예탁수수료
- 상장 초기 안정화 거래와 낮은 유동성
- KRX 원주와 ADS 사이의 신규 발행·희석 구조
- NXT와 KRX 가격 차이

권장 문구:

> 동일 ADS 비율과 환율로 환산한 KRX 종가 대비 표시상 괴리가 확대됐다. 세션·결제·유동성 차이가 있어 직접 차익거래 가능성을 의미하지 않는다.

## 5. 세션 분리

미국 가격은 반드시 다음으로 분리한다.

```text
when_issued_pre_market
when_issued_new_issue_cross
when_issued_regular_session
regular_way_pre_market
regular_way_regular_session
after_hours
not_yet_trading
halted
```

- Afternoon 15:55 KST에는 미국 프리마켓이 통상 시작 전이다. 실제 session status를 확인한다.
- Night 20:25 KST에는 프리마켓이 통상 진행 중이지만, 신규 상장 종목은 Nasdaq의 new-issue opening process 전까지 체결이 없을 수 있다.
- Morning 07:45 KST에는 미국 정규장 종가와 애프터마켓을 분리한다.
- 미국 서머타임과 휴장·조기폐장을 확인한다.
- `SKHYV`와 `SKHY`를 동시에 두 개의 독립 가격 확인 신호로 세지 않는다.

## 6. 상장 초기 5거래일 규칙

상장 직후에는 5일 패널을 강제로 채우지 않는다.

```text
available_regular_sessions: 0..5
history_status: partial / complete
```

첫 5개 정규장에서는 다음을 우선한다.

- offering/reference price
- 첫 거래가
- 정규장 OHLCV
- close location
- VWAP, 검증된 분봉이 있을 때만
- premarket/regular/after-hours 분리
- spread와 거래량
- KRX 환산값 대비 표시 괴리
- MU/SNDK/DRAM/SOX/NQ 대비 상대강도

상장 초기에는 underwriter stabilization, 배정 물량, 신규 지수·ETF 편입 기대, 가격발견이 섞일 수 있으므로 SKHY 움직임을 곧바로 SK하이닉스 펀더멘털 변화로 해석하지 않는다.

## 7. WTS와 Tier 2 사용

TossInvest WTS에 신규 미국 종목이 즉시 등록된다고 가정하지 않는다.

권장 절차:

```text
1. SKHY와 가능한 현재 심볼을 resolver/공개 종목 검색으로 확인
2. name, symbol, market, status 검증
3. WTS가 반환한 opaque productCode만 us-s c-chart에 사용
4. display ticker를 c-chart productCode로 직접 넣지 않음
5. WTS 미등록이면 Nasdaq Tier 1 → Investing.com/TradingView Tier 2 순서
```

WTS 응답이 과거 OTC/GDR 프로그램을 가리키는지 확인한다. 신규 Nasdaq ADS와 기존 Luxembourg GDR/OTC 표시는 서로 다른 상품일 수 있다.

## 8. Night Brief 필수 SKHY 패널

가능한 경우 다음을 조사한다.

| 필드 | 규칙 |
|---|---|
| listing phase | pre-listing/when-issued/regular-way/halted |
| current symbol | 공식 현재 심볼만 사용 |
| session | premarket/regular/after-hours |
| last price | timestamp와 통화 표시 |
| bid/ask | 표시 시각과 spread |
| volume | premarket volume과 정규장 volume 분리 |
| high/low | 해당 세션 범위 |
| offering/reference price | 현재 거래가격과 별도 필드 |
| ADS ratio | 공식 prospectus 기준 |
| KRX implied ADS | KRX 종가, 비율, USD/KRW 타임스탬프 표시 |
| NXT implied ADS | NXT 최종가가 검증될 때만 별도 계산 |
| relative strength | MU, SNDK, DRAM, SMH/SOXX, NQ와 비교 |
| data quality | 신규상장·얕은 유동성·세션 혼합 경고 |

## 9. 인과 해석 규칙

### SKHY 강세 + MU/SNDK/DRAM 강세

메모리 업종 전반의 위험선호와 일치할 수 있다. 다만 신규상장 수요가 별도로 섞였을 수 있다.

### SKHY 강세 + MU/SNDK 약세

상장 배정·가격발견·미국 투자자 접근성 같은 SKHY 고유 요인일 수 있다. 한국 원주의 다음날 방향으로 일반화하지 않는다.

### SKHY 약세 + KRX 원주 강세

공모가 부담, 희석 우려, 세션/FX 차이, 신규상장 매물 소화 가능성을 확인한다. 펀더멘털 악화로 단정하지 않는다.

### SKHY 표시 괴리 확대 + 낮은 거래량

신뢰도 낮은 가격발견일 수 있다. 거래량·spread·정규장 지속성을 확인한다.

## 10. Confidence cap

- listing status 또는 현재 symbol 불명: SKHY 기반 해석 최대 `0.45`
- premarket만 있고 volume/spread 미확인: 최대 `0.50`
- when-issued 데이터만 사용: 최대 `0.55`
- 상장 후 첫 5개 정규장: SKHY 교차시장 추론 최대 `0.60`
- ADS ratio 또는 USD/KRW timestamp 불명: 괴리 계산 금지
- Nasdaq와 WTS/Investing 값 충돌 미해결: 최대 `0.40`

## 11. Source Audit 예시

```json
{
  "source_name": "Nasdaq SKHY quote",
  "tier": 1,
  "url": "https://www.nasdaq.com/market-activity/stocks/skhy",
  "instrument": "SK hynix ADS",
  "symbol": "SKHY",
  "listing_phase": "",
  "session": "pre_market",
  "market_date": "",
  "source_timestamp": "",
  "fields_used": [],
  "limitations": []
}
```
