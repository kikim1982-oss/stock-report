---
name: dart-report
description: "주식 종목번호 또는 회사명을 입력받아 DART API로 재무데이터를 조회하고, dart/{종목코드}.html 리포트 파일을 자동 생성하는 에이전트. 연간(과거 3개년+최신) 및 분기 데이터를 포함한 인터랙티브 재무제표 대시보드를 만든다."
model: opus
---

# DART 재무제표 리포트 생성 에이전트

종목코드 또는 회사명을 받아 DART API로 데이터를 조회하고 `dart/{종목코드}.html` 리포트를 생성합니다.

## 성능 목표

**tool_uses 최소화가 핵심.** LLM 라운드트립 1회 = ~20초. curl 자체는 1-2초.
- 목표: **총 5-7회 tool use**, 2-3분 내 완료
- 절대 curl을 개별 Bash 호출로 나누지 말 것. 모든 API 호출을 단일 bash 스크립트에서 병렬 실행.

## Step 1: 기업코드 조회 + API 키 로드 + 기업정보 (Bash 1회)

```bash
cd "C:/Users/우진산업 연구소/Downloads/stock_report"
API_KEY=$(grep DART_API_KEY .env | cut -d= -f2 | tr -d '\r')

# 종목코드로 검색 (또는 회사명으로: grep ",회사명$" ...)
CORP_LINE=$(grep "^{종목코드}," dart/corpcode/corpcode.csv)
CORP_CODE=$(echo "$CORP_LINE" | cut -d, -f2)
echo "CORP: $CORP_LINE"

# 기업 기본정보
curl -s "https://opendart.fss.or.kr/api/company.json?crtfc_key=${API_KEY}&corp_code=${CORP_CODE}"
```

응답에서 추출: corp_name, corp_name_eng, ceo_nm, corp_cls(Y=KOSPI/K=KOSDAQ), adres, hm_url, est_dt(YYYYMMDD→YYYY.MM.DD), acc_mt

## Step 2: 모든 재무제표 데이터 일괄 조회 (Bash 1회)

**단일 bash에서 15개 curl을 백그라운드(&)로 병렬 실행, 결과를 JSON 파일로 저장 후 cat.**

최신 연도 판단: 12월 결산이면 현재 3월 이후 → 전년 사업보고서 존재. 2025부터 시도, 없으면 2024.

```bash
cd "C:/Users/우진산업 연구소/Downloads/stock_report"
API_KEY=$(grep DART_API_KEY .env | cut -d= -f2 | tr -d '\r')
CC="{CORP_CODE}"
BASE="https://opendart.fss.or.kr/api/fnlttSinglAcntAll.json?crtfc_key=${API_KEY}&corp_code=${CC}"
OUT="/tmp/dart_${CC}"
mkdir -p "$OUT"

# 연간 사업보고서 (11011) - CFS + OFS × 4년 = 8호출
curl -s "${BASE}&bsns_year=2025&reprt_code=11011&fs_div=CFS" -o "${OUT}/a25c.json" &
curl -s "${BASE}&bsns_year=2024&reprt_code=11011&fs_div=CFS" -o "${OUT}/a24c.json" &
curl -s "${BASE}&bsns_year=2023&reprt_code=11011&fs_div=CFS" -o "${OUT}/a23c.json" &
curl -s "${BASE}&bsns_year=2022&reprt_code=11011&fs_div=CFS" -o "${OUT}/a22c.json" &
curl -s "${BASE}&bsns_year=2025&reprt_code=11011&fs_div=OFS" -o "${OUT}/a25o.json" &
curl -s "${BASE}&bsns_year=2024&reprt_code=11011&fs_div=OFS" -o "${OUT}/a24o.json" &
curl -s "${BASE}&bsns_year=2023&reprt_code=11011&fs_div=OFS" -o "${OUT}/a23o.json" &
curl -s "${BASE}&bsns_year=2022&reprt_code=11011&fs_div=OFS" -o "${OUT}/a22o.json" &

# 분기 (최신연도) - 1Q(11013) + 반기(11012) + 3Q(11014) × CFS/OFS = 6호출
curl -s "${BASE}&bsns_year=2025&reprt_code=11013&fs_div=CFS" -o "${OUT}/q1c.json" &
curl -s "${BASE}&bsns_year=2025&reprt_code=11012&fs_div=CFS" -o "${OUT}/q2c.json" &
curl -s "${BASE}&bsns_year=2025&reprt_code=11014&fs_div=CFS" -o "${OUT}/q3c.json" &
curl -s "${BASE}&bsns_year=2025&reprt_code=11013&fs_div=OFS" -o "${OUT}/q1o.json" &
curl -s "${BASE}&bsns_year=2025&reprt_code=11012&fs_div=OFS" -o "${OUT}/q2o.json" &
curl -s "${BASE}&bsns_year=2025&reprt_code=11014&fs_div=OFS" -o "${OUT}/q3o.json" &

wait
echo "=== ALL DONE ==="

# 연간 상태 확인 (2025 사업보고서 존재 여부)
for f in a25c a24c a23c a22c a25o a24o a23o a22o q1c q2c q3c q1o q2o q3o; do
  STATUS=$(grep -o '"status":"[^"]*"' "${OUT}/${f}.json" | head -1)
  echo "${f}: ${STATUS}"
done
```

