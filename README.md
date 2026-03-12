# Market Research Agent

브랜드/기업의 주가, 재무제표, 뉴스, 애널리스트 의견을 자동 수집하고 마켓 컨센서스 리포트를 생성하는 **Claude Code 플러그인**입니다.

국내/해외, 상장/비상장 기업 모두 지원합니다.

## 설치

### 프로젝트 로컬 설치 (팀 공유)

```bash
cd your-project
git clone https://github.com/silverkyoon-Rachel/market-research-agent.git .claude-plugin
```

### 글로벌 설치 (모든 프로젝트에서 사용)

```bash
git clone https://github.com/silverkyoon-Rachel/market-research-agent.git ~/.claude/plugins/market-research-agent
```

## 사용법

### `/brand-analysis`

기업명을 입력하면 4개 데이터를 **병렬 수집**하고 종합 리포트를 생성합니다.

```
/brand-analysis 영원무역
/brand-analysis Nike
```

**출력 내용:**
- 주가/시가총액/PER/PBR
- 재무제표 (매출, 영업이익, 순이익, 자산, 부채)
- 최신 뉴스 + 감성분석
- 애널리스트 투자의견/목표가
- M&A 매력도 평가

### `/setup-keys`

데이터 수집에 사용할 API 키를 설정합니다.

```
/setup-keys
```

## API 키

| API | 용도 | 키 없을 때 | 발급처 |
|-----|------|-----------|--------|
| **DART API** | 재무제표, 기업 개황 | 수집 불가 | [opendart.fss.or.kr](https://opendart.fss.or.kr) |
| **NewsAPI** | 뉴스 기사 수집 | Google RSS로 대체 | [newsapi.org](https://newsapi.org) |
| **GNews API** | 뉴스 보조 소스 | NewsAPI/RSS로 대체 | [gnews.io](https://gnews.io) |
| **Anthropic API** | AI 감성분석 | AI 분석 스킵 | [console.anthropic.com](https://console.anthropic.com) |

> API 키 없이도 주가, 애널리스트 의견, 기본 뉴스는 수집 가능합니다.

## 지원 기업 유형

| 데이터 | 국내 상장 | 해외 상장 | 국내 비상장 | 해외 비상장 |
|--------|----------|----------|------------|------------|
| 주가/시총 | 네이버 금융 | Yahoo Finance | WebSearch | WebSearch |
| 재무제표 | DART API | SEC/WebSearch | DART API | WebSearch |
| 뉴스 | NewsAPI/RSS | NewsAPI/RSS | NewsAPI/RSS | NewsAPI/RSS |
| 애널리스트 | 네이버 금융 | WebSearch | WebSearch | WebSearch |

## 플러그인 구조

```
market-research-agent/
├── plugin.json
├── commands/
│   ├── brand-analysis.md    # 종합 리서치 오케스트레이터
│   └── setup-keys.md        # API 키 설정
└── skills/
    ├── collect-stock/       # 주가/시총 수집
    ├── collect-financials/  # 재무제표 수집
    ├── collect-news/        # 뉴스 수집 + 감성분석
    ├── collect-analyst/     # 애널리스트 의견 수집
    └── collect-consensus/   # 마켓 컨센서스 종합
```

## 요구사항

- [Claude Code](https://claude.com/claude-code) CLI
- Claude Opus 4.6 / Sonnet 4.6 이상 권장

## License

MIT
