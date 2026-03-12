---
description: 브랜드/기업 종합 리서치 분석. 주가, 재무제표, 뉴스, 애널리스트 의견을 병렬로 수집하고 마켓 컨센서스 리포트를 생성합니다.
user_invocable: true
allowed-tools: ["Read", "Write", "WebSearch", "WebFetch", "Bash", "Agent", "Grep", "Glob", "AskUserQuestion"]
---

# 브랜드 종합 리서치 분석

이 커맨드는 지정된 기업에 대해 모든 데이터를 병렬로 수집하고 종합 리포트를 생성하는 오케스트레이터입니다.

## 실행 단계

### 1단계: 기업 식별

사용자가 입력한 기업명으로 유형을 판별합니다.

**판별 순서:**
1. 프로젝트 DB 확인 — `backend/database/` 에 등록된 브랜드인지 확인
2. 종목코드 유무로 상장/비상장 판별
3. 종목코드 형태로 국내/해외 판별:
   - 6자리 숫자 (예: 005380) → **국내 상장사**
   - 알파벳 티커 (예: NKE, LVMH.PA) → **해외 상장사**
   - 종목코드 없음 → **비상장사**

**DB에 없는 기업일 경우:**
- WebSearch로 "{기업명} 종목코드" 또는 "{company name} stock ticker" 검색
- 검색 결과로 유형 판별
- 판별이 안 되면 AskUserQuestion으로 사용자에게 확인:
  - "이 기업의 유형을 선택해주세요: 국내 상장사 / 해외 상장사 / 국내 비상장사 / 해외 비상장사"

판별 결과를 기록:
```
기업명: {name}
유형: {국내상장 / 해외상장 / 국내비상장 / 해외비상장}
종목코드/티커: {있으면 기록}
```

### 2단계: API 키 확인

다음 순서로 API 키를 확인합니다:
1. **`backend/.env` 파일 확인 (우선)** — Read 도구로 파일을 읽어 설정된 키를 파악
2. **`.claude/market-research-agent.local.md` 확인** — 1번에 없는 키가 있는지 확인

키 매핑:
| backend/.env | local.md | 용도 |
|---|---|---|
| `DART_API_KEY` | `dart_api_key` | DART 재무제표 |
| `NEWS_API_KEY` | `news_api_key` | NewsAPI |
| `GNEWS_API_KEY` | `gnews_api_key` | GNews |
| `ANTHROPIC_API_KEY` | `anthropic_api_key` | AI 분석 |
| `FSC_API_KEY` | — | 금융위원회 기업재무정보 |
| `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET` | — | 네이버 검색 API |

두 곳 모두 키가 없으면:
- 키 없이 수집 가능한 데이터만 수집 (WebSearch, 네이버 금융, Yahoo Finance, Google RSS)
- 리포트 마지막에 "/setup-keys로 API 키를 설정하면 더 정확한 데이터를 수집할 수 있습니다" 안내

### 3단계: 병렬 수집

**반드시 Agent 도구를 사용하여 아래 4개를 동시에 병렬 실행합니다.**
하나의 메시지에서 4개의 Agent 도구 호출을 동시에 보내세요.

각 Agent에게 전달할 정보:
- 기업명, 유형 (국내상장/해외상장/국내비상장/해외비상장)
- 종목코드/티커 (있으면)
- 사용 가능한 API 키 목록
- 프로젝트 경로: 이 프로젝트의 루트 디렉토리

**⚡ 빠른 실패 규칙 (모든 Agent 공통):**
각 Agent 프롬프트에 반드시 아래 규칙을 포함하세요:
```
## 빠른 실패 규칙 (반드시 준수)
- 1차 소스(API/스크래핑) 시도 → 실패 시 WebSearch 1회만 시도 → 그래도 실패 시 즉시 "수집 실패" 반환
- WebFetch는 최대 2개 URL까지만 시도. 응답이 없거나 파싱 불가하면 즉시 포기
- 전체 수집 시도를 3회 이내로 제한 (1차 소스 1회 + 대체 소스 최대 2회)
- 부분 데이터라도 수집되면 그것만 정리하여 반환 (완벽한 데이터를 기다리지 말 것)
- "수집 실패" 시 반환 형식: `⚠️ 수집 실패: {실패 사유}. 시도한 소스: {소스 목록}`
```

#### Agent 1: 주가/시총 수집
```
프롬프트: "{기업명}"의 주가, 시가총액, PER, PBR 등 주식 데이터를 수집해줘.
유형: {유형}, 종목코드: {코드}

collect-stock 스킬을 참조하여 수집해줘.
스킬 위치: .claude-plugin/skills/collect-stock/SKILL.md

{빠른 실패 규칙 전문 삽입}

결과는 마크다운 표 형식으로 정리해줘.
```

