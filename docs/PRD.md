# PRD: QQQM 데일리 마켓 브리핑 에이전트

- **작성일**: 2026-10-24
- **작성자**: Yoo
- **상태**: Draft v0.1

---

## 1. 배경 & 문제

- QQQM(Invesco NASDAQ 100 ETF)은 매일 오르락내리락하는데, **왜 움직였는지** 매번 직접 뉴스를 찾아보기가 번거롭다.
- ETF 시장 전반의 흐름(지수, 섹터, 자금 유입·유출 등)도 한눈에 보고 싶다.
- Slack 무료 플랜은 **최근 90일 메시지만 볼 수 있어서** 기록이 오래 남지 않는다. 지난 브리핑을 다시 보거나 패턴을 돌아볼 수 없다.

## 2. 목표

1. 미국장이 끝나면 **자동으로** QQQM의 등락과 그 **이유**를 분석해서 알려준다.
2. ETF 시장 전체 소식과 현재 시장 분위기를 함께 요약한다.
3. 결과를 **Slack으로 전송**한다 (빠르게 확인하는 용도).
4. 같은 내용을 **Obsidian에 마크다운으로 쌓는다** (오래 보관하고 다시 찾아보는 용도).

## 3. 비목표 (Non-goals)

- 매수/매도 추천이나 목표가 제시는 하지 않는다. 사실과 원인 분석만 다룬다.
- 실시간(장중) 알림은 MVP 범위에서 제외한다.
- 자동 매매 기능은 만들지 않는다.

## 4. 사용자

- 본인 1명 (QQQM 보유 또는 관심 있는 개인 투자자)
- 한국에 살고 있어서 **아침에 일어나서** 전날 미국장 결과를 확인하는 패턴

## 5. 핵심 사용 흐름

```
[스케줄 트리거: 매일 아침, 미국장 마감 후]
        │
        ▼
① 데이터 수집: QQQM 시세, 주요 구성종목 시세, 매크로 지표, 뉴스, 경제 일정
        │
        ▼
② 분석 (Claude): 왜 올랐나/내렸나, 시장 분위기, ETF 전반 소식 요약
        │
        ├── Slack MCP ──────► ③ Slack 전송 (요약본)
        │
        └── Obsidian MCP ───► ④ Obsidian 저장 (전체본 .md)
```

- 에이전트는 Claude가 **MCP 도구**를 직접 호출하는 구조로 만든다. Webhook URL을 코드에 넣지 않는다.

## 6. 기능 요구사항

### F1. 자동 실행 (트리거)

| 항목 | 내용 |
|---|---|
| 실행 시점 | 미국 정규장 마감(16:00 ET) 후. 한국 시간 기준 **화~토 오전 7:00** |
| 서머타임 | 미국 서머타임 해제(2026-11-01) 후에는 마감이 KST 06:00가 되므로, 07:00으로 고정하면 1년 내내 안전함 |
| 휴장일 | NYSE 휴장일에는 분석을 건너뛰고 "오늘은 휴장" 짧은 메시지만 보냄 (또는 보내지 않음, 선택) |
| 수동 실행 | 필요할 때 명령어 한 번으로 바로 실행할 수 있어야 함 |
| 중복 방지 | 같은 날짜 리포트가 이미 있으면 다시 보내지 않음 |

### F2. QQQM 등락 원인 분석 (핵심 기능)

- **기본 수치**: 종가, 전일 대비 등락률(%), 거래량(평소 대비), 52주 고점 대비 위치
- **기여도 분석**: 상위 구성종목(NVDA, MSFT, AAPL, AMZN, AVGO, META, GOOGL, TSLA 등)의 `비중 × 등락률`로 "어떤 종목이 지수를 끌어올렸/내렸는지" 계산
  - 예: "NVDA가 -4.2% 빠지면서 QQQM 하락분의 약 40%를 차지"
