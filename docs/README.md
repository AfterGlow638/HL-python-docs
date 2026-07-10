# Korea Semiconductor Briefing / Afternoon / Night Brief 문서 세트

> 버전: 2026-07-10
> 대상: 멀티 AI 주식 프레임워크의 Morning / Afternoon / Night Brief
> 핵심 종목: SK하이닉스, 삼성전자, Micron, Sandisk 및 관련 단일종목·반도체 ETF

이 디렉터리는 하나의 거대한 Task 프롬프트를 네 개의 역할별 문서로 분리한다. 목적은 매일 바뀌는 시장 데이터와 상대적으로 안정적인 조사 규칙을 분리해, 프롬프트를 짧게 유지하면서도 데이터 건전성과 재현성을 높이는 것이다.

## 권장 저장소 구조

```text
docs/
├── prompts/
│   ├── morning-brief-task-prompt.md
│   ├── afternoon-brief-task-prompt.md
│   └── night-brief-task-prompt.md
├── references/
│   └── data-sources/
│       ├── tossinvest-wts/
│       │   ├── README.md
│       │   └── afternoon-night-addendum.md
│       ├── source-tier-routing.md
│       ├── source-tier-routing-addendum-afternoon-night.md
│       └── skhynix-adr-us-session-guide.md
└── workflows/
    ├── briefing-workflow-examples.md
    ├── afternoon-night-workflow-examples-addendum.md
    └── brief-handoff-contract.md
```

## 설계 원칙

### 1. 프롬프트는 오케스트레이션만 담당한다

Task 프롬프트에는 다음만 둔다.

- 어떤 세션과 종목을 조사할지
- 어떤 순서로 자료를 모을지
- 어떤 데이터 건전성 규칙을 적용할지
- 어떤 출력 구조와 신뢰도 상한을 사용할지

세부 API 경로, 수십 개 링크, 필드 정의, 사례는 참조 문서로 분리한다.

### 2. 데이터 문서는 실행 방법을 담당한다

TossInvest WTS 문서는 다음을 담당한다.

- `productCode` 확인
- 한국·미국 주식의 최근 5거래일 OHLCV
- 한국 종목·ETF의 투자자별 수급
- 프로그램 매매, 대차·공매도 등 선택적 확장 자료
- 지수·환율·원자재·뉴스 보조 조회
- 날짜·단위·세션 정합성 검사

### 3. 소스 문서는 폴백 순서를 담당한다

`source-tier-routing.md`의 Tier는 “공식성”만의 순위가 아니다. 다음을 함께 평가한다.

- 현재 Task가 직접 읽을 수 있는가
- 매 거래일 갱신되는가
- 날짜와 지연 여부가 명확한가
- 자료 범위가 충분한가
- 동일한 방식으로 반복 조사할 수 있는가

예를 들어 TossInvest WTS는 비공식·무보증 엔드포인트지만, Task가 공개 웹에서 가격·차트·수급을 반복 조회하기에 유용하므로 이 프로젝트에서는 **Task-direct Tier 1**로 취급한다. 반대로 KRX Open API와 토스증권 공식 Open API는 품질은 높지만 인증·키 관리가 필요한 경우가 있어 **외부 에이전트 전용**으로 분리한다.

### 4. 예시는 규칙이 아니라 가설 템플릿이다

`workflow-examples.md`의 사례는 다음 날 방향을 단정하는 공식이 아니다.

- “영향을 줄 수 있다”
- “하방 헤지 수요와 일치할 수 있다”
- “현물·선물·ETF가 함께 확인될 때 신뢰도가 높아진다”
- “LP, 설정·환매, 리밸런싱 가능성을 배제할 수 없다”

같은 조건부 표현을 사용한다.

## 2. 참조 URL 설정

두 Task Prompt의 `REFERENCE DOCUMENTS`에는 실제 raw URL 또는 연결 문서 URL을 넣는다.

권장 공개 raw URL 패턴:

```text
https://raw.githubusercontent.com/<OWNER>/<REPO>/main/<PATH>
```

`token` 쿼리 파라미터가 붙은 임시 서명 URL이나 세션 쿠키가 필요한 URL은 장기 Scheduled Task에 넣지 않는다. 토큰이 만료되거나 저장소·로그에 노출될 수 있다. 비공개 저장소라면 GitHub connector, 읽기 전용 프록시, 또는 인증된 MCP를 사용한다.

## 3. 브리핑 간 상태 전달

Scheduled Task가 다른 Task의 이전 출력을 자동으로 읽는다고 가정하지 않는다.

