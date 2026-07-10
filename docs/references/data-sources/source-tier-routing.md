# Market Data Source-Tier Routing Policy

> 버전: 2026-07-10
> 목적: Korea Semiconductor Morning / Afternoon / Night Brief의 소스 선택·폴백·감사 규칙

## 1. Tier의 의미

이 문서의 Tier는 출처의 명성만 평가하지 않는다. 다음 요소를 함께 본다.

1. ChatGPT Scheduled Task가 공개 웹에서 직접 읽을 수 있는가
2. 매 거래일 또는 필요한 주기로 갱신되는가
3. 시장 날짜·타임스탬프·지연 여부가 명확한가
4. 가격·수급·거래량·파생지표 등 필요한 범위를 제공하는가
5. 같은 방식으로 반복 조사할 수 있는가
6. 오류·차단 시 대체 경로가 있는가

따라서 다음을 구분한다.

- **External-agent authoritative**: 가장 좋은 원천일 수 있지만 Task 단독으로는 인증·키 관리 때문에 직접 사용할 수 없음
- **Task-direct Tier 1**: Task가 매일 우선 확인할 공개 소스
- **Tier 2**: Tier 1에서 필드가 없거나 페이지가 막혔을 때 쓰는 범용 폴백
- **Tier 3**: 주간·비정기·기사형·맥락용 자료

## 2. 전역 라우팅 알고리즘

각 데이터 필드마다 다음 순서로 조사한다.

```text
1. 필요한 필드를 정확히 정의한다.
   예: "SK하이닉스 2026-07-09 외국인 순매수 수량"

2. Task-direct Tier 1에서 찾는다.

3. Tier 1 값에 날짜·단위·시장·세션·지연표시가 있는지 검증한다.

4. 핵심 필드가 없거나 읽을 수 없을 때만 Tier 2로 이동한다.

5. Tier 2도 부족하면 Tier 3에서 배경 맥락 또는 날짜가 명확한 기사형 보조값을 찾는다.

6. 여전히 검증되지 않으면 "확인 불가"로 종료한다.

7. 하위 Tier 값으로 상위 Tier 값을 평균하거나 임의 보정하지 않는다.
```

핵심 필드:

- SK하이닉스·삼성전자 종가와 5일 OHLCV
- 해당 종목·ETF의 투자자별 수급
- KOSPI/KOSDAQ/KOSPI200
- KOSPI200 선물·옵션 수급
- MU·SNDK·NVDA·AMD·AVGO·TSM 종가
- Binance stock perpetual mark/index/funding/OI/volume

핵심 필드가 Tier 2 또는 Tier 3에만 의존하면 데이터 품질과 신뢰도 상한을 낮춘다.


## 3. 시간대별 라우팅 원칙

같은 출처라도 브리핑 시각에 따라 의미가 달라진다.

| 데이터 | Afternoon 15:55 KST | Night 20:25 KST |
|---|---|---|
| KRX 현물 | 완료 세션 | 완료 세션 |
| 종목/ETF 당일 수급 | 잠정일 수 있음 | 최종값 우선 |
| KOSPI200 파생 정규장 | 15:45 종료 직후, 20분 지연이면 미완성 가능 | 최종 정규장 값 확인 가능 |
| NXT 애프터마켓 | 진행 중, 20분 지연 초기값 | 20:00 종료 후 최종 지연값 가능 |
| KRX 파생 야간세션 | 아직 시작 전 | 18:00 이후 진행 중 |
| 미국 주식 프리마켓 | 통상 시작 전 | 진행 중 |
| 미국 정규장 | 시작 전 | 시작 전 |
| SKHY ADS/ADR | 상장·공모·환산 기준 준비 | 프리마켓/when-issued/거래상태 핵심 |

시각표는 휴장·조기폐장·서머타임에 따라 달라질 수 있으므로 매회 공식 세션 상태를 확인한다.


## 4. External-agent authoritative sources

아래 소스는 인증된 외부 수집기, MCP, 연결 앱, 서버리스 함수 또는 로컬 에이전트가 있을 때 사용한다. 일반 Scheduled Task가 직접 호출했다고 가정하지 않는다.

### 4.1 KRX Open API