- **원인 설명**: 뉴스와 지표를 근거로 오늘 움직임의 주요 이유 **3개 이내** 정리
  - 개별 종목 이슈 (실적, 가이던스, 규제, 신제품)
  - 매크로 (CPI·고용지표, FOMC·연준 발언, 10년물 금리, 달러 인덱스)
  - 시장 심리 (VIX, 섹터 로테이션, 위험 선호/회피)
- **근거 표시**: 각 이유마다 출처 뉴스 링크를 붙임. 근거가 약하면 "뚜렷한 재료 없음, 수급/차익실현 추정"처럼 솔직하게 표시

### F3. 현재 시장 상황 요약

- 주요 지수: S&P 500, 나스닥 100, 다우, 러셀 2000
- 매크로 지표: 미 10년물 금리, 달러 인덱스(DXY), VIX, WTI 유가, 비트코인(참고용)
- 한 줄 시장 분위기: 예) "금리 상승에 기술주 약세, 위험회피 분위기"

### F4. ETF 전체 소식

- 주요 ETF 등락: QQQ, SPY, VOO, SCHD, SMH, XLK, TLT 등 (관심 목록은 설정 파일로 관리)
- ETF 관련 주요 뉴스: 자금 유입·유출, 신규 상장, 리밸런싱, 수수료 변경 등
- QQQM에 영향을 줄 만한 이벤트: 나스닥 100 리밸런싱, 구성종목 변경 등

### F5. 다가오는 일정

- 다음 거래일의 경제지표 발표 (CPI, PCE, 고용, FOMC 등)
- 주요 구성종목 실적 발표 일정 (향후 1주)

### F6. Slack 전송

- **Slack MCP**(공식 커넥터, `https://mcp.slack.com/mcp`)의 `slack_send_message`로 지정한 채널에 전송
  - OAuth로 내 Slack 계정에 연결하므로 Webhook URL을 따로 관리할 필요가 없음
- 모바일에서 읽기 쉬운 **요약본** (스크롤 1~2번 안에 다 읽히는 길이)
  - 상단: 📈/📉 QQQM 종가와 등락률, 한 줄 요약
  - 중간: 오른/내린 이유 3개, 기여도 상위 종목
  - 하단: 시장 분위기, 내일 일정
- 실패하면 최대 3회 재시도, 그래도 실패하면 로그 남기기 (Obsidian 저장은 그대로 진행)

### F7. Obsidian 저장 (장기 보관)

- Slack보다 자세한 **전체본**을 마크다운 파일로 저장
- **Obsidian MCP**로 Vault에 파일을 씀
  - 공식 filesystem MCP 서버(`@modelcontextprotocol/server-filesystem`)를 Vault 폴더(`/Users/yoomac/Documents/Obsidian Vault`)에만 접근하도록 연결
  - Obsidian 앱이 꺼져 있어도 저장되고, 앱을 열면 바로 보임
- 폴더 구조 (Slack 채널과 1:1 대응, 자세한 건 "6-1. 분류 체계" 참고):
  ```
  Obsidian Vault/Market/
  ├── 00-Index.md          ← 허브 노트 (Dataview로 최근 리포트 표)
  ├── 01-Daily/2026/10/2026-10-23.md
  ├── 02-ETF/2026/10/2026-10-23.md
  ├── 03-Macro/2026/10/2026-10-23.md
  ├── 04-Weekly/2026/2026-W43.md
  ├── 05-Tickers/NVDA.md   ← 종목별 누적 노트
  ├── 99-Logs/2026/10/2026-10-23.md
  └── _Templates/          ← daily, etf, macro, weekly, ticker, log
  ```
- 파일마다 **YAML frontmatter**를 넣어서 나중에 Dataview로 검색·통계를 낼 수 있게 함
  ```yaml
  ---
  date: 2026-10-23
  category: daily
  ticker: QQQM
  close: 245.12
  change_pct: -1.34
  mood: risk-off
  top_movers: [NVDA, AAPL, MSFT]
  tags: [market/daily, qqqm]
  ---
  ```
- 종목명은 `[[NVDA]]`처럼 링크로 써서 Obsidian 그래프 뷰에서 종목별로 모아볼 수 있게 함
- 같은 날짜 파일이 있으면 덮어쓰지 않고 건너뜀

