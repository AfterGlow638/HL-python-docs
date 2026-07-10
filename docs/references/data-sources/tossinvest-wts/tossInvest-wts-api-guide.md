# TossInvest WTS 공개 API 사용 가이드

> 용도: 한국 반도체 Morning / Afternoon / Night Brief의 공개 웹 데이터 수집
> 버전: 2026-07-10
> 상태: **비공식·무보증·읽기 전용**
> 원본 카탈로그: [TossInvest Web API Catalog](https://github.com/dd3ok/tossinvest-api-skill/blob/main/references/api-catalog.md)

## 1. 문서의 목적

이 문서는 TossInvest 웹페이지가 공개적으로 사용하는 WTS 엔드포인트 중, 반도체 브리핑에 필요한 최소 범위만 정리한다.

주요 사용 목적은 다음과 같다.

- SK하이닉스·삼성전자 및 단일종목 ETF의 전일 종가와 최근 5거래일 OHLCV
- 한국 종목·ETF의 외국인·기관·개인 일별 수급
- 프로그램 매매, 대차거래, 공매도 등 보조 자료
- Micron·Sandisk·Nvidia·AMD·Broadcom·TSMC 및 미국 반도체 ETF의 최근 5거래일 OHLCV
- KOSPI·KOSDAQ·SOX·NQ·미 국채금리·WTI·USD/KRW·BTC/ETH의 보조 시장 지도
- 종목 공시·뉴스의 후보 수집

복잡한 화면 API, 스크리너, 테마, 커뮤니티, 개인화 기능은 이 문서에서 다루지 않는다. 필요한 경우 원본 `api-catalog.md`를 참조한다.

## 2. 법적·운영적 위치

WTS 엔드포인트는 토스증권의 공식 개발자 Open API가 아니다. 공개 웹페이지를 비로그인 상태에서 표시하기 위해 관찰된 내부 엔드포인트다.

따라서 다음 원칙을 적용한다.

- **읽기 전용**으로 사용한다.
- 쿠키, 로그인 세션, 인증 헤더, 계좌 식별자, 주문 관련 요청을 사용하지 않는다.
- 대량 크롤링, 고빈도 폴링, 병렬 폭주, 자동 우회, CAPTCHA 우회, 반복 재시도를 하지 않는다.
- `403`, `429`, access denied, challenge, 로그인 요구, 예상하지 못한 응답이 나오면 즉시 중단한다.
- 엔드포인트가 문서와 다르면 현재 공개 브라우저 동작을 다시 확인하기 전까지 `확인 불가`로 처리한다.
- 브리핑에는 “TossInvest WTS 공개 웹 데이터, 비공식”이라고 표시한다.

원본 카탈로그도 엔드포인트가 문서화되지 않았고 변경될 수 있으므로, 소규모·순차적·사용자 주도 요청만 권장한다.

## 3. 호스트와 식별자

### 3.1 기본 호스트

```text
https://wts-info-api.tossinvest.com
```

반도체 브리핑에서 사용하는 가격, 차트, 종목정보, 수급, 프로그램 매매, 공시, 뉴스의 기본 호스트다.

`wts-cert-api.tossinvest.com`에는 캘린더·랭킹 등 공개 화면 데이터가 있으나 민감한 호스트로 취급한다. 이 문서에서는 기본 브리핑에 꼭 필요한 경우만 언급하며, 개인화·계정·커뮤니티 관련 호출은 사용하지 않는다.

### 3.2 주요 식별자

| 식별자 | 예시 | 용도 |
|---|---|---|
| `productCode` | `A000660`, `A005930`, `US19890516001` | 종목 상세, 가격, 차트, 한국 종목 수급 |
| `companyCode` | `000660`, `005930` | 일부 회사 공시·뉴스 API |
| `codes` | `A000660,A005930` | 배치 조회 |
| `indexCode` | `KGG01P`, `QGG01P`, `SOX.NAI`, `RFU.NQc1` | 지수·선물·금리·원자재 |

한국 보통주는 대체로 거래소 코드 앞에 `A`가 붙지만, 신규 알파벳 종목코드 ETF는 반드시 resolver 응답을 확인한다. 미국 종목은 티커 대신 Toss 내부 opaque `productCode`가 필요한 경우가 많다.

## 4. 필수 사용 순서

### 4.1 종목 코드 확인

처음 보는 종목이나 신규 ETF는 다음 resolver를 먼저 사용한다.

```http
GET https://wts-info-api.tossinvest.com/api/v2/stock-infos/code-or-symbol/{codeOrSymbol}
```

예시:

```text
https://wts-info-api.tossinvest.com/api/v2/stock-infos/code-or-symbol/A000660
https://wts-info-api.tossinvest.com/api/v2/stock-infos/code-or-symbol/0193T0
```

응답에서 최소한 다음을 확인한다.

- `code`
- `symbol`
- `name`
- `market`
- `companyCode`
- `status`

**검증 규칙:** 입력한 코드와 응답 종목명이 일치하지 않으면 다음 단계로 진행하지 않는다.

### 4.2 현재·전일 가격 배치 조회

```http
GET /api/v3/stock-prices/details?productCodes={commaSeparatedCodes}
```

예시:

```text
https://wts-info-api.tossinvest.com/api/v3/stock-prices/details?productCodes=A000660,A005930
```

관찰된 주요 필드:

```text
code
exchange
tradeDateTime
open
high
low
close
volume
value
base
changeType
currency
```

이 호출은 여러 종목의 최신 가격 상태를 빠르게 비교하는 데 사용한다. 단, 전일 종가와 최근 5일 구조는 반드시 일봉 차트로 다시 확인한다.

### 4.3 한국 종목 최근 5거래일 OHLCV

```http
GET /api/v1/c-chart/kr-s/{productCode}/day:1
    ?count=5
    &session=all
    &investMode=krx
    &useAdjustedRate=true
```

SK하이닉스:

```text
https://wts-info-api.tossinvest.com/api/v1/c-chart/kr-s/A000660/day:1?count=5&session=all&investMode=krx&useAdjustedRate=true
```

삼성전자:

```text
https://wts-info-api.tossinvest.com/api/v1/c-chart/kr-s/A005930/day:1?count=5&session=all&investMode=krx&useAdjustedRate=true
```

관찰된 캔들 필드:

```text
dt
base
open
high
low
close
volume
amount
```

`count=5`는 최근 5개 캔들을 요청하지만 휴장, 장중 미완료 캔들, 데이터 지연, 응답 정렬 방향을 확인해야 한다. 안정성을 높이려면 `count=7` 또는 `count=10`으로 받고 최신 **완료** 거래일 5개와 그 직전 종가 앵커를 확보한다. 정확한 5세션 수익률에는 5개 세션의 시작 전 종가가 추가로 필요하다.

### 4.4 미국 종목 최근 5거래일 OHLCV

```http
GET /api/v1/c-chart/us-s/{productCode}/day:1
    ?count=5
    &session=all
    &investMode=krx
    &useAdjustedRate=true
```

Micron:

```text
https://wts-info-api.tossinvest.com/api/v1/c-chart/us-s/US19890516001/day:1?count=5&session=all&investMode=krx&useAdjustedRate=true
```

Sandisk:

```text
https://wts-info-api.tossinvest.com/api/v1/c-chart/us-s/NAS0250224006/day:1?count=5&session=all&investMode=krx&useAdjustedRate=true
```

미국 티커 `MU`, `SNDK`, `NVDA`를 `productCode` 자리에 직접 넣지 않는다. 먼저 Toss 종목 페이지 또는 resolver에서 내부 코드를 확인한다.

`session=all`은 관찰된 호출에서 모든 세션을 포함한다. 따라서 미국 일봉의 `close`를 자동으로 정규장 종가라고 부르지 않는다. 정규장 종가가 필요한 경우 메인세션 가격, 공식 거래소 페이지 또는 날짜·세션이 명확한 보조 소스로 교차 확인하고, 애프터마켓 값은 별도 필드로 기록한다.

### 4.5 한국 종목·ETF 최근 5거래일 투자자 수급

```http
GET /api/v1/stock-infos/trade/trend/trading-trend
    ?productCode={productCode}
    &size=5
```

SK하이닉스:

```text
https://wts-info-api.tossinvest.com/api/v1/stock-infos/trade/trend/trading-trend?productCode=A000660&size=5
```

삼성전자:

```text
https://wts-info-api.tossinvest.com/api/v1/stock-infos/trade/trend/trading-trend?productCode=A005930&size=5
```

관찰된 주요 필드:

```text
baseDate
individualsBuyVolume
individualsSellVolume
netIndividualsBuyVolume
foreignerBuyVolume
foreignerSellVolume
netForeignerBuyVolume
institutionBuyVolume
institutionSellVolume
netInstitutionBuyVolume
netFinancialInvestmentBuyVolume
netInsuranceBuyVolume
netOtherFinancialInstitutionsBuyVolume
netTrustBuyVolume
netPrivateEquityFundBuyVolume
netPensionFundBuyVolume
netBankBuyVolume
netOtherCorporationBuyVolume
```

**단위 규칙:** 이 필드들은 기본적으로 원화 금액이 아니라 **주식 수량 또는 거래량 기반 순매수**다. 브리핑에 `억원`으로 바꿔 쓰지 않는다.

`size=5`는 완료된 5거래일을 보장하지 않는다. 장중에는 최신 `baseDate` 행이 `inMarketTime=true`, 일부 `has*` 필드가 false, 또는 매수·매도 수량이 미완성인 상태로 포함될 수 있다. Morning Brief는 최신 완료 거래일을 먼저 확정하고 충분한 행을 요청한 뒤 완료 행 5개만 사용한다. `updatedAt`도 함께 감사 로그에 남긴다.

### 4.6 정확한 날짜 범위의 수급

```http
GET /api/v1/stock-infos/trade/trend/fixed-trading-trend
    ?productCode={productCode}
    &from={YYYY-MM-DD}
    &to={YYYY-MM-DD}
```

예시:

```text
https://wts-info-api.tossinvest.com/api/v1/stock-infos/trade/trend/fixed-trading-trend?productCode=A000660&from=2026-07-03&to=2026-07-09
```

이 호출은 “시장 데이터”가 아니라 날짜 범위의 **투자자 수급 이력**이다. 종가·고가·저가·거래량은 `c-chart`에서 별도로 받아 날짜로 결합한다.

장기간 조회는 응답이 잘릴 수 있으므로 브리핑에서는 최근 5~10거래일 정도의 작은 구간만 사용한다.

## 5. 최근 5거래일 패널 생성법

Morning Brief의 핵심은 가격과 수급을 같은 날짜 기준으로 결합하는 것이다.

### 5.1 권장 절차

1. 최신 완료 KRX 거래일을 확정한다.
2. `c-chart`에서 7~10개 일봉을 요청한다.
3. `trading-trend?size=7` 또는 작은 날짜 범위의 `fixed-trading-trend`를 요청한다.
4. `dt`와 `baseDate`를 `YYYY-MM-DD`로 정규화한다.
5. `inMarketTime=true`, `hasIndividual=false`, 미래 시각, 장중 갱신 행 등 미완료 가능성이 있는 행을 제외하거나 `provisional`로 분리한다.
6. 두 데이터셋의 공통 **완료 거래일**만 inner join한다.
7. 최신 완료일을 포함한 마지막 5개 공통 거래일을 선택하고, 5세션 수익률 계산용으로 가장 오래된 선택일의 직전 종가 앵커를 보존한다.
8. 가격·수급·거래량 지표를 계산한다.

### 5.2 최소 출력 패널

| 날짜 | 종가 | 일간 등락률 | 거래량 | 외국인 순매수 | 기관 순매수 | 개인 순매수 |
|---|---:|---:|---:|---:|---:|---:|
| D-4 |  |  |  |  |  |  |
| D-3 |  |  |  |  |  |  |
| D-2 |  |  |  |  |  |  |
| D-1 |  |  |  |  |  |  |
| D0 |  |  |  |  |  |  |

### 5.3 권장 파생 지표

```text
1일 수익률 = 최근 종가 / 전일 종가 - 1
5세션 수익률 = 최근 종가 / 가장 오래된 선택 세션의 직전 종가 - 1
              = Π(1 + 각 완료 세션 일간수익률) - 1
최근 거래량 배수 = 최근 거래량 / 직전 4개 완료 세션 평균 거래량
외국인 5일 누적 = Σ netForeignerBuyVolume
기관 5일 누적 = Σ netInstitutionBuyVolume
개인 5일 누적 = Σ netIndividualsBuyVolume
외국인 순매수율 = netForeignerBuyVolume / 당일 거래량
기관 순매수율 = netInstitutionBuyVolume / 당일 거래량
```

수급 절대량은 본주와 ETF 사이에서 직접 비교하지 않는다. 발행좌수·거래량이 크게 다르므로 **당일 거래량 대비 순매수 비율**을 함께 본다.

### 5.4 가격과 수급의 결합 분류

| 가격 | 외국인·기관 수급 | 해석 문구 |
|---|---|---|
| 상승 | 동반 순매수 | 가격과 수급이 같은 방향으로 확인됨 |
| 상승 | 동반 순매도 | 숏커버·개인 추격·리밸런싱 가능성 등 수급 괴리 확인 필요 |
| 하락 | 동반 순매도 | 약세와 매도 압력이 같은 방향일 수 있음 |
| 하락 | 동반 순매수 | 저가 매수 또는 종가 방어 가능성; 다음 세션 확인 필요 |
| 보합 | 큰 순매수·순매도 | 흡수 또는 분배 가능성; 거래량·종가 위치·프로그램 매매 확인 |

이 표는 인과관계를 확정하지 않는다. “영향을 줄 수 있다”, “일치할 수 있다”, “추가 확인이 필요하다”로 표현한다.

## 6. 주요 종목 코드 레지스트리

### 6.1 한국 본주

| 종목 | 거래소 코드 | 검증된 Toss `productCode` |
|---|---|---|
| SK하이닉스 | `000660` | `A000660` |
| 삼성전자 | `005930` | `A005930` |

### 6.2 한국 단일종목 ETF

| 종목 | 거래소 코드 | WTS 사용 규칙 |
|---|---|---|
| KODEX SK하이닉스단일종목레버리지 | `0193T0` | resolver로 Toss `productCode` 확인 후 사용 |
| SOL SK하이닉스단일종목레버리지 | `0197W0` | resolver로 확인 |
| SOL SK하이닉스선물단일종목인버스2X | `0197X0` | resolver로 확인 |
| KODEX 삼성전자단일종목레버리지 | `0193W0` | resolver로 확인 |

신규 알파벳 종목코드는 `A` 접두사를 임의로 붙이지 않는다. `code-or-symbol` 응답의 `code`와 `name`이 일치하는지 확인한 뒤 차트·수급 호출에 사용한다.

### 6.3 미국 핵심 종목·ETF

| 종목 | 티커 | Toss `productCode` |
|---|---|---|
| Micron Technology | `MU` | `US19890516001` |
| Sandisk | `SNDK` | `NAS0250224006` |
| Nvidia | `NVDA` | `US19990122001` |
| AMD | `AMD` | `US20150102001` |
| Broadcom | `AVGO` | `US20090806002` |
| TSMC ADR | `TSM` | `US19971008002` |
| VanEck Semiconductor ETF | `SMH` | `US20191211007` |
| iShares Semiconductor ETF | `SOXX` | `US20010713001` |
| Direxion Semiconductor Bull 3X | `SOXL` | `US20100311002` |
| Direxion Semiconductor Bear 3X | `SOXS` | `US20100311003` |

미국 코드도 상장변경·기업행사·Toss 내부 개편 가능성이 있으므로 resolver 또는 종목 페이지로 재검증한다.

## 7. Morning Brief 필수 수집 범위

### 7.1 한국 핵심 패널

각 종목에 대해 다음을 조사한다.

- 최신 완료 거래일 종가
- 최근 5거래일 OHLCV
- 1일·5일 수익률
- 최근 거래량 배수
- 최근 5거래일 외국인·기관·개인 순매수
- 외국인·기관 수급 방향 일치 여부
- 거래량 대비 외국인·기관 순매수율
- 당일 KRX 종가와 장중 구조
- current-day 수급의 잠정/최종 판정
- 프로그램 매매
- NXT와 분리된 KRX 정규장 분봉
- 미국 프리마켓/애프터마켓 분리
- 신규 Nasdaq 상장 SKHY resolver
- Afternoon → Night 재조회 전략


대상:

- SK하이닉스
- 삼성전자
- KODEX SK하이닉스단일종목레버리지
- SOL SK하이닉스단일종목레버리지
- SOL SK하이닉스선물단일종목인버스2X
- KODEX 삼성전자단일종목레버리지

### 7.2 미국 메모리·반도체 패널

각 종목·ETF에 대해 다음을 조사한다.

- 최신 완료 미국 정규장 종가
- `session=all` 캔들만으로 정규장 종가를 확정하지 않음
- 애프터마켓 표시값이 있으면 정규장과 분리
- 최근 5거래일 OHLCV
- 1일·5일 수익률
- 최근 거래량 배수
- MU 대비 SNDK 상대강도
- SOX·SMH·SOXX 대비 상대강도

대상:

- MU
- SNDK
- NVDA
- AMD
- AVGO
- TSM
- SMH
- SOXX

미국 종목에는 한국식 외국인·기관·개인 수급 API를 사용하지 않는다.

## 8. 당일 가격 스냅샷

### 8.1 상세 가격 배치

```http
GET https://wts-info-api.tossinvest.com/api/v3/stock-prices/details
    ?productCodes=A000660,A005930
```

주요 필드:

```text
code
exchange
tradeDateTime
open
high
low
close
volume
value
base
changeType
currency
```

검사:

- `tradeDateTime`이 당일 KRX 정규장 종료 이후인지
- `exchange`가 KRX인지
- `base`가 전일 종가 또는 기준가 중 무엇인지
- NXT/통합 가격이 아닌지

### 8.2 메인세션·애프터세션

```http
GET /api/v1/stock-prices/mainsession?codes={codes}
GET /api/v1/stock-prices/after?codes={codes}
```

한국 종목에서 `after`가 NXT인지 다른 표시인지 응답 메타데이터로 확인한다. 의미가 불명확하면 NXT 공식 페이지를 사용한다.

미국 종목에서는 정규장과 애프터마켓을 분리하는 보조 호출로 사용하되 Nasdaq와 교차 확인한다.

## 9. KRX 장중 1분봉과 VWAP

```http
GET /api/v1/c-chart/kr-s/{productCode}/min:1
    ?count={N}
    &session=all
    &investMode=krx
    &useAdjustedRate=true
```

### 9.1 KRX 세션 필터

`session=all`이라는 이름만 보고 모든 캔들을 VWAP에 넣지 않는다.

1. candle timestamp를 KST로 변환
2. 최신 완료 KRX 거래일만 선택
3. KRX regular continuous/closing auction에 해당하는 시간만 선택
4. 15:30 이후 캔들 또는 NXT 추정 캔들을 제외
5. 중복 timestamp와 누락 구간 검사

정확한 세션 구분이 불가능하면:

```text
VWAP behavior: 확인 불가 — KRX/NXT session separation unavailable
```

### 9.2 VWAP

```text
typical_price = (high + low + close) / 3
vwap = sum(typical_price × volume) / sum(volume)
```

또는 체결가·체결량 데이터가 완전하면 체결 VWAP를 사용할 수 있다.

필수 메타데이터:

- bar interval
- first/last timestamp
- bar count
- missing bars
- included session
- calculated locally = true

### 9.3 종가 위치

```text
close_location = (close - low) / (high - low)
```

- `0.0` 근처: 저점권
- `0.5` 근처: 범위 중간
- `1.0` 근처: 고점권
- `high == low`이면 계산 불가

## 10. 당일 투자자 수급

```http
GET /api/v1/stock-infos/trade/trend/trading-trend
    ?productCode={productCode}
    &size=10
```

당일 행을 찾을 때 list position이 아니라 `baseDate`를 사용한다.

가능하면 다음 상태 필드를 확인한다.

```text
updatedAt
inMarketTime
hasIndividual
```

필드가 없으면 값의 완결성을 추정하지 않는다.



## 11. Afternoon / Night Brief 확장 호출

### 11.1 Afternoon 15:55

다음 중 하나로 분류한다.

```text
final
provisional
unavailable
```

- `baseDate`가 당일이고 완료 표지가 명확하지 않으면 `provisional`
- 개인/외국인/기관 일부 필드가 누락되면 `provisional` 또는 `unavailable`
- 전일 행만 있으면 당일 수급은 `unavailable`

### 11.2 Night 20:25

같은 endpoint를 다시 조회해 Afternoon 값과 비교한다.

```text
flow_revision = night_value - afternoon_value
```

단, 둘 다 같은 단위와 같은 `baseDate`일 때만 계산한다.

보고 예시:

> 15:55 잠정 외국인 순매도 수량은 20:25 재조회에서 확대됐다. 두 값 모두 WTS 순수량이며 KRX 공식 금액 수급과 동일 개념이 아니다.


### 11.3 분봉

한국 종목:

```http
GET /api/v1/c-chart/kr-s/{productCode}/min:1
    ?count={N}
    &session=all
    &investMode=krx
    &useAdjustedRate=true
```

미국 종목:

```http
GET /api/v1/c-chart/us-s/{productCode}/min:1
    ?count={N}
    &session=all
    &investMode=krx
    &useAdjustedRate=true
```

관찰된 `c-chart` 범위는 `min:1`, `day:1`, `week:1`, `month:1`이다. `1D`, `1H`, `hour:1` 같은 별칭을 임의로 사용하지 않는다.

VWAP는 API가 별도 지표로 제공한다고 가정하지 않는다. 분봉 가격·거래량으로 로컬에서 계산한다.

```text
Typical Price = (high + low + close) / 3
Session VWAP = Σ(Typical Price × volume) / Σ(volume)
```

장 시작 이전·장중 미완성 캔들이 포함되면 VWAP를 확정값으로 사용하지 않는다.

### 11.4 메인세션·애프터세션 가격 분리

관찰된 공개 가격 호출은 다음과 같다.

```http
GET /api/v1/stock-prices/mainsession?codes={codes}
GET /api/v1/stock-prices/after?codes={codes}
```

- 메인세션과 애프터세션을 별도 필드로 저장한다.
- 미국 정규장 종가와 애프터마켓 표시값을 한 값으로 합치지 않는다.
- 응답에 시장 날짜·시각이 없거나 세션 의미가 불명확하면 공식 거래소 또는 Tier 2 소스로 교차 확인한다.
- 엔드포인트가 현재 공개 화면에서 실패하면 재시도 루프를 돌리지 않고 `확인 불가`로 처리한다.

### 11.5 호가와 체결

```http
GET /api/v3/stock-prices/{productCode}/quotes?investMode=krx
GET /api/v2/stock-prices/{productCode}/ticks?viewType=krx&count=20&investMode=krx
```

호가는 브리핑 보조용이다. Task가 지연·갱신시각을 확인하지 못하면 실행 가능 유동성으로 해석하지 않는다.

### 11.6 대차·공매도·신용

```http
GET /api/v1/mds/info/credit?stockCode={productCode}&number=1&size=5
GET /api/v1/mds/info/lending-trading?stockCode={productCode}&number=1&size=5
GET /api/v1/mds/info/short-selling-trend?stockCode={productCode}&number=1&size=5
```

주요 필드 예시:

```text
marginLoanBalanceQuantity
marginLoanIncreaseDecreaseQuantity
marginLoanBalanceRate
executionQuantity
repaymentQuantity
lendingTradingBalanceVolume
lendingTradingBalanceAmount
shortTradingVolume
shortTradingAmount
shortSellingTradingAmountRatio
shortSellingAveragePrice
```

공매도·대차 증가는 다음 날 하락을 자동으로 의미하지 않는다. 헤지·시장조성·차익거래 가능성을 함께 고려한다.

## 12. 프로그램 매매

```http
GET /api/v1/stock-infos/trade/trend/program-trading
    ?productCode={productCode}
    &size=10
```

주요 필드:

```text
baseDate
arbitrageNetBuyQuantity
nonArbitrageNetBuyQuantity
totalNetBuyQuantity
```

해석:

- 외국인/기관 수급과 같은 방향인지
- 비차익이 지배적인지
- 종가 동시호가 또는 ETF 헤지와 연결될 수 있는지

프로그램 순매수가 가격 상승의 원인이라고 단정하지 않는다.


## 13. 호가·체결과 종가 품질

```http
GET /api/v3/stock-prices/{productCode}/quotes?investMode=krx
GET /api/v2/stock-prices/{productCode}/ticks?viewType=krx&count=50&investMode=krx
```

Task 공개 데이터는 지연될 수 있으므로 다음 용도로만 사용한다.

- 마지막 체결 시각 교차확인
- 종가 근처 거래량 집중 여부
- bid/ask 표시 존재 여부

실행 가능한 현재 유동성·슬리피지 판단은 로컬 대시보드가 수행한다.

## 14. 지수·환율·원자재 보조 조회

### 14.1 지수 장중 구조

```http
GET /api/v1/r-chart/kr-s/KGG01P/1d/min:5
    ?session=main
    &investMode=krx
    &last=false
```

KOSDAQ은 현재 검증된 indexCode를 사용한다. KOSPI200은 WTS에서 정확한 current code가 확인되지 않으면 KRX 공식 또는 Tier 2를 사용한다.

계산 가능 항목:

- open/high/low/close
- VWAP, volume이 제공되고 완전할 때
- close location
- 오전/오후 상대강도


### 14.2 주요 indexCode

| 대상 | 관찰된 코드 |
|---|---|
| KOSPI | `KGG01P` |
| KOSDAQ | `QGG01P` |
| Nasdaq-100 futures | `RFU.NQc1` |
| SOX | `SOX.NAI` |
| VIX | `RGI..VIX` |
| 미국 10년물 금리 | `ROB.US10YT-RR` |
| WTI | `RFU.CLv1` |
| 원화 BTC | `VWAP.KRW-BTC` |
| 원화 ETH | `VWAP.KRW-ETH` |

### 14.3 지수 현재값

```http
GET /api/v2/index-infos/{indexCode}
GET /api/v1/index-prices/{indexCode}
```

예시:

```text
https://wts-info-api.tossinvest.com/api/v1/index-prices/KGG01P
https://wts-info-api.tossinvest.com/api/v1/index-prices/QGG01P
https://wts-info-api.tossinvest.com/api/v1/index-prices/SOX.NAI
https://wts-info-api.tossinvest.com/api/v1/index-prices/RFU.NQc1
https://wts-info-api.tossinvest.com/api/v1/index-prices/ROB.US10YT-RR
https://wts-info-api.tossinvest.com/api/v1/index-prices/RFU.CLv1
```

점이 포함된 코드는 대소문자와 구두점을 그대로 보존한다.


### 14.4 USD/KRW

```text
https://wts-info-api.tossinvest.com/api/v1/product/exchange-rate?buyCurrency=USD&sellCurrency=KRW
```

또는 공개 환율 차트:

```http
GET /api/v1/r-chart/fx/EXCHANGE_RATE/{range}/{step}
    ?last=false
    &useAdjustedRate=true
    &currency=USD
```

## 15. 공시·뉴스 후보 수집

### 15.1 회사 공시

```http
GET /api/v1/stock-detail/companies/{companyCode}/filings
    ?number=1
    &size=5
```

예시:

```text
https://wts-info-api.tossinvest.com/api/v1/stock-detail/companies/005930/filings?number=1&size=5
```

공시는 후보 발견용으로만 사용하고, 중요한 내용은 DART·SEC·기업 IR 원문에서 확인한다.

### 15.2 회사 뉴스

```http
GET /api/v2/news/companies/{companyCode}
    ?size=5
    &orderBy=latest
```

예시:

```text
https://wts-info-api.tossinvest.com/api/v2/news/companies/000660?size=5&orderBy=latest
```

뉴스는 가격·수급·지수 조사 이후 마지막에 읽는다. 날짜 없는 제목이나 재게시된 오래된 기사는 촉매로 사용하지 않는다.

## 16. 미국 정규장·프리마켓·애프터마켓

미국 종목의 `session=all` 일봉은 정규장 종가를 보장하지 않는다.

권장 순서:

```text
1. Nasdaq official session quote
2. WTS mainsession / after
3. WTS c-chart for historical regular bars after validation
4. Investing.com/TradingView fallback
```

Night 20:25 KST의 프리마켓 패널은 다음을 분리한다.

```text
previous_regular_close
current_premarket_last
premarket_change_pct
premarket_volume
premarket_high_low
```

프리마켓 volume이 없으면 gap 방향 신뢰도를 낮춘다.

## 17. 신규 상장 SKHY

### 17.1 resolver

신규 Nasdaq ADS가 WTS에 등록됐는지 확인한다.

```http
GET /api/v2/stock-infos/code-or-symbol/SKHYV
GET /api/v2/stock-infos/code-or-symbol/SKHY
```

2026-07-10에는 공식 공지상 `SKHYV` when-issued, 2026-07-13부터 `SKHY` regular-way 전환이 예상된다. 현재 날짜와 공식 심볼 상태에 맞는 코드를 먼저 확인한다. 이 경로가 symbol을 받지 않거나 실패할 수 있다. 실패 시 현재 TossInvest 공개 종목 페이지/검색 트래픽을 확인하기 전까지 임의 productCode를 만들지 않는다.

검증 필드:

```text
code
name
symbol
market
status
isinCode
```

- `name`이 SK hynix인지
- `market`이 Nasdaq인지
- 과거 OTC 또는 Luxembourg GDR이 아닌지
- current trading status가 무엇인지

### 17.2 c-chart

resolver가 opaque US productCode를 반환할 때만:

```http
GET /api/v1/c-chart/us-s/{resolvedProductCode}/min:1?count=120&session=all&investMode=krx&useAdjustedRate=true
GET /api/v1/c-chart/us-s/{resolvedProductCode}/day:1?count=10&session=all&investMode=krx&useAdjustedRate=true
```

신규 상장으로 데이터가 5일 미만이면 정상적인 partial history로 처리한다.

## 18. ETF current-day 패널

한국 단일종목 ETF 각각에 대해:

- current OHLCV
- current investor flow
- flow as percentage of own volume
- underlying 1D return
- ETF theoretical direction과 실제 방향
- leverage/inverse pair divergence

본주와 ETF의 raw net share count를 직접 비교하지 않는다.

## 19. 데이터 건전성 검사

각 호출마다 다음 체크리스트를 적용한다.

### 19.1 공통

- HTTP 성공 여부
- 응답이 JSON인지
- 예상 top-level 키가 있는지
- 종목명·코드가 요청 대상과 일치하는지
- 날짜·시간이 존재하는지
- 최신 완료 거래일과 일치하는지
- 중복 날짜가 없는지
- 값이 문자열인지 숫자인지
- 통화·단위·거래소가 명확한지
- 결측치와 0을 구분했는지

### 19.2 가격

- `tradeDateTime`이 현재 세션인지 전일 세션인지
- `base`가 전일 종가인지 기준가인지
- 수정주가 적용 여부
- 미국 데이터에서 정규장과 애프터마켓을 혼합하지 않았는지
- 신규 상장 ETF의 5일 이력이 실제로 5개 존재하는지

### 19.3 수급

- `baseDate`가 최신 완료 KRX 거래일인지
- 장중 당일 수급이면 `잠정`으로 표시했는지
- 순매수 단위를 주식 수량으로 표시했는지
- `institution_total`과 기관 세부 합계를 중복 합산하지 않았는지
- `other_corporation`을 기관계에 합치지 않았는지

### 19.4 지수·원자재

- 지수코드가 정확한지
- 현물·선물·CFD를 구분했는지
- 거래 중인지 종료된 세션인지
- 지연 여부가 확인 가능한지

## 20. 실패 처리

다음 중 하나이면 해당 필드를 `확인 불가`로 둔다.

- endpoint가 삭제 또는 변경됨
- access denied, challenge, throttling, 로그인 요구
- 응답 날짜가 최신 완료 세션과 맞지 않음
- 코드와 종목명이 일치하지 않음
- 단위가 불명확함
- 0과 결측을 구분할 수 없음
- 같은 날짜에 서로 모순되는 값이 있음
- 응답이 지나치게 오래되었거나 캐시 시각이 불명확함

동일 요청을 반복해 우회하지 않는다. Tier 2 소스로 이동하거나 `확인 불가`로 종료한다.

## 21. Morning Brief 권장 요청 묶음

매일 무조건 모든 엔드포인트를 호출하지 않는다. 다음 순서로 작은 요청을 수행한다.

### 필수 묶음 A — 한국 핵심 5일 패널

각 본주·ETF:

1. 코드 resolver
2. `c-chart/kr-s/.../day:1?count=7`
3. `trading-trend?size=7`
4. 필요 시 `fixed-trading-trend`

### 필수 묶음 B — 미국 핵심 5일 패널

각 종목·ETF:

1. 코드 검증
2. `c-chart/us-s/.../day:1?count=7`
3. 최신 가격 배치 조회

### 필수 묶음 C — 지수·환율

- KOSPI
- KOSDAQ
- SOX
- NQ
- 미국 10년물
- WTI
- USD/KRW

## 22. Afternoon 요청 묶음

```text
A. A000660, A005930, four single-stock ETFs
   - price details
   - day bars
   - min bars
   - trading trend
   - program trading when price/flow diverges

B. KOSPI/KOSDAQ/index context
   - index price
   - intraday chart
   - USD/KRW

C. US/macro watch
   - previous US close panel from Morning cache or re-fetch
   - NQ/ES/WTI/BTC/ETH
   - Binance symbols validated
```

미국 프리마켓은 session status가 open일 때만 조회한다.

## 23. Night 요청 묶음

```text
A. Korea final re-fetch
   - core stock/ETF flow
   - program flow
   - final daily bars

B. NXT official final

C. KRX night derivatives official/current source

D. US premarket
   - SKHY/current listing symbol
   - MU, SNDK, NVDA, AMD, AVGO, TSM
   - DRAM, SMH, SOXX, SOXL, SOXS

E. Binance and macro
```

### 조건부 묶음 D — Afternoon/Night 확장

다음 상황에서만 사용한다.

- 가격과 수급이 크게 괴리: 프로그램 매매
- 하락 압력이 비정상적으로 큼: 대차·공매도
- 장중 구조가 중요: 분봉과 로컬 VWAP
- 이벤트 발생: 공시·뉴스

## 24. Task에 전달할 권장 정규화 스키마

```json
{
  "instrument": {
    "name": "SK Hynix",
    "market": "KRX",
    "exchange_code": "000660",
    "toss_product_code": "A000660",
    "currency": "KRW"
  },
  "session": {
    "market_date": "YYYY-MM-DD",
    "completed": true,
    "source_timestamp": "",
    "delay_label": ""
  },
  "five_day_price": [
    {
      "date": "YYYY-MM-DD",
      "open": null,
      "high": null,
      "low": null,
      "close": null,
      "volume": null,
      "amount": null
    }
  ],
  "five_day_flow": [
    {
      "date": "YYYY-MM-DD",
      "foreign_net_volume": null,
      "institution_net_volume": null,
      "individual_net_volume": null,
      "pension_net_volume": null,
      "other_corporation_net_volume": null
    }
  ],
  "derived": {
    "return_1d": null,
    "return_5d": null,
    "latest_volume_ratio": null,
    "foreign_5d_net_volume": null,
    "institution_5d_net_volume": null,
    "foreign_latest_share_of_volume": null,
    "institution_latest_share_of_volume": null
  },
  "quality": {
    "source": "TossInvest WTS public web API",
    "official": false,
    "date_aligned": false,
    "unit_verified": false,
    "warnings": []
  }
}
```

## 25. 데이터 차이 기록

Afternoon과 Night 값이 다르면 덮어쓰지 않고 revision을 기록한다.

```json
{
  "field": "sk_hynix_foreign_net_volume",
  "market_date": "",
  "afternoon_value": null,
  "afternoon_status": "provisional",
  "night_value": null,
  "night_status": "final",
  "unit": "shares",
  "revision": null,
  "source": "TossInvest WTS public web API"
}
```

## 26. 원본 문서로 넘길 항목

다음은 일상 브리핑 필수 범위를 넘어가므로 원본 카탈로그를 참조한다.

- 시장 랭킹·거래대금 상위
- 테마·산업 분류
- 재무제표·밸류에이션·컨센서스
- 배당·실적 캘린더
- 스크리너
- 월간 경제 캘린더
- 상세 뉴스 피드
- 공개 커뮤니티 데이터
- AI 요약·시그널

원본:

- [TossInvest Web API Catalog](https://github.com/dd3ok/tossinvest-api-skill/blob/main/references/api-catalog.md)
- [tossinvest-api-skill repository](https://github.com/dd3ok/tossinvest-api-skill)

AI 요약·커뮤니티·사용자 sentiment는 이 프로젝트의 데이터 신호로 사용하지 않는다.
