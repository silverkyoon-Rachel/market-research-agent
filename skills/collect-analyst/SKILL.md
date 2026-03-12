---
name: collect-analyst
description: 애널리스트 투자의견, 목표가, 리서치 리포트를 수집합니다. 국내 상장사는 네이버 금융 스크래핑, 해외/비상장사는 WebSearch를 활용합니다.
---

# 애널리스트 투자의견 수집 스킬

이 스킬은 증권사 애널리스트의 투자의견과 목표가를 수집합니다.

## 수집 대상 데이터

- 증권사명 (broker)
- 투자의견 (매수/중립/매도 또는 Buy/Hold/Sell)
- 목표가 (target price)
- 리포트 제목
- 발행일
- 컨센서스 목표가 (평균)
- 의견 분포 (매수/중립/매도 비율)

## 수집 분기 로직

### 1. 국내 상장사 (종목코드 있음)

네이버 금융에서 스크래핑합니다. API 키 불필요.

**수집 절차:**
1. 프로젝트 DB에서 종목코드(stock_code) 확인
2. `backend/collectors/analyst_collector.py`의 `AnalystCollector` 사용
3. 네이버 금융 리서치 페이지 스크래핑:
   - URL: `https://finance.naver.com/research/company_list.naver`
   - 수집: 리포트 제목, 증권사, 투자의견, 목표가, 날짜
4. 엔드포인트: `POST /api/analyst-opinions/collect`

**직접 수집이 안 될 경우:**
- WebSearch로 "{기업명} 증권사 목표가 투자의견 {현재연도}" 검색
- WebFetch로 네이버 금융, 한경 컨센서스, FnGuide 등에서 수집

### 2. 해외 상장사

WebSearch를 통해 수집합니다.

**수집 절차:**
1. WebSearch로 순차 검색:
   - "{company name} analyst ratings target price {current year}"
   - "{company name} consensus estimate buy sell hold"
   - "{company name} broker recommendations"
2. 검색 결과에서 수집:
   - TipRanks, MarketBeat, CNN Money 등 컨센서스 사이트
   - Yahoo Finance analyst recommendations
   - Bloomberg 컨센서스
3. WebFetch로 상세 데이터 추출:
   - 증권사별 투자의견, 목표가
   - 컨센서스 목표가 (평균/중간값)
   - 최근 의견 변경 이력

### 3. 비상장사

비상장사는 공식 애널리스트 커버리지가 거의 없으므로 대체 정보를 수집합니다.

**수집 절차:**
1. WebSearch로 대체 정보 탐색:
   - "{기업명} 기업분석 보고서" 또는 "{company name} company analysis report"
   - "{기업명} 투자 유치 기업가치" (스타트업인 경우)
   - "{기업명} 산업 분석 리포트"
2. 수집 가능한 대체 정보:
   - 산업 리서치 보고서에서 기업 언급
   - VC/PE 투자 관련 보도 (기업가치 추정)
   - 업계 전문가 코멘트
3. "공식 애널리스트 커버리지 없음"을 명시하고, 찾은 대체 정보를 제공

## API 키 조회 순서

API 키는 다음 순서로 확인합니다:
1. `backend/.env` 파일 (프로젝트 환경변수) — Read 도구로 파일을 읽어 해당 키 값을 확인
2. `.claude/market-research-agent.local.md` (플러그인 설정) — 1번에 없을 경우 확인

키 매핑:
| backend/.env | local.md | 용도 |
|---|---|---|
| `ANTHROPIC_API_KEY` | `anthropic_api_key` | AI 종합분석 |
| `NAVER_CLIENT_ID` / `NAVER_CLIENT_SECRET` | — | 네이버 검색 API |

## AI 종합분석 (Anthropic 키가 있는 경우)

위 API 키 조회 순서에 따라 `ANTHROPIC_API_KEY` 확인.

**키가 있으면:**
- `backend/analyzers/analyst_analyzer.py`의 `AnalystAnalyzer` 활용
- 종합의견 (매수/중립/매도/관망)
- 신뢰도, 컨센서스 목표가, 상승여력
- 핵심근거 3가지, 리스크 2가지

**키가 없으면:**
- 수집된 의견의 단순 통계 제공 (매수/중립/매도 비율, 평균 목표가)
- Claude (현재 대화 모델)가 간이 종합분석

## 빠른 실패 규칙

수집 시 아래 규칙을 반드시 준수합니다:

1. **시도 횟수 제한**: 1차 소스 1회 + 대체 소스 최대 2회 = 총 3회 이내
2. **1차 소스 실패 시**: 즉시 WebSearch 1회로 전환. WebSearch에서도 유의미한 데이터가 없으면 "수집 실패" 반환
3. **WebFetch 제한**: 최대 2개 URL까지만 시도. 응답 없거나 파싱 불가 시 즉시 포기
4. **부분 수집 허용**: 컨센서스 목표가만 있고 개별 증권사 데이터가 없어도 OK. 수집된 것만 반환
5. **비상장사 빠른 종료**: WebSearch 1회 시도 후 커버리지 없으면 "공식 애널리스트 커버리지 없음"으로 즉시 반환
6. **실패 반환 형식**: `⚠️ 수집 실패: {사유}. 시도한 소스: {소스 목록}`

**수집 우선순위 (국내 상장사 예시):**
```
네이버 금융 스크래핑 → (실패) → WebSearch "{기업명} 증권사 목표가" → (실패) → 수집 실패 반환
```

## 출력 형식

```
## 애널리스트 의견: {기업명}

### 투자의견 현황 (최근 3개월)

| 증권사 | 투자의견 | 목표가 | 날짜 |
|--------|----------|--------|------|
| OO증권 | 매수 | ₩00,000 | MM-DD |
| XX증권 | 매수 | ₩00,000 | MM-DD |
| △△증권 | 중립 | ₩00,000 | MM-DD |

### 컨센서스 요약
| 항목 | 값 |
|------|-----|
| 컨센서스 목표가 | ₩00,000 |
| 현재가 대비 상승여력 | +00.0% |
| 의견 분포 | 매수 0건 / 중립 0건 / 매도 0건 |

### AI 종합분석
(수집된 의견을 종합한 분석 요약)

수집 소스: 네이버 금융 / WebSearch ({구체적 출처})
수집 시각: YYYY-MM-DD HH:MM
```