## 6-1. 분류 체계 (Slack 채널 ↔ Obsidian 폴더)

리포트를 종류별로 나눠서, Slack에서는 채널로, Obsidian에서는 폴더로 똑같이 분류한다. 설정은 [`config/destinations.yaml`](../config/destinations.yaml)에 모아둔다.

| 종류 | Slack 채널 (비공개) | Obsidian 폴더 | 내용 | 단계 |
|---|---|---|---|---|
| daily | `#market-daily` | `Market/01-Daily/` | QQQM 등락 원인 + 시장 분위기 | MVP |
| etf | `#market-etf` | `Market/02-ETF/` | ETF 전반 소식 | MVP |
| macro | `#market-macro` | `Market/03-Macro/` | 금리·달러·VIX, 경제지표·실적 일정 | MVP |
| weekly | `#market-weekly` | `Market/04-Weekly/` | 주간 요약 | v2 |
| ticker | (없음) | `Market/05-Tickers/` | 종목별 누적 노트 | MVP |
| log | `#market-bot-log` | `Market/99-Logs/` | 실행 성공/실패 기록 | MVP |

- 채널 이름은 모두 `market-`으로 시작해서 Slack 사이드바에서 한데 모인다.
- 폴더 이름 앞의 숫자(01~99)는 Obsidian에서 보기 좋은 순서로 정렬하려는 용도다.
- 모든 노트의 frontmatter에 `category`를 넣어서 폴더와 상관없이 검색할 수 있게 한다.
- 데일리 노트에서 `[[NVDA]]`처럼 종목을 링크하면 `05-Tickers/NVDA.md`에 그 종목이 언급된 날이 자동으로 모인다.

## 6-2. 코드 패키지 구조 (제안)

숫자는 Python 코드가 API에서 가져오고, 해석과 전송은 Claude가 MCP로 한다. 이렇게 나누면 LLM이 숫자를 지어내지 않는다.

```
stock-news-agent/
├── docs/
│   └── PRD.md
├── config/
│   ├── destinations.yaml     # 종류 → Slack 채널 ID, Obsidian 경로 (✅ 생성됨)
│   └── watchlist.yaml        # QQQM, 관심 ETF, 매크로 지표 목록
├── prompts/
│   ├── daily_brief.md        # 예약 작업이 매일 실행할 메인 프롬프트
│   └── weekly_brief.md       # 주간 요약 프롬프트 (v2)
├── src/stock_news_agent/
│   ├── collectors/           # 데이터 수집 (외부 API 호출만 담당)
│   │   ├── prices.py         #   시세 (QQQM, 구성종목, ETF, 지수)
│   │   ├── holdings.py       #   QQQM 구성종목 비중
│   │   ├── news.py           #   뉴스
│   │   ├── macro.py          #   금리, 달러, VIX, 유가
│   │   └── calendar.py       #   경제지표·실적 일정, 휴장일
│   ├── analysis/             # 계산 (LLM 없이 숫자만)
│   │   └── attribution.py    #   종목별 기여도 = 비중 × 등락률
│   ├── models/               # 데이터 형태 정의
│   │   └── snapshot.py       #   하루치 수집 결과 (MarketSnapshot)
│   └── cli.py                # `collect --date 2026-10-23` → JSON 출력
├── tests/
├── .env.example              # 데이터 API 키 자리 (실제 .env는 git 제외)
└── pyproject.toml
```

**하루 실행 흐름**
1. 예약 작업이 `prompts/daily_brief.md`로 Claude를 실행한다.
2. Claude가 `cli.py collect`를 돌려서 그날 숫자와 뉴스를 JSON으로 받는다.
3. Claude가 원인을 분석하고 종류별(daily, etf, macro)로 리포트를 쓴다.
4. `config/destinations.yaml`을 보고 Slack MCP로 각 채널에 보내고, Obsidian MCP로 각 폴더에 저장한다.
5. 결과를 `#market-bot-log`와 `99-Logs/`에 남긴다.

## 7. 비기능 요구사항