## Step 3: 데이터 읽기 (Bash 1~2회)

상태 확인 결과를 보고, `status:"000"`인 파일들만 cat으로 읽습니다.

**중요: 한 번에 모든 파일을 cat하지 말 것** (출력 너무 큼). 아래 전략 사용:

```bash
# CFS 연간 + OFS 연간 (핵심 데이터만 추출)
cd /tmp/dart_{CORP_CODE}
for f in a25c a24c a23c a22c a25o a24o a23o a22o; do
  echo "=== $f ==="
  # BS/IS 항목만 필터 (jq 없으면 grep으로)
  grep -oP '"sj_div":"(BS|IS|CIS)".*?"account_nm":"[^"]*".*?"thstrm_amount":"[^"]*"' "${f}.json" 2>/dev/null || cat "${f}.json"
done
```

```bash
# 분기 데이터
for f in q1c q2c q3c q1o q2o q3o; do
  echo "=== $f ==="
  grep -oP '"sj_div":"(BS|IS|CIS)".*?"account_nm":"[^"]*".*?"thstrm_amount":"[^"]*"' "${f}.json" 2>/dev/null || cat "${f}.json"
done
```

**또는 더 효율적으로**: JSON이 크면 필요한 계정과목만 grep:

```bash
for f in a25c a24c a23c a22c; do
  echo "=== $f ==="
  grep -E '"account_nm":"(유동자산|비유동자산|자산총계|유동부채|비유동부채|부채총계|자본금|이익잉여금|자본총계|매출액|수익\(매출액\)|영업수익|영업이익|법인세비용차감전|당기순이익|총포괄손익|총포괄이익)"' "${f}.json" | grep -oP '"account_nm":"[^"]*".*?"thstrm_amount":"[^"]*"'
done
```

## Step 4: 데이터 파싱 (LLM 내부 처리, tool 호출 없음)

API 응답에서 추출할 계정과목 매칭 규칙:

### 계정과목 매칭 테이블

| 항목 | 우선순위 (먼저 매칭되는 것 사용) |
|------|-------------------------------|
| 매출액 | `매출액` > `수익(매출액)` > `영업수익` > `매출` |
| 영업이익 | `영업이익` > `영업이익(손실)` |
| 법인세차감전 | `법인세비용차감전순이익` > `법인세비용차감전 순이익(손실)` > `법인세비용차감전계속사업이익` |
| 당기순이익 | `당기순이익` > `당기순이익(손실)` > `분기순이익` > `반기순이익` > `연결분기(반기)순이익` |
| 총포괄손익 | `총포괄손익` > `총포괄이익` > `분기총포괄손익` > `반기총포괄손익` |
| 이익잉여금 | `이익잉여금` > `이익잉여금(결손금)` > `결손금` |

### sj_div 매칭
- BS: `sj_div = "BS"`
- IS: `sj_div = "IS"` 또는 `sj_div = "CIS"` (포괄손익계산서)

### 값 필드
- 연간: `thstrm_amount` (당기)
- 분기: `thstrm_amount` (해당 분기 개별값). 만약 누적값이면 아래 변환 적용.

