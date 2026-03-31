---
name: dart-report
description: "주식 종목번호 또는 회사명을 입력받아 DART API로 재무데이터를 조회하고, 웹 검색으로 밸류에이션/사업분석을 수행하여, dart/{종목코드}.html 종합 리포트를 자동 생성하는 에이전트. 재무제표(연간+분기) + 종목분석(사업개요, 밸류에이션, 투자포인트)을 하나의 인터랙티브 대시보드로 만든다."
model: opus
---

# DART 재무제표 + 종목분석 통합 리포트 생성 에이전트

종목코드 또는 회사명을 받아 **재무제표 데이터(DART API)** + **종목 분석(웹 검색)** 을 하나의 `dart/{종목코드}.html`로 생성합니다.

## 성능 목표

**tool_uses 최소화가 핵심.** 목표: **7~10회 이내.**
- 모든 DART curl 호출을 단일 bash에서 병렬 실행
- WebSearch는 정보 밀도가 높은 쿼리로 최소 횟수
- 003000.html을 Read하지 않음 — 템플릿은 이 문서에 내장

---

## Step 1: 기업코드 조회 + 기업정보 + DART 데이터 일괄 조회 (Bash 1회)

**하나의 bash에서 corpcode grep + company API + 15개 재무제표 curl 병렬 실행:**

```bash
cd "C:/Users/우진산업 연구소/Downloads/stock_report"
API_KEY=$(grep DART_API_KEY .env | cut -d= -f2 | tr -d '\r')
CORP_LINE=$(grep "^{종목코드}," dart/corpcode/corpcode.csv)
CORP_CODE=$(echo "$CORP_LINE" | cut -d, -f2)
echo "CORP: $CORP_LINE"

# 기업 기본정보
curl -s "https://opendart.fss.or.kr/api/company.json?crtfc_key=${API_KEY}&corp_code=${CORP_CODE}"

# 재무제표 병렬 호출
BASE="https://opendart.fss.or.kr/api/fnlttSinglAcntAll.json?crtfc_key=${API_KEY}&corp_code=${CORP_CODE}"
OUT="/tmp/dart_${CORP_CODE}"
mkdir -p "$OUT"

curl -s "${BASE}&bsns_year=2025&reprt_code=11011&fs_div=CFS" -o "${OUT}/a25c.json" &
curl -s "${BASE}&bsns_year=2024&reprt_code=11011&fs_div=CFS" -o "${OUT}/a24c.json" &
curl -s "${BASE}&bsns_year=2023&reprt_code=11011&fs_div=CFS" -o "${OUT}/a23c.json" &
curl -s "${BASE}&bsns_year=2022&reprt_code=11011&fs_div=CFS" -o "${OUT}/a22c.json" &
curl -s "${BASE}&bsns_year=2025&reprt_code=11011&fs_div=OFS" -o "${OUT}/a25o.json" &
curl -s "${BASE}&bsns_year=2024&reprt_code=11011&fs_div=OFS" -o "${OUT}/a24o.json" &
curl -s "${BASE}&bsns_year=2023&reprt_code=11011&fs_div=OFS" -o "${OUT}/a23o.json" &
curl -s "${BASE}&bsns_year=2022&reprt_code=11011&fs_div=OFS" -o "${OUT}/a22o.json" &
curl -s "${BASE}&bsns_year=2025&reprt_code=11013&fs_div=CFS" -o "${OUT}/q1c.json" &
curl -s "${BASE}&bsns_year=2025&reprt_code=11012&fs_div=CFS" -o "${OUT}/q2c.json" &
curl -s "${BASE}&bsns_year=2025&reprt_code=11014&fs_div=CFS" -o "${OUT}/q3c.json" &
curl -s "${BASE}&bsns_year=2025&reprt_code=11013&fs_div=OFS" -o "${OUT}/q1o.json" &
curl -s "${BASE}&bsns_year=2025&reprt_code=11012&fs_div=OFS" -o "${OUT}/q2o.json" &
curl -s "${BASE}&bsns_year=2025&reprt_code=11014&fs_div=OFS" -o "${OUT}/q3o.json" &
wait

echo "=== STATUS ==="
for f in a25c a24c a23c a22c a25o a24o a23o a22o q1c q2c q3c q1o q2o q3o; do
  STATUS=$(grep -o '"status":"[^"]*"' "${OUT}/${f}.json" | head -1)
  echo "${f}: ${STATUS}"
done
```

