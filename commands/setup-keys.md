---
description: 브랜드 데이터 수집에 필요한 API 키를 설정합니다
user_invocable: true
allowed-tools: ["Read", "Write", "AskUserQuestion"]
---

# API 키 설정

이 커맨드는 브랜드 데이터 수집에 사용되는 외부 API 키를 설정합니다.
설정된 키는 `.claude/market-research-agent.local.md`에 저장되며, git에 포함되지 않습니다.

## 실행 단계

### 1단계: 기존 설정 확인

다음 순서로 기존 API 키 설정을 확인합니다:

1. **`backend/.env` 파일 확인 (우선)** — 이미 환경변수로 API 키가 세팅되어 있는지 확인합니다.
   - 키가 있으면: "backend/.env에 이미 설정된 키가 있습니다"라고 안내하고, 설정된 키 목록을 보여줍니다 (키 값은 노출하지 않음).
   - 추가 키만 필요한 경우 `.env`에 직접 추가하도록 안내합니다.
2. **`.claude/market-research-agent.local.md` 파일 확인** — `.env`에 없는 키가 여기 있는지 확인합니다.
3. 두 곳 모두 없는 키만 새로 입력받습니다.

### 2단계: API 키 소개 및 선택

사용자에게 AskUserQuestion으로 어떤 API 키를 설정할지 물어봅니다.

각 API 키에 대해 다음 정보를 안내합니다:

| API | 용도 | 키 없을 때 대체 | 발급처 |
|-----|------|-----------------|--------|
| **DART API** | 재무제표, 기업 개황 | 수집 불가 | https://opendart.fss.or.kr (회원가입 후 즉시 발급) |
| **NewsAPI** | 뉴스 기사 수집 | Google RSS로 대체 (제한적) | https://newsapi.org (무료 100건/일) |
| **GNews API** | 뉴스 보조 소스 | NewsAPI 또는 Google RSS로 대체 | https://gnews.io (무료 100건/일) |
| **Anthropic API** | AI 감성분석, M&A 시그널 분석 | AI 분석 스킵, 원문만 제공 | https://console.anthropic.com (유료, 토큰 과금) |

multiSelect: true로 설정하여 여러 키를 한번에 선택할 수 있게 합니다.

### 3단계: 키 입력 받기

선택된 각 API에 대해 AskUserQuestion으로 키를 하나씩 입력받습니다.
질문 형식: "DART API 키를 입력해주세요 (발급: https://opendart.fss.or.kr)"

**중요**: 사용자가 입력한 키 값을 대화에 그대로 노출하지 마세요. "키가 설정되었습니다" 정도로만 안내합니다.

### 4단계: 설정 파일 저장

`.claude/market-research-agent.local.md` 파일에 YAML frontmatter 형식으로 저장합니다.

기존 파일이 있으면 기존 값을 유지하면서 새로 입력받은 키만 업데이트합니다.
기존 파일이 없으면 새로 생성합니다.

파일 형식:
```markdown
---
dart_api_key: "입력받은_키"
news_api_key: "입력받은_키"
gnews_api_key: ""
anthropic_api_key: ""
---

# Brand Collector API 설정

## 수집 가능 데이터
설정된 키에 따라 아래 데이터를 수집할 수 있습니다.

### 키 없이 수집 가능 (항상 사용 가능)
- 주가/시총: 네이버 금융 (국내), Yahoo Finance (해외)
- 애널리스트 의견: 네이버 금융 (국내 상장사)
- 기본 뉴스: Google News RSS

### API 키 필요
- 재무제표/기업정보: DART API 키 필요
- 상세 뉴스 수집: NewsAPI 또는 GNews 키 필요
- AI 감성분석/M&A 시그널: Anthropic API 키 필요
```

### 5단계: 결과 안내

설정 완료 후 아래를 안내합니다:

1. 어떤 키가 설정되었는지 요약 (키 값은 표시하지 않고 설정 여부만)
2. 현재 설정으로 수집 가능한 데이터 목록
3. 추가 키를 등록하면 더 수집할 수 있는 데이터 안내
4. `/setup-keys`를 다시 실행하면 키를 추가하거나 변경할 수 있다는 안내

결과 예시:
```
API 키 설정 완료

  설정됨:
  - DART API        ✓ → 재무제표, 기업 개황 수집 가능
  - NewsAPI         ✓ → 상세 뉴스 수집 가능

  미설정:
  - GNews API       ✗ → NewsAPI로 대체
  - Anthropic API   ✗ → AI 분석 스킵 (원문만 제공)

  항상 사용 가능:
  - 주가/시총 (네이버 금융, Yahoo Finance)
  - 애널리스트 의견 (네이버 금융)
  - 기본 뉴스 (Google News RSS)

키를 추가하거나 변경하려면 /setup-keys를 다시 실행하세요.
```