- 이전 Morning/Afternoon 출력이 같은 대화, 연결 앱, DB 또는 명시적 입력으로 제공되면 `brief-handoff-contract.md`에 따라 비교한다.
- 접근할 수 없으면 `previous_brief_access: unavailable`로 표시한다.
- 이전 브리핑을 재구성한 경우에는 `reconstructed`, 실제 이전 결과를 읽은 것처럼 표현하지 않는다.

## 4. 시간대 핵심

### Afternoon 15:55 KST

- KRX 현물 정규장 종료 후다.
- KOSPI200 주요 파생 정규장은 통상 15:45까지이므로, 20분 지연 공개값은 아직 최종 구간을 포함하지 않을 수 있다.
- NXT 애프터마켓은 진행 중이며, 20분 지연 공개값은 초기 구간만 반영할 수 있다.
- 미국 주식 프리마켓은 통상 아직 시작 전이다. 세션 상태를 확인하지 않고 프리마켓 가격을 요구하지 않는다.

### Night 20:25 KST

- NXT 20:00 종료 후 20분 지연 최종값을 확인할 수 있는 시점이다.
- KRX 파생 야간세션은 진행 중일 수 있다.
- 미국 주식 프리마켓은 진행 중이며, 정규장은 아직 시작 전이다.
- SK hynix 미국 ADS/ADR의 상장 단계, 세션, 심볼, 거래상태를 매회 확인한다.

## 5. 데이터 철학

```text
Morning: 전일 한국 수급 + 완료된 미국장 + 야간 위험지표로 KRX 개장 전 가설 수립
Afternoon: Morning 가설의 KRX 본장 검증 + 현물/ETF/파생 종가 품질 평가
Night: Morning/Afternoon 가설의 최종 한국 세션 검증 + NXT/KRX 야간파생/미국 프리마켓으로 다음날 조건 갱신
```

방향 예측보다 다음을 우선한다.

1. 세션과 날짜 정합성
2. 종가 형성의 질
3. 가격·수급·ETF·파생 간 일치와 모순
4. 이전 브리핑의 사전 선언 조건을 기준으로 한 평가
5. 데이터가 부족할 때 no-trade 및 confidence cap 적용

## 업데이트 규칙

다음 사건이 발생하면 문서를 재검토한다.

- TossInvest WTS 응답 스키마 또는 경로 변경
- 신규 단일종목 ETF 상장·상장폐지·종목코드 변경
- Binance 주식 연동 선물 심볼 변경 또는 거래 중단
- KRX/NXT 공개 페이지의 지연 시간·메뉴 변경
- Brief 출력 스키마 변경
- 동일 데이터가 반복적으로 Tier 2/3에 의존하는 경우

권장 검토 주기:

- WTS 엔드포인트: 월 1회 또는 오류 발생 즉시
- 주요 종목 코드: 월 1회 및 신규 상장 시
- 공용 소스 링크: 분기 1회
- 워크플로우 사례: 실제 브리핑 회고 후 수시

## 명시적 제외

공개 시장 데이터 수집에서는 다음을 제외한다.

- 프로젝트에서 제외하기로 한 실행 거래소의 공개 가격·funding·OI·링크
- 사용자 커뮤니티 의견, 포럼, 투표형 sentiment
- 날짜가 없는 검색 스니펫
- ETF 가격·거래량을 순자금 유입으로 바꿔 표현한 자료

실제 거래소의 체결·포지션 데이터가 필요하면 브리핑에는 출처를 공개하지 않고 `local execution-venue data`로만 표시하며, 로컬 리스크 엔진에서 별도로 검증한다.

## Scheduled Task 배포 시 중요한 제한

ChatGPT Scheduled Tasks는 **프로젝트에 업로드된 파일을 자동으로 읽지 못할 수 있다**. 따라서 이 문서들을 로컬 저장소에만 두고 Task 프롬프트에서 상대 경로를 적는 것만으로는 충분하지 않다.

배포 방법은 다음 중 하나를 선택한다.

1. 공개 또는 Task가 읽을 수 있는 GitHub raw URL로 문서를 게시한다.
2. Task에서 접근 가능한 연결 앱의 문서 링크를 사용한다.
3. 핵심 규칙만 Task 프롬프트 안에 남기고, 세부 문서는 사람이 필요할 때 참조한다.
4. 외부 에이전트가 문서를 읽고 정제된 JSON을 Task에 제공한다.

`morning-brief-task-prompt.md`에는 아래와 같은 URL 플레이스홀더가 있다.

```text
{{TOSS_WTS_GUIDE_URL}}
{{SOURCE_TIER_GUIDE_URL}}
{{WORKFLOW_EXAMPLES_URL}}
```

저장소에 문서를 게시한 뒤 실제 URL로 치환한다. 참조 문서를 읽지 못한 경우 Task는 읽었다고 주장하지 말고, 프롬프트 안의 최소 규칙으로 진행해야 한다.