- Portal: <https://openapi.krx.co.kr/>
- 역할: KRX 공식 일별·시장별 데이터 자동 적재
- 강점: 공식성, 구조화된 데이터
- 제약: 서비스키·등록·호출 절차가 필요할 수 있음
- Task 규칙: 연결 에이전트가 명시적으로 결과를 제공하지 않으면 “KRX Open API 확인”이라고 쓰지 않는다.

### 4.2 Toss Securities official Open API

- Product: <https://corp.tossinvest.com/ko/open-api>
- Overview: <https://openapi.tossinvest.com/openapi-docs/overview.md>
- OpenAPI spec: <https://openapi.tossinvest.com/openapi-docs/latest/openapi.json>
- 역할: 인증된 가격·캔들·호가·체결·시장지표·투자자 매매 자료
- 제약: OAuth 2.0 `client_credentials`, 비밀키, 토큰 관리 필요
- Task 규칙: 프롬프트에 `client_id`, `client_secret`, token을 넣지 않는다.

### 4.3 기타 외부 에이전트용

- KIS Open API / MCP
- 유료 시장데이터 공급자
- 내부 Streamlit API와 데이터베이스
- 실시간 execution-venue feed

이 데이터가 제공되면 공개 웹 자료보다 우선할 수 있지만, 보고서에는 공급 시각·필드·지연·정합성 검사를 기록한다.

## 5. Task-direct Tier 1

### 5.1 TossInvest WTS 공개 웹 API

- 상세 가이드: [`tossinvest-wts/README.md`](tossinvest-wts/README.md)
- 원본 카탈로그: <https://github.com/dd3ok/tossinvest-api-skill/blob/main/references/api-catalog.md>
- 기본 호스트: `https://wts-info-api.tossinvest.com`

#### 사용 목적

- 한국·미국 종목 5일 OHLCV
- 한국 종목·ETF 투자자별 일별 수급
- 프로그램 매매
- 대차·공매도·신용
- 지수·환율·원자재 보조값
- 종목 공시·뉴스 후보

#### 품질 특성

- 장점: 범위가 넓고 구조화되어 있으며 로그인 없이 공개 화면 데이터에 접근 가능
- 단점: 비공식·무보증·변경 가능
- 분류 이유: **Task 실행 가능성과 범용성 때문에 Tier 1**, 공식성은 별도 경고

#### 우선순위

한국 종목·ETF의 최근 5일 패널은 WTS를 우선 시도하고, 중요한 종가·시장 수급은 KRX 공개 페이지와 가능한 범위에서 교차 확인한다.

### 5.2 WTS 시간대별 역할

#### Afternoon

우선 사용:

- `v3/stock-prices/details`
- `c-chart/kr-s/.../day:1`
- `c-chart/kr-s/.../min:1`
- `trading-trend`
- `program-trading`
- `stock-prices/mainsession`
- KOSPI/KOSDAQ index price and intraday chart

주의:

- current-day flow가 완료 상태인지 확인
- `session=all`이 KRX 정규장 이외 세션을 섞는지 확인
- 분봉이 15:30 이후 캔들을 포함하면 KRX VWAP에서 제외

#### Night

우선 사용:

- KRX 종목·ETF의 최종 일별 수급 재조회
- NXT 공식 최종값
- 미국 종목 `mainsession`/`after` 또는 Nasdaq 프리마켓
- SKHY resolver 및 price details, 등록된 경우에만
- 지수·FX·원자재 보조값

WTS 신규 상장 종목이 미등록이면 Nasdaq Tier 1을 우선하고 Tier 2로 이동한다.

### 5.3 KRX 파생상품 정규·야간세션

- KOSPI200 Futures product page:
  https://global.krx.co.kr/contents/GLB/02/0201/0201040201/GLB0201040201.jsp
- KOSPI200 Options product page:
  https://global.krx.co.kr/contents/GLB/02/0201/0201040202/GLB0201040202.jsp
- KRX night-session overview:
  https://open.krx.co.kr/contents/OPN/01/01041402/OPN01041402.jsp
- KRX night-session guide:
  https://global.krx.co.kr/contents/GLB/02/0201/0201041003/Guide_to_Night_Session_in_KRX_Derivatives_Market.pdf