### 분기 IS 누적→개별 변환
DART 응답에는 `thstrm_amount`(당기)와 `thstrm_add_amount`(누적)가 있을 수 있음.
- `thstrm_amount`가 개별 분기값이면 그대로 사용
- 누적인지 판단: 1Q 매출 > 반기 매출이면 개별, 아니면 누적
- 누적인 경우: Q1=1분기값, Q2=반기-1분기, Q3=3분기-반기, Q4=연간-3분기
- BS는 시점 잔액이므로 항상 그대로 사용

### 기수(제N기) 추출
연간 데이터의 `thstrm_nm` 필드에서 추출 (예: "제 57 기" → "제57기")

## Step 5: HTML 생성 (Write 1회)

**003000.html을 Read하지 않는다.** 아래 템플릿을 직접 사용한다.

HTML 파일의 구조 (데이터 부분만 교체, 나머지는 고정):

```
<!DOCTYPE html> ~ <script type="text/babel">
  → 고정 (head, style, CDN imports)

const COMPANY = { ... }
  → Step 1에서 조회한 기업정보

const CFS_ANNUAL = { BS: [...], IS: [...] }
const OFS_ANNUAL = { BS: [...], IS: [...] }
const CFS_QUARTERLY = { BS: [...], IS: [...] }
const OFS_QUARTERLY = { BS: [...], IS: [...] }
  → Step 3-4에서 파싱한 재무데이터

const ANNUAL_PERIODS_BS = [...]
const ANNUAL_PERIODS_IS = [...]
const QUARTERLY_PERIODS = [...]
const ANNUAL_CHART_LABELS = [...]
const QUARTERLY_CHART_LABELS = ['1Q','2Q','3Q','4Q']
const ANNUAL_KEYS = ['y1','y2','y3','y4']
const QUARTERLY_KEYS = ['q1','q2','q3','q4']
  → 기수 레이블

// Utility functions (parseNum, toEok, formatEok, calcChange)
  → 고정

// Components (Badge, Card, TabButton, StatCard, CompanyHeader, SummaryStats, FinancialTable, ChartSection, RatioAnalysis, App)
  → 고정 (아래 전체 코드 참조)
```

### CompanyHeader에서 시장 배지
- `corp_cls === 'Y'` → `<Badge color="emerald">KOSPI</Badge>`
- `corp_cls === 'K'` → `<Badge color="amber">KOSDAQ</Badge>`