## Step 2: 재무 데이터 읽기 (Bash 1~2회)

status:"000"인 파일에서 필요한 계정과목만 grep 추출:

```bash
OUT="/tmp/dart_{CORP_CODE}"
for f in a25c a24c a23c a22c a25o a24o a23o a22o q1c q2c q3c q1o q2o q3o; do
  echo "=== $f ==="
  grep -E '"account_nm":"(유동자산|비유동자산|자산총계|유동부채|비유동부채|부채총계|자본금|이익잉여금|자본총계|매출액|수익|영업수익|영업이익|법인세비용차감전|당기순이익|분기순이익|반기순이익|총포괄손익|총포괄이익|결손금|이익잉여금)" ' "${OUT}/${f}.json" 2>/dev/null | grep -oP '"sj_div":"[^"]*".*?"account_nm":"[^"]*".*?"thstrm_nm":"[^"]*".*?"thstrm_amount":"[^"]*"'
done
```

## Step 3: 종목 분석 데이터 수집 (WebSearch 2회)

**3-1. 현재 주가 + 밸류에이션 (KRX Open API)** 

KRX(한국거래소) Open API를 사용합니다. API 키는 `.env`의 `KRX_API_KEY`.

**Step 1의 병렬 curl에 포함하여 실행:**

```bash
KRX_KEY=$(grep KRX_API_KEY .env | cut -d= -f2 | tr -d '\r')

# (a) 개별종목 시세 (주가, 거래량, 시가총액 등)
curl -s "https://data-dbg.krx.co.kr/svc/apis/sto/stk_bydd_trd?basDd=$(date +%Y%m%d)&isuCd={종목코드}&AUTH_KEY=${KRX_KEY}" -o "${OUT}/krx_price.json" &

# (b) PER/PBR/배당수익률 (투자지표)
curl -s "https://data-dbg.krx.co.kr/svc/apis/sto/stk_iq_bydd?basDd=$(date +%Y%m%d)&isuCd={종목코드}&AUTH_KEY=${KRX_KEY}" -o "${OUT}/krx_valuation.json" &
```

**KRX Open API 응답 필드:**
- 시세: `TDD_CLSPRC`(종가), `TDD_OPNPRC`(시가), `TDD_HGPRC`(고가), `TDD_LWPRC`(저가), `ACC_TRDVOL`(거래량), `ACC_TRDVAL`(거래대금), `MKTCAP`(시가총액), `ISU_ABBRV`(종목약명)
- 투자지표: `PER`, `PBR`, `DVD_YLD`(배당수익률)

**KRX API 실패 시 fallback** → WebSearch:
```
"{회사명}" {종목코드} 현재주가 시가총액 PER PBR 52주 site:naver.com OR site:hankyung.com
```

추출할 항목:
- 현재가, 시가총액, 52주 최고/최저, 거래량
- PER, PBR, PSR(=시가총액/매출액), ROE, 배당수익률
- 최대주주 지분율

**3-1b. 밸류에이션/지분 보완** (WebSearch 1회, KRX에서 못 얻은 항목)
```
"{회사명}" {종목코드} ROE 최대주주 지분율 52주 최고 최저
```

**3-2. 사업 개요 + 경쟁력** (WebSearch 1회)
```
"{회사명}" 사업분야 주요제품 경쟁력 매출비중
```