핵심 시간 기준:

```text
KOSPI200 futures regular session: 08:45–15:45, except contract-specific last-day rules
KRX derivatives night session: 18:00–next day 06:00 for eligible products
```

규칙:

- Afternoon 15:55에는 파생 정규장 종가는 끝났지만 20분 지연 공개값이 마지막 10분을 반영하지 못할 수 있다.
- Night 20:25에는 정규장 최종값과 18시 이후 야간세션 값을 분리한다.
- 야간거래의 거래일은 종료일 기준일 수 있으므로 calendar date와 trading date를 분리한다.
- 정규장 settlement와 야간 last price를 같은 필드로 합치지 않는다.
- 최종거래일에는 거래시간이 다를 수 있다.


### 5.4 NXT 공식 공개 시장정보

- Market data: <https://www.nextrade.co.kr/menu/marketData/menuList.do>
- Daily trading status: <https://www.nextrade.co.kr/menu/transactionStatusDaily/menuList.do>

### 조사 대상

- 프리마켓·메인마켓·애프터마켓
- 기준가 대비
- KRX 종가 대비
- 거래량·거래대금
- 거래 대상 종목

### 규칙

- 공개 화면은 20분 지연으로 표시될 수 있다.
- Afternoon 15:55와 Night 20:25 브리핑에서는 지연값임을 명시한다.
- NXT 값이 비어 있거나 기준 시각이 없으면 `확인 불가`로 처리한다.
- NXT 방향을 KRX 정규장 방향과 자동 동일시하지 않는다.

### 5.5 미국 메모리 특화 ETF

- Roundhill Memory ETF `DRAM`:
  https://www.roundhillinvestments.com/etf/dram/
- DRAM Nasdaq quote:
  https://www.nasdaq.com/market-activity/etf/dram
- Roundhill 2X Long DRAM Daily Target ETF `RAM`:
  https://www.roundhillinvestments.com/etf/ram/

사용 목적:

- MU, SNDK, SKHY, Samsung, SK Hynix가 포함된 메모리 업종의 미국 가격발견
- 광범위한 SOX/SMH/SOXX와 메모리 순수 노출의 차이
- Night Brief의 미국 프리마켓 상대강도

규칙:

- ETF 가격·거래량은 순자금 유입이 아니다.
- 운용사 보유종목은 기준일을 확인한다.
- RAM은 일일 2배 목표 상품이므로 장기 성과를 DRAM의 2배로 간주하지 않는다.


### 5.6 DART / KIND / 기업 IR

- DART: <https://dart.fss.or.kr/>
- KIND: <https://kind.krx.co.kr/>
- SK hynix IR: <https://news.skhynix.com/category/ir/>
- Samsung Electronics IR: <https://www.samsung.com/global/ir/>
- Micron IR: <https://investors.micron.com/>
- Sandisk IR: <https://investor.sandisk.com/>

### 조사 대상

- 실적·가이던스
- 투자·공급·생산·설비 공시
- 주주환원
- 거래정지·유상증자·합병·중요 계약

### 규칙

- 공시 원문이 기사보다 우선한다.
- 공시의 제출일과 실제 사건일을 구분한다.
- 오래된 구조적 뉴스는 `background context`로 분류한다.

### 5.7 Binance 공식 공개 선물 데이터

- Market-data catalog: <https://developers.binance.com/en/docs/catalog/core-trading-derivatives-trading-usd-s-m-futures/api/rest-api/market-data>
- General info: <https://developers.binance.com/en/docs/products/derivatives-trading-usds-futures/general-info>
- Public futures pages:
  - SK Hynix: <https://www.binance.com/en/futures/SKHYNIXUSDT>
  - Samsung: <https://www.binance.com/en/futures/SAMSUNGUSDT>
  - Micron: <https://www.binance.com/en/futures/MUUSDT>
  - Sandisk: <https://www.binance.com/en/futures/SNDKUSDT>

### 먼저 심볼 유효성 확인

```http
GET https://fapi.binance.com/fapi/v1/exchangeInfo
```

심볼이 `TRADING` 상태인지, contract type과 onboard date가 무엇인지 확인한 뒤 조회한다. 웹 검색 결과에 보인 심볼이라도 API의 현재 `exchangeInfo`가 우선이다.