| 항목 | 기준 |
|---|---|
| 언어 | 리포트는 **한국어**로 작성 (종목 티커와 지표명은 영어 그대로) |
| 정확성 | 수치(종가, 등락률)는 데이터 API 값을 그대로 쓰고, LLM이 숫자를 만들어내지 않게 함 |
| 비용 | 월 운영비 최소화 (무료 API 우선, Claude API 사용량은 하루 1회 실행 수준) |
| 보안 | 데이터 API 키는 `.env`에 보관하고 git에 올리지 않음. Slack은 MCP OAuth로 연결하고, Obsidian MCP는 Vault 폴더 밖에는 접근하지 못하게 제한 |
| 면책 | 리포트 하단에 "투자 권유가 아닌 정보 제공 목적" 문구 포함 |

## 8. 기술 구성 (초안)

| 영역 | 후보 |
|---|---|
| 언어 | Python |
| 시세 데이터 | yfinance (무료), 필요하면 Finnhub / Alpha Vantage |
| 뉴스 | Finnhub News, NewsAPI, 주요 언론 RSS |
| 구성종목 비중 | Invesco 공식 홀딩스 CSV (주 1회 갱신) |
| 분석·요약 | Claude (MCP 도구를 호출하는 에이전트) |
| 알림 | **Slack MCP** (공식 커넥터) |
| 저장 | **Obsidian MCP** (filesystem MCP, Vault 폴더 한정) |
| 스케줄 | Claude 데스크톱 앱 예약 작업(추천). 아래 "결정 필요 사항" 참고 |

### MCP 연결 현황

| MCP | 방식 | 상태 |
|---|---|---|
| Slack | claude.ai 커넥터 (OAuth), 워크스페이스 `yoo-ka87999` | ✅ 연결됨 (2026-10-24, 나에게 DM으로 테스트 전송 성공) |
| Obsidian | `npx @modelcontextprotocol/server-filesystem@2.0.0 "/Users/yoomac/Documents/Obsidian Vault"` (이 프로젝트 local 설정) | ✅ 연결됨 |

## 9. 결정이 필요한 사항

1. **어디서 실행할까?**
   - Obsidian MCP가 내 Mac의 Vault 폴더에 직접 쓰는 방식이라, **내 Mac에서 실행**하는 게 자연스러움
   - 👉 추천: Claude 데스크톱 앱의 예약 작업으로 매일 실행. 단, 실행 시간에 Mac이 켜져 있어야 함 (잠자기 상태면 깨어날 때 실행되는지 확인 필요)
2. ~~Obsidian Vault 위치~~ → `/Users/yoomac/Documents/Obsidian Vault`로 확정
3. **알림 시간**: 오전 7:00 괜찮은지
4. ~~Slack 채널~~ → 종류별 전용 채널 5개로 확정 ("6-1. 분류 체계" 참고)
4. **관심 ETF 목록**: QQQM 외에 같이 보고 싶은 ETF가 있는지
5. **휴장일 처리**: 휴장 메시지를 보낼지, 아예 조용히 넘어갈지

## 10. 단계별 범위

### MVP (v1)
- F1 자동 실행, F2 QQQM 원인 분석, F3 시장 요약, F6 Slack, F7 Obsidian 저장

### v2
- F4 ETF 전체 소식 확장, F5 일정
- 주간 요약 리포트 (매주 토요일, 한 주 등락 정리 → `Weekly/`)
- 장 시작 전 프리마켓 브리핑 (선택)

### v3 (아이디어)
- Slack에서 질문하면 답해주기 ("NVDA 요즘 왜 이래?")
- Obsidian에 쌓인 과거 기록을 참고해서 "지난번 비슷한 상황" 비교

## 11. 성공 기준

- 거래일의 **95% 이상** 오전 7:30 전에 Slack 도착
- 매 리포트마다 출처가 있는 원인 설명 **최소 1개 이상**
- Obsidian에 빠지는 날 없이 매일 1개씩 파일이 쌓임
- 직접 뉴스를 찾아보는 시간이 줄었다고 느끼기 (주관적 만족도)