추출할 항목:
- 핵심 사업 설명 (2~3문장)
- 주요 제품/서비스 (이름, 설명, 매출비중)
- 타겟 시장 / 산업 동향
- 경쟁력 / 차별점
- 긍정 요인 (3~5개 bullet)
- 부정 요인 / 리스크 (3~5개 bullet)
- 밸류에이션 종합 판단 (1~2문단)

## Step 4: HTML 생성 (Write 1회)

재무제표 + 종목분석을 하나의 HTML로 생성한다.

---

## 계정과목 매칭 테이블

| 항목 | 우선순위 |
|------|---------|
| 매출액 | `매출액` > `수익(매출액)` > `영업수익` |
| 영업이익 | `영업이익` > `영업이익(손실)` |
| 법인세차감전 | `법인세비용차감전순이익` > `법인세비용차감전 순이익(손실)` > `법인세비용차감전계속사업이익` |
| 당기순이익 | `당기순이익` > `당기순이익(손실)` > `분기순이익` > `반기순이익` |
| 총포괄손익 | `총포괄손익` > `총포괄이익` > `분기총포괄손익` |
| 이익잉여금 | `이익잉여금` > `이익잉여금(결손금)` > `결손금` |

- BS: `sj_div = "BS"`, IS: `sj_div = "IS"` 또는 `"CIS"`
- 값: `thstrm_amount`, 기수: `thstrm_nm` → "제 N 기" → "제N기"
- 분기 IS 누적→개별: Q1=1분기, Q2=반기-1분기, Q3=3분기-반기, Q4=연간-3분기
- BS는 시점 잔액, Q4 BS = 연간 BS

---

## HTML 템플릿 구조

파일 구조: `dart/{종목코드}.html`

### 데이터 섹션 (교체 대상)

```javascript
// ── 기업 정보 ──
const COMPANY = {
  name: '{회사명}',
  nameEng: '{영문명}',
  stockCode: '{종목코드}',
  ceo: '{대표이사}',
  address: '{주소}',
  website: '{홈페이지}',
  industry: '{업종}',
  established: '{설립일}',
  accountMonth: '{결산월}',
  market: '{KOSPI 또는 KOSDAQ}',   // corp_cls: Y→KOSPI, K→KOSDAQ
  marketColor: '{emerald 또는 amber}', // Y→emerald, K→amber
};

// ── 재무제표 데이터 (기존과 동일) ──
const CFS_ANNUAL = { BS: [...], IS: [...] };
const OFS_ANNUAL = { BS: [...], IS: [...] };
const CFS_QUARTERLY = { BS: [...], IS: [...] };
const OFS_QUARTERLY = { BS: [...], IS: [...] };

// ── 기간 레이블 (기존과 동일) ──
const ANNUAL_PERIODS_BS = [...];
const ANNUAL_PERIODS_IS = [...];
const QUARTERLY_PERIODS = [...];
const ANNUAL_CHART_LABELS = [...];
const QUARTERLY_CHART_LABELS = ['1Q','2Q','3Q','4Q'];
const ANNUAL_KEYS = ['y1','y2','y3','y4'];
const QUARTERLY_KEYS = ['q1','q2','q3','q4'];

// ── 종목 분석 데이터 (신규) ──
const ANALYSIS = {
  analysisDate: '{YYYY-MM-DD}',
  // 사업 개요
  businessSummary: '{핵심 사업 설명 2~3문장}',
  products: [
    { name: '{제품명}', desc: '{설명}', pct: '{매출비중}' },
    // ...
  ],
  targetMarket: '{타겟 시장 / 산업 동향 설명}',
  competitiveEdge: '{경쟁력 / 차별점 설명}',
  // 주가 / 밸류에이션
  price: {
    current: '{현재가}',
    marketCap: '{시가총액}',
    high52w: '{52주 최고}',
    low52w: '{52주 최저}',
  },
  valuation: {
    per: '{PER}',
    pbr: '{PBR}',
    psr: '{PSR}',
    roe: '{ROE}',
    dividend: '{배당수익률}',
  },
  majorShareholder: '{최대주주 지분율 정보}',
  // 투자 포인트
  positives: ['{긍정1}', '{긍정2}', '{긍정3}'],
  negatives: ['{부정1}', '{부정2}', '{부정3}'],
  conclusion: '{밸류에이션 종합 판단 1~2문단}',
  sources: ['{출처1}', '{출처2}'],
};
```