### 주요 공개 조회

#### Mark·Index·Funding

```http
GET https://fapi.binance.com/fapi/v1/premiumIndex?symbol=SKHYNIXUSDT
```

주요 필드:

```text
symbol
markPrice
indexPrice
estimatedSettlePrice
lastFundingRate
nextFundingTime
time
```

#### Funding history

```http
GET https://fapi.binance.com/fapi/v1/fundingRate?symbol=SKHYNIXUSDT&limit=10
```

#### Current open interest

```http
GET https://fapi.binance.com/fapi/v1/openInterest?symbol=SKHYNIXUSDT
```

#### 5-minute candles

```http
GET https://fapi.binance.com/fapi/v1/klines?symbol=SKHYNIXUSDT&interval=5m&limit=100
```

#### 24-hour ticker

```http
GET https://fapi.binance.com/fapi/v1/ticker/24hr?symbol=SKHYNIXUSDT
```

#### Order book

```http
GET https://fapi.binance.com/fapi/v1/depth?symbol=SKHYNIXUSDT&limit=20
```

#### Basis

```http
GET https://fapi.binance.com/futures/data/basis?pair=SKHYNIXUSDT&contractType=PERPETUAL&period=5m&limit=30
```

### 해석 규칙

- `markPrice - indexPrice`를 계산할 수는 있지만, 계약 multiplier·index 구성·원화 환산 구조를 모르면 KRX 현물 대비 프리미엄이라고 부르지 않는다.
- funding이 양수라고 단순히 하락 신호로 해석하지 않는다. 가격·OI·거래량·현물 방향을 함께 본다.
- OI가 0, null, 비정상적으로 고정되면 `OI 확인 불가`로 처리한다.
- 웹페이지 값보다 공식 API의 timestamp가 명확한 값을 우선한다.
- 24시간 변화율과 KRX 정규장 이후 변화율은 다르다. Morning Brief에는 가능하면 현물 종가 시각 이후 변화율을 별도로 계산한다.
- rate limit을 준수하고 불필요한 심볼 전체 조회를 반복하지 않는다.

### 5.8 Nasdaq 공식 시장 페이지

- SOX: <https://www.nasdaq.com/market-activity/index/sox>
- MU: <https://www.nasdaq.com/market-activity/stocks/mu>
- NVDA: <https://www.nasdaq.com/market-activity/stocks/nvda>
- AMD: <https://www.nasdaq.com/market-activity/stocks/amd>
- AVGO: <https://www.nasdaq.com/market-activity/stocks/avgo>
- SNDK: <https://www.nasdaq.com/market-activity/stocks/sndk>

### 규칙

- 정규장·프리마켓·애프터마켓을 구분한다.
- Nasdaq 페이지가 `Data is currently not available`을 표시하면 숫자를 추출하려고 추정하지 않고 Tier 2로 이동한다.
- 지수 데이터는 최소 지연될 수 있으므로 페이지 고지를 보존한다.

#### SK hynix 미국 ADS/ADR

- Nasdaq Trader when-issued / symbol-transition notice:
  https://www.nasdaqtrader.com/TraderNews.aspx?id=DTN2026-11
- Nasdaq Trader equity alert:
  https://www.nasdaqtrader.com/TraderNews.aspx?id=ETA2026-37
- Nasdaq regular-way SKHY quote:
  https://www.nasdaq.com/market-activity/stocks/skhy
- Nasdaq symbol directory:
  https://www.nasdaqtrader.com/Trader.aspx?id=symbollookup
- SEC F-1/A filing index:
  https://www.sec.gov/Archives/edgar/data/2120882/000119312526295501/0001193125-26-295501-index.htm
- SEC F-6 filing index:
  https://www.sec.gov/Archives/edgar/data/2120882/000119380526000898/0001193805-26-000898-index.htm
- SK hynix disclosures:
  https://www.skhynix.com/ir/UI-FR-IR12_T1/
- SK hynix KRX share price:
  https://www.skhynix.com/ir/UI-FR-IR02/

별도 가이드:

- `skhynix-adr-us-session-guide.md`

규칙:

- 2026-07-10에는 공식 공지상 when-issued `SKHYV`, 2026-07-13부터 regular-way `SKHY` 전환이 예상되지만 매회 실제 상태를 재확인한다.
- `SKHY` 페이지가 존재해도 현재 거래 중인지 별도 확인한다.
- 상장 첫날에는 when-issued, new-issue halt/cross, regular-way, halted 상태를 구분한다.
- `SKHYV`와 `SKHY`를 독립된 두 종목으로 중복 집계하지 않는다.
- ADS 비율은 최신 prospectus에서 확인한다.
- KRX 환산 ADS 값은 동일 비율과 USD/KRW timestamp가 있을 때만 계산한다.
- 미국 프리마켓 가격을 정규장 종가로 표현하지 않는다.
- 신규상장 초기 5개 정규장 동안 교차시장 추론의 confidence를 제한한다.

### 5.9 CME 공식 선물 페이지

- NQ: <https://www.cmegroup.com/markets/equities/nasdaq/e-mini-nasdaq-100.html>
- ES: <https://www.cmegroup.com/markets/equities/sp/e-mini-sandp500.html>
- WTI: <https://www.cmegroup.com/markets/energy/crude-oil/light-sweet-crude.html>
- Brent: <https://www.cmegroup.com/markets/energy/crude-oil/brent-crude-oil.html>

### 조사 대상

- 최근월물
- 전일 settlement
- 시가·고가·저가
- 거래량
- open interest
- 월물 롤오버

### 규칙

- 현재가와 전일 settlement를 구분한다.
- 근월물 변경 시 연속선물 변화처럼 보이는 갭을 주의한다.
- CME 공개 quote의 지연 고지를 기록한다.
- Investing.com CFD를 CME settlement로 대체하지 않는다.

### 5.10 Cboe 공식 옵션·변동성 통계

- Daily options statistics: <https://www.cboe.com/markets/us/options/market-statistics/daily/>
- VIX: <https://www.cboe.com/tradable_products/vix/>
- VIX futures: <https://www.cboe.com/tradable_products/vix/vix_futures/>

### 조사 대상

- Total put/call ratio
- Equity put/call ratio
- Index put/call ratio
- ETP put/call ratio
- 옵션 거래량과 OI
- VIX와 선물곡선

### 규칙

- put/call ratio는 직접적인 기관·개인 수급이 아니다.
- 하루 수치 하나로 방향을 단정하지 않는다.
- extreme 여부는 최근 범위와 함께 해석한다.

### 5.11 US Treasury 공식 금리

- Interest-rate statistics: <https://home.treasury.gov/policy-issues/financing-the-government/interest-rate-statistics>
- Daily par yield curve: <https://home.treasury.gov/resource-center/data-chart-center/interest-rates/TextView?type=daily_treasury_yield_curve>

### 조사 대상

- 2년물
- 10년물
- 2s10s 변화

### 규칙

- Treasury 값은 장중 tick이 아니라 일일 indicative closing quotation이다.
- Morning Brief에서 최신 미국 정규장 날짜와 일치하는지 확인한다.
- 실시간 금리 방향은 CME·Investing.com 등 보조값과 구분한다.

### 5.12 공식 ETF 운용사

- SMH: <https://www.vaneck.com/us/en/investments/semiconductor-etf-smh/>
- SOXX: <https://www.ishares.com/us/products/239705/ishares-phlx-semiconductor-etf>
- KODEX: <https://www.samsungfund.com/etf/main.do>
- SOL ETF: <https://www.soletf.com/>

### 조사 대상

- 상품명·종목코드
- 기초지수·배수·인버스 구조
- 보유종목
- NAV/iNAV·AUM·분배금
- 상품 공지와 괴리율 유의사항

### 규칙

- 운용사 페이지의 거래량 또는 AUM 변화가 명시적 일일 fund flow가 아니면 순유입으로 표현하지 않는다.
- 레버리지·인버스 ETF는 일간 수익률 목표 상품임을 명시한다.

## 6. Tier 2

### 6.1 Investing.com

### 주요 링크

