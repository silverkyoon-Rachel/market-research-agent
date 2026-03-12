---
name: collect-news
description: 기업 관련 최신 뉴스를 수집하고 AI로 감성분석/M&A 시그널을 분석합니다. NewsAPI/GNews(키 있으면), Google News RSS, WebSearch를 조합하여 수집합니다.
---

# 뉴스 수집 + AI 분석 스킬

이 스킬은 기업 관련 최신 뉴스를 수집하고, AI로 감성분석 및 M&A 시그널을 분석합니다.

## 수집 대상 데이터

- 기사 제목, URL, 출처, 발행일
- AI 감성분석 (긍정/부정/중립, -1.0~1.0)
- M&A 관련 여부 및 시그널 강도 (0.0~1.0)
- AI 요약 (2-3문장)
- 핵심 키워드 (최대 5개)

## API 키 조회 순서

API 키는 다음 순서로 확인합니다:
1. `backend/.env` 파일 (프로젝트 환경변수) — Read 도구로 파일을 읽어 해당 키 값을 확인
2. `.claude/market-research-agent.local.md` (플러그인 설정) — 1번에 없을 경우 확인

키 매핑:
| backend/.env | local.md | 용도 |
|---|---|---|
| `NEWS_API_KEY` | `news_api_key` | NewsAPI |
| `GNEWS_API_KEY` | `gnews_api_key` | GNews |
| `ANTHROPIC_API_KEY` | `anthropic_api_key` | AI 감성분석 |

## 수집 분기 로직

### 1단계: API 기반 수집 (키가 있는 경우)

위 API 키 조회 순서에 따라 키를 확인합니다.

**NewsAPI 키가 있으면:**
- `backend/collectors/news_collector.py`의 뉴스 수집 로직 활용
- 엔드포인트: NewsAPI `/v2/everything`
- 검색어: 기업명 (한국어/영어 모두)
- 기간: 최근 30일
- 정렬: publishedAt (최신순)
- 제한: 무료 100건/일

**GNews 키가 있으면:**
- NewsAPI 보조/백업 소스로 사용
- 엔드포인트: GNews `/v4/search`
- 동일 검색어, 최근 30일

### 2단계: Google News RSS (항상 사용 가능)

API 키 유무와 관계없이 항상 실행합니다.

**수집 절차:**
1. Google News RSS 피드 조회
   - 국내: `https://news.google.com/rss/search?q={기업명}&hl=ko&gl=KR`
   - 해외: `https://news.google.com/rss/search?q={company name}&hl=en&gl=US`
2. feedparser로 파싱 (제목, 링크, 발행일, 출처)
3. 최근 20건 수집

### 3단계: WebSearch 보강 (항상 사용 가능)

API와 RSS에서 부족한 경우 WebSearch로 보강합니다.

**검색 쿼리:**
- 국내: "{기업명} 최신 뉴스 {현재연도}"
- 해외: "{company name} latest news {current year}"
- M&A 특화: "{기업명} 인수 합병" 또는 "{company name} acquisition merger"
- 실적 특화: "{기업명} 실적 발표" 또는 "{company name} earnings report"

**WebSearch 결과에서 추출:**
- 기사 제목, URL, 출처
- 스니펫에서 핵심 내용 파악

### 4단계: AI 감성분석 (Anthropic 키가 있는 경우)

위 API 키 조회 순서에 따라 `ANTHROPIC_API_KEY` 확인.

**키가 있으면:**
- `backend/analyzers/news_analyzer.py`의 `NewsAnalyzer` 활용
- 각 기사별 분석:
  - 감성 (positive/negative/neutral)
  - 감성점수 (-1.0 ~ 1.0)
  - M&A 관련 여부
  - M&A 시그널 강도 (0.0 ~ 1.0)
  - AI 요약 (2-3문장)
  - 키워드 (최대 5개)

**키가 없으면:**
- Claude (현재 대화의 모델)가 직접 수집된 기사 제목과 스니펫을 분석
- 간이 감성 판단 (제목 기반)
- M&A 키워드 매칭 (인수, 합병, 매각, acquisition, merger, takeover 등)

## 빠른 실패 규칙

수집 시 아래 규칙을 반드시 준수합니다:

1. **시도 횟수 제한**: API 1회 + RSS 1회 + WebSearch 1회 = 총 3회 이내
2. **API 실패 시**: 즉시 Google News RSS로 전환. RSS도 실패 시 WebSearch 1회만 시도
3. **WebFetch 제한**: 최대 2개 URL까지만 시도. 응답 없거나 파싱 불가 시 즉시 포기
4. **부분 수집 허용**: 뉴스 1건이라도 수집되면 그것만 정리하여 반환
5. **실패 반환 형식**: `⚠️ 수집 실패: {사유}. 시도한 소스: {소스 목록}`

**수집 우선순위:**
```
NewsAPI/GNews → (실패) → Google News RSS → (실패) → WebSearch 1회 → (실패) → 수집 실패 반환
```

## 출력 형식

```
## 최신 뉴스: {기업명}

### 주요 뉴스 (최근 30일, {N}건 수집)

| # | 제목 | 출처 | 날짜 | 감성 | M&A |
|---|------|------|------|------|-----|
| 1 | 기사 제목... | 출처 | MM-DD | 긍정 +0.7 | ✓ 0.8 |
| 2 | 기사 제목... | 출처 | MM-DD | 중립 +0.1 | ✗ |
| 3 | 기사 제목... | 출처 | MM-DD | 부정 -0.5 | ✗ |

### 뉴스 감성 요약
- 전체 감성 트렌드: 긍정적 (평균 +0.3)
- M&A 관련 기사: N건 (시그널 강도 평균 0.0)
- 핵심 키워드: 키워드1, 키워드2, 키워드3

### AI 분석 요약
(수집된 뉴스를 종합한 2-3문장 요약)

수집 소스: NewsAPI / Google RSS / WebSearch
수집 시각: YYYY-MM-DD HH:MM
```