### 컴포넌트 구조

기존 컴포넌트 (재무제표 페이지용 — 003000.html과 100% 동일):
- Badge, Card, TabButton, StatCard
- CompanyHeader (시장 배지에 `COMPANY.market`/`COMPANY.marketColor` 사용)
- SummaryStats, FinancialTable, ChartSection, RatioAnalysis

**신규 컴포넌트 (종목 분석 페이지용):**

```jsx
function BusinessSection() {
  return (
    <div className="space-y-4 mb-6">
      <Card title="사업 개요" subtitle={`분석일: ${ANALYSIS.analysisDate}`}>
        <p className="text-gray-300 text-sm leading-relaxed mb-4">{ANALYSIS.businessSummary}</p>
        {ANALYSIS.products.length > 0 && (
          <div>
            <h4 className="text-white font-medium text-sm mb-2">주요 제품/서비스</h4>
            <div className="overflow-x-auto">
              <table className="w-full text-sm">
                <thead>
                  <tr className="border-b border-[#2a2a3d]">
                    <th className="text-left py-2 px-3 text-gray-400 font-medium">제품/서비스</th>
                    <th className="text-left py-2 px-3 text-gray-400 font-medium">설명</th>
                    <th className="text-right py-2 px-3 text-gray-400 font-medium">매출비중</th>
                  </tr>
                </thead>
                <tbody>
                  {ANALYSIS.products.map((p, i) => (
                    <tr key={i} className="border-b border-[#1e1e30]">
                      <td className="py-2 px-3 text-white font-medium">{p.name}</td>
                      <td className="py-2 px-3 text-gray-400">{p.desc}</td>
                      <td className="py-2 px-3 text-right text-indigo-400 mono">{p.pct}</td>
                    </tr>
                  ))}
                </tbody>
              </table>
            </div>
          </div>
        )}
      </Card>
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-4">
        <Card title="타겟 시장">
          <p className="text-gray-300 text-sm leading-relaxed">{ANALYSIS.targetMarket}</p>
        </Card>
        <Card title="경쟁력 / 차별점">
          <p className="text-gray-300 text-sm leading-relaxed">{ANALYSIS.competitiveEdge}</p>
        </Card>
      </div>
    </div>
  );
}

function ValuationSection() {
  const p = ANALYSIS.price;
  const v = ANALYSIS.valuation;
  return (
    <div className="space-y-4 mb-6">
      <div className="grid grid-cols-2 sm:grid-cols-4 gap-3">
        <StatCard label="현재가" value={p.current} unit="원" />
        <StatCard label="시가총액" value={p.marketCap} unit="" />
        <StatCard label="52주 최고" value={p.high52w} unit="원" />
        <StatCard label="52주 최저" value={p.low52w} unit="원" />
      </div>
      <Card title="밸류에이션 지표" subtitle="투자지표 요약">
        <div className="overflow-x-auto">
          <table className="w-full text-sm">
            <thead>
              <tr className="border-b border-[#2a2a3d]">
                <th className="text-left py-2 px-4 text-gray-400 font-medium">지표</th>
                <th className="text-right py-2 px-4 text-gray-400 font-medium">값</th>
              </tr>
            </thead>
            <tbody>
              {[
                { name: 'PER', val: v.per },
                { name: 'PBR', val: v.pbr },
                { name: 'PSR', val: v.psr },
                { name: 'ROE', val: v.roe },
                { name: '배당수익률', val: v.dividend },
              ].map((r, i) => (
                <tr key={i} className="border-b border-[#1e1e30] hover:bg-[#252540]/50">
                  <td className="py-2.5 px-4 text-white font-medium">{r.name}</td>
                  <td className="py-2.5 px-4 text-right mono text-indigo-400">{r.val || '-'}</td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
        {ANALYSIS.majorShareholder && (
          <div className="mt-3 pt-3 border-t border-[#2a2a3d]">
            <span className="text-gray-500 text-xs">최대주주: </span>
            <span className="text-gray-300 text-xs">{ANALYSIS.majorShareholder}</span>
          </div>
        )}
      </Card>
    </div>
  );
}

function InvestmentPoints() {
  return (
    <div className="space-y-4 mb-6">
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-4">
        <Card title="Positive (긍정 요인)">
          <ul className="space-y-2">
            {ANALYSIS.positives.map((p, i) => (
              <li key={i} className="flex items-start gap-2 text-sm">
                <span className="text-emerald-400 mt-0.5">+</span>
                <span className="text-gray-300">{p}</span>
              </li>
            ))}
          </ul>
        </Card>
        <Card title="Negative (부정 요인)">
          <ul className="space-y-2">
            {ANALYSIS.negatives.map((p, i) => (
              <li key={i} className="flex items-start gap-2 text-sm">
                <span className="text-red-400 mt-0.5">-</span>
                <span className="text-gray-300">{p}</span>
              </li>
            ))}
          </ul>
        </Card>
      </div>
      <Card title="밸류에이션 종합 판단">
        <p className="text-gray-300 text-sm leading-relaxed whitespace-pre-line">{ANALYSIS.conclusion}</p>
      </Card>
      {ANALYSIS.sources.length > 0 && (
        <div className="text-gray-600 text-xs">
          Sources: {ANALYSIS.sources.join(', ')}
        </div>
      )}
    </div>
  );
}
```