- Korea markets: <https://kr.investing.com/markets/south-korea>
- KOSPI: <https://kr.investing.com/indices/kospi>
- Economic calendar: <https://kr.investing.com/economic-calendar>
- Nasdaq-100 futures: <https://kr.investing.com/indices/nq-100-futures>
- USD/KRW: <https://kr.investing.com/currencies/usd-krw>
- US 10Y: <https://kr.investing.com/rates-bonds/u.s.-10-year-bond-yield>
- WTI: <https://kr.investing.com/commodities/crude-oil>
- Brent: <https://kr.investing.com/commodities/brent-oil>
- SK Hynix: <https://kr.investing.com/equities/sk-hynix-inc>
- Samsung Electronics: <https://kr.investing.com/equities/samsung-electronics-co-ltd>
- MU: <https://www.investing.com/equities/micron-tech>
- SNDK: <https://www.investing.com/equities/sandisk-corp>

### 좋은 용도

- 경제 캘린더 actual / forecast / previous
- 일별 OHLCV와 과거 데이터
- FX·금리·원유·지수선물의 범용 시장 지도
- Nasdaq·CME 페이지가 읽히지 않을 때의 교차확인
- 프리마켓·애프터마켓 표기가 명확한 주식 가격

### 필수 메타데이터

- instrument type
- exchange
- currency
- contract month
- timestamp
- real-time / delayed / derived / CFD

### 금지

- ownership percentage를 일별 수급으로 사용
- ETF 가격·거래량을 net flow라고 표현
- CFD 가격을 CME official settlement라고 표현
- user sentiment, forum, community forecast 사용
- 날짜가 없는 AI summary를 촉매로 사용

### 추가 규칙

Afternoon:

- KRX 종가 교차확인
- USD/KRW, NQ, WTI 방향
- 미국 경제일정
- 미국 프리마켓으로 오인하지 않도록 session label 확인

Night:

- SKHY, MU, SNDK, DRAM 등 프리마켓 폴백
- instrument type, exchange, timestamp, real-time/delayed/derived 상태 기록
- 신규상장 페이지가 검색되지 않으면 ticker URL을 추측하지 말고 사이트 검색을 사용

### 6.2 TradingView 공개 종목 페이지

좋은 용도:

- 가격·OHLC·거래량 폴백
- 5일·1개월 방향의 시각적 교차확인

금지:

- Community ideas
- 사용자 예측
- 공식 범용 REST market-data API가 있다고 가정
- 차트 이미지만 보고 정확한 VWAP 숫자를 추출

### 추가 규칙

- SKHY와 신규 ETF의 fallback quote/OHLC
- 커뮤니티 아이디어 제외
- `extended hours`와 regular session 표시를 구분
- 정확한 VWAP 숫자를 이미지에서 추출하지 않음

### 6.3 FRED

- <https://fred.stlouisfed.org/>

좋은 용도:

- 느리게 변하는 거시 시계열
- 과거 비교
- 금리·유동성·경기지표의 장기 맥락

제약:

- 장중 주가·선물·FX 대체재가 아님
- 시리즈별 업데이트 지연과 release date를 확인

### 6.4 기타 ETF 데이터 사이트

ETF flow가 명시적 날짜와 순유입·순유출 값으로 제공될 때만 사용한다. 단순 거래량·가격 상승·AUM을 flow로 변환하지 않는다.

## 7. Tier 3

### 7.1 CFTC Commitments of Traders

- <https://www.cftc.gov/MarketReports/CommitmentsofTraders/index.htm>

용도:

- Nasdaq/S&P equity-index futures의 주간 포지셔닝
- WTI·원유 포지셔닝
- USD·주요 통화 포지셔닝

규칙:

- 매일 수급이 아니다.
- position as-of date와 publication date를 함께 표시한다.
- 전날 외국인·기관 수급처럼 표현하지 않는다.

### 7.2 EIA Weekly Petroleum Status Report

- <https://www.eia.gov/petroleum/supply/weekly/>

용도:

- 원유 재고·생산·정제·수입의 주간 촉매

규칙:

- 새로 발표되었거나 시장 반응이 지속될 때만 사용한다.
- WTI 실시간 가격과 같은 시계열로 취급하지 않는다.

### 7.3 ETF.com flow tool

- <https://www.etf.com/etfanalytics/etf-fund-flows-tool>

용도:

- QQQ·SPY·SMH·SOXX·XLK·XLE·USO·GLD·TLT의 명시적 날짜별 net flow가 실제로 노출될 때

규칙:

- 날짜·기간·단위가 없으면 사용하지 않는다.
- 가격·거래량을 flow로 대체하지 않는다.

### 7.4 날짜가 명확한 금융 뉴스

허용:

- Reuters
- Bloomberg
- Financial Times
- Wall Street Journal
- 연합뉴스·한국경제·매일경제·조선비즈 등 날짜와 원출처가 명확한 보도

용도:

- 사건·정책·기업 발표 맥락
- 공식 표를 읽지 못했을 때의 수급 요약

규칙:

- 기사 게시일과 시장 거래일을 분리한다.
- 기사 내 수급이 KRX 원자료인지 확인한다.
- 한 기사만으로 숫자를 확정하지 않는다.

## 8. 명시적 제외

다음은 브리핑 시장데이터 소스로 사용하지 않는다.

- 프로젝트에서 제외하기로 한 실행 거래소의 공개 가격·funding·OI·계약 페이지와 링크
- 소셜미디어 루머
- 커뮤니티 게시물
- 사용자 sentiment 투표
- 날짜 없는 검색 스니펫
- 재게시된 오래된 기사
- 출처 없는 AI 요약
- 가격·거래량으로 추정한 ETF 순유입

실제 거래소 포지션 데이터는 `local execution-venue data`로만 다루고 로컬 리스크 엔진에서 검증한다.

## 9. 필드별 라우팅 맵

### 9.1 Morning 라우팅 맵

| 필요한 필드 | 1차 | 2차 | 3차 |
|---|---|---|---|
| SK하이닉스·삼성전자 5일 OHLCV | Toss WTS | KRX / Investing.com | dated close article |
| 종목별 외국인·기관·개인 수급 | Toss WTS + KRX | dated close article | 확인 불가 |
| 단일종목 ETF 5일 OHLCV·수급 | Toss WTS + KRX ETF | 운용사 / Investing.com | 확인 불가 |
| KOSPI/KOSDAQ/KOSPI200 | KRX / WTS | Investing.com / TradingView | news article |
| KOSPI200 선물·옵션 수급 | KRX | dated market report | 확인 불가 |
| NXT | NXT official | broker/investing article | 확인 불가 |
| MU/SNDK/NVDA/AMD/AVGO/TSM | Toss WTS + Nasdaq | Investing.com / TradingView | news article |
| SOX | Nasdaq / WTS | Investing.com / TradingView | news article |
| SMH/SOXX | Toss WTS + issuer | Investing.com | ETF article |
| NQ/ES | CME | Investing.com / TradingView | news article |
| US 2Y/10Y | US Treasury | WTS / Investing.com / FRED | news article |
| WTI/Brent | CME | WTS / Investing.com | EIA + news |
| BTC/ETH | Binance / WTS | Investing.com / TradingView | news |
| Binance stock-perp mark/index/funding/OI | Binance API | Binance web page | 확인 불가 |
| 경제일정 | Investing.com + official release pages | WTS calendar | news preview |
| 기업 촉매 | DART/SEC/IR | WTS news / reputable news | other news |

### 9.2 Afternoon 라우팅 맵

| 필드 | 1차 | 2차 | 실패 시 |
|---|---|---|---|
| SK Hynix/Samsung 당일 OHLCV | WTS + KRX | Investing.com | 확인 불가 |
| KRX 정규장 1분봉/VWAP | WTS min bars, session filtered | local dashboard | VWAP 확인 불가 |
| 당일 종목·ETF 수급 | WTS, KRX | dated close report | provisional/확인 불가 |
| 프로그램 매매 | WTS/KRX | dated report | 확인 불가 |
| KOSPI200 파생 종가/수급 | KRX | market report | provisional/확인 불가 |
| NXT 초기 애프터 | NXT official | broker article | 확인 불가 |
| NQ/ES | CME | Investing/TradingView | 확인 불가 |
| USD/KRW | WTS/KRX/official FX source | Investing | 확인 불가 |
| Binance risk | Binance official API | Binance page | 확인 불가 |
| SKHY offering/listing baseline | SEC/Nasdaq/SK hynix IR | reputable news | 확인 불가 |