#### Agent 2: 재무제표 수집
```
프롬프트: "{기업명}"의 재무제표(매출액, 영업이익, 순이익, 자산, 부채)를 수집해줘.
유형: {유형}
사용 가능한 API 키: {DART_API_KEY 유무}

collect-financials 스킬을 참조하여 수집해줘.
스킬 위치: .claude-plugin/skills/collect-financials/SKILL.md

{빠른 실패 규칙 전문 삽입}

결과는 마크다운 표 형식으로 정리해줘.
```

#### Agent 3: 뉴스 수집
```
프롬프트: "{기업명}"의 최근 뉴스를 수집하고 감성분석해줘.
사용 가능한 API 키: {NEWS_API_KEY, GNEWS_API_KEY, ANTHROPIC_API_KEY 유무}

collect-news 스킬을 참조하여 수집해줘.
스킬 위치: .claude-plugin/skills/collect-news/SKILL.md

{빠른 실패 규칙 전문 삽입}

결과는 마크다운 표 형식으로 정리해줘.
```

#### Agent 4: 애널리스트 의견 수집
```
프롬프트: "{기업명}"의 애널리스트 투자의견, 목표가를 수집해줘.
유형: {유형}, 종목코드: {코드}

collect-analyst 스킬을 참조하여 수집해줘.
스킬 위치: .claude-plugin/skills/collect-analyst/SKILL.md

{빠른 실패 규칙 전문 삽입}

결과는 마크다운 표 형식으로 정리해줘.
```

### 4단계: 결과 통합

4개 Agent의 결과가 모두 돌아오면, collect-consensus 스킬을 참조하여 종합 리포트를 생성합니다.

**통합 순서:**
1. 각 Agent 결과를 수신
2. 누락된 데이터 확인 (어떤 수집이 실패했는지)
3. 실패한 수집은 해당 섹션에 `⚠️ 수집 실패` 표시하고, 성공한 데이터만으로 리포트 작성
4. collect-consensus 스킬의 출력 형식에 맞춰 종합 리포트 작성 (부분 데이터로도 분석 진행)
5. M&A 매력도 평가 포함

**실패 처리 원칙:**
- 4개 중 일부만 성공해도 리포트를 생성한다 (수집 성공한 데이터만으로 분석)
- 4개 모두 실패한 경우에만 "데이터 수집에 실패했습니다" 메시지 출력
- 실패 원인과 대안을 간단히 안내 (예: "API 키 미설정", "해당 기업 데이터 없음")

### 5단계: 최종 출력

아래 형식으로 종합 리포트를 출력합니다:

```
# 📊 {기업명} 종합 리서치 리포트

> 유형: {국내상장/해외상장/비상장} | 리서치 일시: YYYY-MM-DD HH:MM

---

## 1. 주가/시총
(Agent 1 결과)

## 2. 재무제표
(Agent 2 결과)

## 3. 최신 뉴스 동향
(Agent 3 결과)

## 4. 애널리스트 의견
(Agent 4 결과)

## 5. 마켓 컨센서스 종합
(collect-consensus 스킬 기반 종합 분석)

---

### 수집 현황
| 데이터 | 소스 | 상태 |
|--------|------|------|
| 주가 | {소스명} | ✓ 수집 완료 |
| 재무제표 | {소스명} | ✓ 수집 완료 |
| 뉴스 | {소스명} | ✓ {N}건 수집 |
| 애널리스트 | {소스명} | ✓ {N}건 수집 |

### 참고사항
- {키 미설정으로 인한 제한 사항 안내}
- {비상장사라 수집 못한 데이터 안내}
- 더 정확한 데이터를 원하시면 /setup-keys로 API 키를 설정하세요.
```

## 유형별 수집 매트릭스

커맨드가 기업 유형에 따라 어떤 스킬을 어떻게 실행하는지 참고용입니다.

| 스킬 | 국내 상장 | 해외 상장 | 국내 비상장 | 해외 비상장 |
|------|----------|----------|------------|------------|
| collect-stock | 네이버 금융 | Yahoo Finance | WebSearch (추정가치) | WebSearch (추정가치) |
| collect-financials | DART API → WebSearch | SEC/WebSearch | DART API → WebSearch | WebSearch |
| collect-news | NewsAPI/RSS/WebSearch | NewsAPI/RSS/WebSearch | NewsAPI/RSS/WebSearch | NewsAPI/RSS/WebSearch |
| collect-analyst | 네이버 금융 | WebSearch (TipRanks 등) | WebSearch (대체정보) | WebSearch (대체정보) |