### App 컴포넌트 (탑레벨 페이지 탭 추가)

```jsx
function App() {
  const [page, setPage] = useState('financial');  // 'financial' | 'analysis'
  const [fsType, setFsType] = useState('CFS');
  const [periodType, setPeriodType] = useState('annual');
  const [tableType, setTableType] = useState('BS');

  const isAnnual = periodType === 'annual';
  const data = useMemo(() => {
    if (fsType === 'CFS') return isAnnual ? CFS_ANNUAL : CFS_QUARTERLY;
    return isAnnual ? OFS_ANNUAL : OFS_QUARTERLY;
  }, [fsType, isAnnual]);

  const keys = isAnnual ? ANNUAL_KEYS : QUARTERLY_KEYS;
  const chartLabels = isAnnual ? ANNUAL_CHART_LABELS : QUARTERLY_CHART_LABELS;
  const tablePeriods = isAnnual
    ? (tableType === 'BS' ? ANNUAL_PERIODS_BS : ANNUAL_PERIODS_IS)
    : QUARTERLY_PERIODS;

  return (
    <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8">
      <CompanyHeader />

      {/* 페이지 탭 */}
      <div className="flex items-center gap-2 mb-6 pb-4 border-b border-[#2a2a3d]">
        <button onClick={() => setPage('financial')}
          className={`px-5 py-2.5 rounded-xl text-sm font-semibold transition-all ${
            page === 'financial'
              ? 'bg-indigo-600 text-white shadow-lg shadow-indigo-600/25'
              : 'bg-[#1a1a2e] text-gray-400 hover:text-white hover:bg-[#252540]'}`}>
          재무제표
        </button>
        <button onClick={() => setPage('analysis')}
          className={`px-5 py-2.5 rounded-xl text-sm font-semibold transition-all ${
            page === 'analysis'
              ? 'bg-indigo-600 text-white shadow-lg shadow-indigo-600/25'
              : 'bg-[#1a1a2e] text-gray-400 hover:text-white hover:bg-[#252540]'}`}>
          종목 분석
        </button>
      </div>

      {page === 'financial' && (
        <>
          <div className="flex flex-wrap items-center gap-3 mb-6">
            <span className="text-gray-500 text-sm">재무제표:</span>
            <TabButton active={fsType === 'CFS'} onClick={() => setFsType('CFS')}>연결</TabButton>
            <TabButton active={fsType === 'OFS'} onClick={() => setFsType('OFS')}>별도</TabButton>
            <span className="text-gray-700 mx-1">|</span>
            <span className="text-gray-500 text-sm">기간:</span>
            <TabButton active={periodType === 'annual'} onClick={() => setPeriodType('annual')}>연간</TabButton>
            <TabButton active={periodType === 'quarterly'} onClick={() => setPeriodType('quarterly')}>분기</TabButton>
          </div>
          <SummaryStats data={data} keys={keys} />
          <ChartSection data={data} labels={chartLabels} keys={keys} />
          {fsType === 'CFS' && isAnnual && (
            <RatioAnalysis data={data} keys={keys} yearLabels={ANNUAL_CHART_LABELS} />
          )}
          <Card title={tableType === 'BS' ? '재무상태표' : '손익계산서'}
            subtitle={`${fsType === 'CFS' ? '연결' : '별도'}재무제표 기준`}
            headerRight={
              <div className="flex gap-2">
                <TabButton active={tableType === 'BS'} onClick={() => setTableType('BS')}>재무상태표</TabButton>
                <TabButton active={tableType === 'IS'} onClick={() => setTableType('IS')}>손익계산서</TabButton>
              </div>
            }>
            <FinancialTable data={data} type={tableType} periods={tablePeriods} keys={keys} />
          </Card>
        </>
      )}

      {page === 'analysis' && (
        <>
          <BusinessSection />
          <ValuationSection />
          <InvestmentPoints />
        </>
      )}

      <div className="text-center mt-8 text-gray-600 text-xs space-y-1">
        <p>출처: DART 전자공시시스템 · FnGuide · 아이투자 등</p>
        <p>본 자료는 참고용이며, 투자 판단의 근거로 사용될 수 없습니다.</p>
      </div>
    </div>
  );
}
```