### 9.3 Night 라우팅 맵

| 필드 | 1차 | 2차 | 실패 시 |
|---|---|---|---|
| KRX 최종 종가·수급 | KRX + WTS | dated close article | 확인 불가 |
| NXT 최종 애프터 | NXT official | broker article | 확인 불가 |
| KRX 야간 KOSPI200 | KRX official market data | Investing/TradingView only if instrument is exact | 확인 불가 |
| SKHY listing status/price | Nasdaq + SEC | WTS resolver / Investing / TradingView | 확인 불가 |
| MU/SNDK/NVDA/AMD/AVGO/TSM premarket | Nasdaq | WTS/Investing | 확인 불가 |
| DRAM/RAM/SMH/SOXX premarket | issuer/Nasdaq | Investing/TradingView | 확인 불가 |
| NQ/ES/VIX | CME/Cboe | Investing/TradingView | 확인 불가 |
| Binance funding/OI/basis | Binance official API | Binance page | 확인 불가 |


## 10. 교차검증 규칙

### 10.1 반드시 교차검증할 데이터

- SK하이닉스·삼성전자 최신 종가
- 단일종목 ETF 종목코드와 상품 구조
- 큰 폭의 투자자 수급
- 거래정지·공시·실적
- Binance 신규·중단 심볼

### 10.2 교차검증 우선순위

```text
공식 거래소·공시·프로토콜
> WTS 구조화 공개값
> Investing.com / TradingView
> 날짜가 명확한 기사
```

단, Task 접근성이 없는 인증 API는 실제 결과가 제공되었을 때만 우선한다.

### 10.3 충돌 시

1. 시장 날짜가 같은지 확인한다.
2. 정규장·프리·애프터·NXT 세션을 확인한다.
3. 통화와 수정주가 여부를 확인한다.
4. 거래량·거래대금·순매수 금액·순매수 수량을 구분한다.
5. 해결되지 않으면 두 값을 모두 쓰지 않고 `출처 간 불일치로 확인 불가`라고 표시한다.

## 11. Source Audit 출력 표준

브리핑의 `sources_checked`는 실제로 사용한 소스만 포함한다.


```json
{
  "source_name": "",
  "tier": 1,
  "url": "",
  "instrument": "",
  "market_date": "",
  "trading_date": "",
  "session": "regular/pre_market/after_hours/nxt_after/krx_night",
  "listing_phase": "",
  "source_timestamp": "",
  "delay_or_frequency": "",
  "unit": "",
  "fields_used": [],
  "status": "verified/provisional/fallback/unavailable/conflicting",
  "limitations": []
}
```

Tier 2 또는 Tier 3를 사용했다면 다음을 기록한다.

- 어떤 Tier 1 필드가 부족했는가
- 하위 Tier 값의 지연·파생·CFD·주간 특성
- 신뢰도 상한에 어떤 영향을 주었는가

## 12. Confidence cap 연동

| 상태 | 최대 confidence |
|---|---:|
| 핵심 가격과 5일 패널이 Tier 1에서 날짜 정합성까지 검증 | 0.70 이내 |
| 핵심 가격 중 일부가 Tier 2 폴백 | 0.55 |
| 한국 종목·ETF 5일 수급이 불완전 | 0.50 |
| 파생 funding/OI/spread 미확인 | 0.55 |
| 뉴스만 있고 가격·수급 패널 부재 | 0.45 |
| 출처 간 충돌 미해결 | 0.40 |
| Afternoon 당일 핵심 수급이 잠정 | 0.50 |
| Afternoon 파생 정규장 마지막 20분 미반영 가능 | 0.55 |
| Afternoon VWAP 세션 분리 실패 | 0.55 |
| Night NXT 최종값 불명 | 0.55 |
| Night KRX 야간파생 불명 | 0.55 |
| SKHY listing phase/symbol 불명 | 0.45 |
| SKHY when-issued만 확인 | 0.55 |
| SKHY 상장 후 첫 5개 정규장 | SKHY 교차시장 추론 0.60 |
| 미국 프리마켓만 있고 정규장 미개장 | Night next-day 방향 0.65 |

confidence는 방향 확률이 아니라 **자료 품질과 시나리오 신뢰도**다.
