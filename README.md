# Stock Report - DART 재무제표 리포트 생성기

DART OpenAPI를 활용하여 한국 상장사(KOSPI/KOSDAQ)의 재무제표 데이터를 자동 조회하고, 인터랙티브 HTML 리포트를 생성합니다.

## Features

- **DART API 자동 조회**: 종목코드 또는 회사명 입력 → 재무데이터 자동 수집
- **연간 + 분기 데이터**: 최근 4개년 연간 실적 + 최신 분기별 실적
- **연결/별도 재무제표**: CFS(연결), OFS(별도) 전환 지원
- **인터랙티브 차트**: Chart.js 기반 재무상태/손익 추이 시각화
- **재무비율 분석**: 부채비율, 유동비율, 영업이익률, 순이익률, ROE, ROA
- **다크 테마 UI**: React 18 + Tailwind CSS 기반 단일 HTML 파일

## Generated Reports

| 종목코드 | 회사명 | 시장 |
|----------|--------|------|
| [003000](dart/003000.html) | 부광약품 | KOSPI |
| [005930](dart/005930.html) | 삼성전자 | KOSPI |
| [035720](dart/035720.html) | 카카오 | KOSPI |
| [082920](dart/082920.html) | 비츠로셀 | KOSDAQ |

## Project Structure

```
stock_report/
├── .claude/agents/
│   ├── dart-report.md          # DART 리포트 생성 에이전트
│   └── single-react-dev.md     # React 단일파일 개발 에이전트
├── dart/
│   ├── {종목코드}.html          # 생성된 재무제표 리포트
│   └── corpcode/
│       └── corpcode.csv         # DART 상장사 기업코드 매핑 (3,954사)
├── calculator/
│   └── index.html               # 공학용 계산기
├── .env                         # DART API KEY (gitignore)
└── .gitignore
```

## Usage

### Claude Code 에이전트로 리포트 생성

```
# 종목코드로 생성
"005930 삼성전자 리포트 만들어줘"

# 회사명으로 생성
"카카오 재무제표 리포트 생성해줘"
```

### 필요 사항

- **DART API Key**: [OpenDART](https://opendart.fss.or.kr)에서 무료 발급
- `.env` 파일에 `DART_API_KEY=your_key_here` 형식으로 저장

## Data Sources

- [DART 전자공시시스템](https://dart.fss.or.kr) - 재무제표 데이터
- [OpenDART API](https://opendart.fss.or.kr) - 기업정보 및 재무데이터 API

## Tech Stack

- **Frontend**: React 18, Tailwind CSS, Chart.js (CDN)
- **Data**: DART OpenAPI (`fnlttSinglAcntAll.json`)
- **Agent**: Claude Code custom agent (`dart-report.md`)

---

> 본 자료는 참고용이며, 투자 판단의 근거로 사용될 수 없습니다.