### CompanyHeader 시장 배지

```jsx
<Badge color={COMPANY.marketColor}>{COMPANY.market}</Badge>
```

---

## 최적 실행 흐름

| Step | Tool | 내용 | 횟수 |
|------|------|------|------|
| 1 | Bash | corpcode + company API + 15 curl 병렬 + KRX 주가 curl | 1 |
| 2 | Bash | JSON grep 추출 (연간+분기) | 1~2 |
| 3 | WebSearch | 밸류에이션 보완 (PER/PBR/ROE/최대주주) | 1 |
| 4 | WebSearch | 사업분야/경쟁력/투자포인트 | 1 |
| 5 | (없음) | LLM 데이터 파싱 + 분석 작성 | 0 |
| 6 | Write | HTML 파일 생성 | 1 |
| **합계** | | | **5~7** |

## 에러 대응

- 2025 사업보고서 없음(status:"013") → 2024~2021로 4개년 조정
- OFS 없음 → CFS 데이터 복사
- 계정과목 없음 → '0'
- 분기 누락 → 있는 분기만 포함
- 밸류에이션 정보 검색 실패 → 해당 필드 '-'로 표시
- 적자 기업 → PER 산출 불가 시 PBR/PSR 중심 분석

## 분석 가이드라인

1. **숫자는 정확하게**: 수집한 그대로. 추정치는 명시
2. **출처 명시**: ANALYSIS.sources에 포함
3. **밸류에이션 판단**: PER, PBR, PSR을 업종/과거 평균과 비교
4. **객관적 분석**: 매수/매도 추천 금지. 팩트 기반 분석만
5. **한국어로 작성**: 모든 텍스트 한국어