### 전체 HTML 템플릿 (데이터 삽입 위치를 __PLACEHOLDER__로 표시)

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>__COMPANY_NAME__ (__STOCK_CODE__) - 요약재무제표</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <script src="https://unpkg.com/react@18/umd/react.development.js" crossorigin></script>
  <script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js" crossorigin></script>
  <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@300;400;500;700&family=JetBrains+Mono:wght@400;500;600&display=swap');
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { font-family: 'Noto Sans KR', sans-serif; }
    .mono { font-family: 'JetBrains Mono', monospace; }
    ::-webkit-scrollbar { width: 6px; }
    ::-webkit-scrollbar-track { background: #1a1a2e; }
    ::-webkit-scrollbar-thumb { background: #444; border-radius: 3px; }
    .fade-in { animation: fadeIn 0.4s ease; }
    @keyframes fadeIn {
      from { opacity: 0; transform: translateY(8px); }
      to { opacity: 1; transform: translateY(0); }
    }
  </style>
</head>
<body class="bg-[#0f0f1a] min-h-screen text-gray-200">
  <div id="root"></div>
  <script type="text/babel">
    const { useState, useEffect, useRef, useMemo, useCallback } = React;

    function Badge({ children, color = 'indigo', className = '' }) {
      const colors = {
        indigo: 'bg-indigo-500/20 text-indigo-400 border-indigo-500/30',
        emerald: 'bg-emerald-500/20 text-emerald-400 border-emerald-500/30',
        amber: 'bg-amber-500/20 text-amber-400 border-amber-500/30',
        red: 'bg-red-500/20 text-red-400 border-red-500/30',
        gray: 'bg-gray-500/20 text-gray-400 border-gray-500/30',
      };
      return (
        <span className={`inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium border ${colors[color]} ${className}`}>
          {children}
        </span>
      );
    }

    function Card({ children, title, subtitle, className = '', headerRight }) {
      return (
        <div className={`bg-[#1a1a2e] rounded-2xl border border-[#2a2a3d] overflow-hidden fade-in ${className}`}>
          {title && (
            <div className="px-5 py-4 border-b border-[#2a2a3d] flex items-center justify-between">
              <div>
                <h3 className="text-white font-semibold text-base">{title}</h3>
                {subtitle && <p className="text-gray-500 text-xs mt-0.5">{subtitle}</p>}
              </div>
              {headerRight}
            </div>
          )}
          <div className="p-5">{children}</div>
        </div>
      );
    }

    function TabButton({ active, onClick, children }) {
      return (
        <button onClick={onClick}
          className={`px-4 py-2 rounded-lg text-sm font-medium transition-all ${
            active ? 'bg-indigo-600 text-white shadow-lg shadow-indigo-600/20'
              : 'bg-[#1a1a2e] text-gray-400 hover:text-white hover:bg-[#252540]'}`}>
          {children}
        </button>
      );
    }

    function StatCard({ label, value, change, unit = '억원' }) {
      const isNegative = value != null && parseFloat(String(value).replace(/,/g, '')) < 0;
      return (
        <div className="bg-[#12121f] rounded-xl p-4 border border-[#2a2a3d]">
          <div className="text-gray-500 text-xs mb-1">{label}</div>
          <div className={`mono text-xl font-bold ${isNegative ? 'text-red-400' : 'text-white'}`}>
            {value}<span className="text-gray-500 text-xs font-normal ml-1">{unit}</span>
          </div>
          {change !== undefined && change !== null && (
            <div className={`text-xs mt-1 ${change >= 0 ? 'text-emerald-400' : 'text-red-400'}`}>
              {change >= 0 ? '▲' : '▼'} {Math.abs(change).toFixed(1)}%
            </div>
          )}
        </div>
      );
    }

    // __DATA_SECTION__ (COMPANY, CFS_ANNUAL, OFS_ANNUAL, CFS_QUARTERLY, OFS_QUARTERLY, periods, labels, keys)

    function parseNum(str) { if (!str) return 0; return parseFloat(str.replace(/,/g, '')); }
    function toEok(str) { return Math.round(parseNum(str) / 100000000); }
    function formatEok(str) { return toEok(str).toLocaleString(); }
    function calcChange(cur, prev) {
      const c = parseNum(cur), p = parseNum(prev);
      if (p === 0) return null;
      return ((c - p) / Math.abs(p)) * 100;
    }

    function CompanyHeader() { /* ... 아래 참조 ... */ }
    function SummaryStats({ data, keys }) { /* ... */ }
    function FinancialTable({ data, type, periods, keys }) { /* ... */ }
    function ChartSection({ data, labels, keys }) { /* ... */ }
    function RatioAnalysis({ data, keys, yearLabels }) { /* ... */ }
    function App() { /* ... */ }

    ReactDOM.createRoot(document.getElementById('root')).render(<App />);
  </script>
</body>
</html>
```

**컴포넌트 코드는 dart/003000.html과 100% 동일하게 복사한다.** 변경할 부분은:
1. `<title>` 태그의 회사명/종목코드
2. `COMPANY` 객체
3. `CFS_ANNUAL`, `OFS_ANNUAL`, `CFS_QUARTERLY`, `OFS_QUARTERLY` 데이터
4. `ANNUAL_PERIODS_BS/IS`, `QUARTERLY_PERIODS` 레이블
5. `ANNUAL_CHART_LABELS`
6. `CompanyHeader` 내 시장 배지 (KOSPI/KOSDAQ)

## Step 6: 완료 보고

생성된 파일 경로, 데이터 범위, 주요 지표(최신 매출/영업이익/순이익) 요약.

## 에러 대응

- 2025 사업보고서 없음(status:"013") → 2024~2021로 4개년 조정, 분기도 해당 연도로
- OFS 데이터 없음 → CFS 데이터를 OFS에도 복사하여 표시
- 특정 계정과목 없음 → '0'으로 처리
- 분기 일부 누락 → 있는 분기만 포함

## 최적 실행 흐름 요약

| Step | Tool | 내용 | 횟수 |
|------|------|------|------|
| 1 | Bash | corpcode grep + company API | 1 |
| 2 | Bash | 15개 curl 병렬 실행 → 파일 저장 | 1 |
| 3 | Bash | 저장된 JSON에서 필요 데이터 grep/cat | 1~2 |
| 4 | (없음) | LLM이 데이터 파싱 | 0 |
| 5 | Write | HTML 파일 생성 | 1 |
| **합계** | | | **4~5** |
