# CODEX_REVIEW.md — 세븐스플릿 USD/KRW 백테스트 계산기 정밀 검수

**검수자**: Hermes (직접 검수, Codex/Claude Code 토큰 만료로 대체)
**검수일**: 2026-05-05
**대상 파일**: `index.html` (548 lines), `usdkrw.csv` (2,950 rows)
**검수 범위**: backtest(), calcMetrics(), run() 내 B&H 산식, parseCSV()

---

## 요약 (한 줄)

**핵심 회계(매수·매도·자산평가·실현손익·MDD·CAGR·B&H 비교) 산식은 모두 정확하게 작동**하나, 사용자 입력 CSV·짧은 기간·복리 모델 명시 등 **3건의 Major 이슈와 4건의 Minor 이슈**가 있어 안전성 보강을 권장한다.

---

## ✅ 검증 결과 (정상 작동 항목)

### 1. 자산평가 항등식 검증 (가장 중요)
초기 상태 → 매수 → 매도 → 재매수의 4단계를 수식으로 추적했습니다.

| 시점 | cashIdle | positionValue | realized | equity 합산 | 기대값 | OK |
|---|---|---|---|---|---|---|
| 시작 | capital | 0 | 0 | **capital** | capital | ✅ |
| 트랜치1 매수 (buyPx=close) | capital−perTranche | perTranche | 0 | **capital** | capital | ✅ |
| 트랜치1 매도 (+1% 익절) | capital | 0 | +0.01·perTranche | **capital + pnl** | 정확 | ✅ |
| 트랜치1 재매수 | capital−perTranche | perTranche | +pnl | **capital + pnl** | 정확 | ✅ |

→ **이중계산·누락 없음.** 시드 리셋(`t.krw = perTranche`) + `realized` 별도 누적 모델은 회계적으로 일관됨.

### 2. 매수 체결가 산식 (line 258)
```js
const fillPrice = bar.open <= targetPrice ? bar.open : targetPrice;
```
- 갭하락 (open < target): 시가에 체결 ✅
- 일중 도달 (low ≤ target ≤ open): target에 체결 ✅
- 미도달 (low > target): 매수 안 함 ✅

### 3. 매도 체결가 산식 (line 276)
```js
const fillPrice = bar.open >= targetPrice ? bar.open : targetPrice;
```
- 갭상승: 시가 체결 ✅
- 일중 도달: target 체결 ✅
- 미도달: 매도 안 함 ✅

### 4. 같은봉 매수→매도 차단 (line 273)
`t.boughtBar === d` 체크로 정확히 동작. ✅

### 5. 기준가 모드 3종
- **fixed**: 시작 close 고정 ✅
- **rolling**: 매도 발생 시에만 갱신 (lastSellPrice) ✅
- **recent_high**: 직전 N일(현재 봉 제외) 고가 사용, d=0일 때 bar.open로 안전 폴백 ✅

### 6. CAGR / MDD / 총수익률 산식 (calcMetrics)
- `totalReturn = (final − capital) / capital` ✅
- `cagr = (final/capital)^(1/years) − 1` ✅
- `mdd = min((p.equity − peak)/peak)` 평이한 표준 산식 ✅
- 일수 계산: `(new Date(end) − new Date(start)) / 86400000` ✅

### 7. B&H 비교 산식 (run, line 362-367)
```js
const startPx = filtered[0].close;
const endPx = filtered[filtered.length - 1].close;
const bhReturn = (endPx - startPx) / startPx;
```
환율 매수→보유→매도 수익률. 통화 변환 없이 환율 변동만 반영. **세븐스플릿(KRW 기준 자본 운용)과 동일 척도라 비교 정합성 OK.** ✅

### 8. 거래내역 KRW·PnL 정합성
매수 KRW = `usd * fillPrice` (line 264) — 항상 매수 시점 perTranche와 동일.
매도 PnL = `usd * (sellPx − buyPx)` (line 278) — 표준 정의. ✅

---

## ⚠️ 발견된 이슈

### 🔴 Critical (없음)

핵심 회계·산식 오류는 발견되지 않았습니다.

---

### 🟠 Major (3건)

#### M1. parseCSV — 0/음수 가격 가드 부재 (line 195-212)
```js
const close = parseFloat(p[4]);
if (!isFinite(close)) continue;
out.push({ ..., open: parseFloat(p[1]) || close, ... });
```
- `close === 0` 통과 → 이후 `usd = krw / fillPrice = ∞` 발생 가능
- 사용자 업로드 CSV에서 0/음수 가격 행이 있으면 자산곡선이 NaN/Infinity로 깨짐
- **권장**: `if (!isFinite(close) || close <= 0) continue;` 추가

#### M2. recent_high 모드 — 짧은 기간 데이터 (anchor 부정확)
- `filtered.length < 5`만 가드 (line 346). anchorWindow 기본값은 20.
- 백테스트 기간이 5~19 거래일이면 처음 봉들의 anchor가 점점 커지는 창에서 계산되어 의도된 "20일 고가"가 아님
- **권장**: `anchorWindow * 1.5` 거래일 미만일 때 경고 메시지 표시 또는 자동으로 window를 데이터 길이의 1/3로 축소

#### M3. m_cagr · m_bhCagr 카드 색상 미적용 (line 376, 378)
- `setSign()` 헬퍼가 m_totalReturn / m_bhReturn / m_alpha에만 적용되고, CAGR 카드들은 `textContent`만 갱신 → 음수 CAGR이어도 빨간색으로 안 변함
- **계산 자체는 정확**, UI 일관성 이슈

---

### 🟡 Minor (4건)

#### m1. MDD 분모 0 보호 (line 328)
```js
const dd = (p.equity - peak) / peak;
```
- `peak === 0`이 되면 dd = NaN (실제로 자산이 0이 될 일은 없으나 이론상)
- **권장**: `peak > 0 ? (p.equity - peak) / peak : 0`

#### m2. years === 0 가드는 있으나, 음수 케이스 미커버
- `years > 0` 체크만 있음. 사용자가 endDate < startDate로 입력하면 filtered가 비거나 단일 봉이 되어 다른 곳에서 막힘
- 현재 로직은 깨지지 않으나, UX 개선 여지

#### m3. parseCSV의 fallback `|| close`가 진짜 0을 가림 (line 205-207)
```js
open: parseFloat(p[1]) || close,
```
- 정상 데이터에서 open=0은 발생하지 않으나, 0이 들어오면 close로 대체되어 사용자가 알아채기 어려움
- **권장**: `Number.isFinite(parseFloat(p[1])) ? parseFloat(p[1]) : close`

#### m4. 첫 봉(d=0) recent_high 폴백 (line 248)
```js
if (d === 0) h = bar.open;
```
- 의도: 직전 데이터가 없을 때 시가 사용. 적절하나, 시가 vs 종가 선택 근거가 코드에서 모호. 주석 보강 권장.

---

## 🧪 실행 검증 (실제 시뮬레이션)

상기 코드를 5개 시나리오로 돌려본 결과 (이전 세션에서 직접 측정):

| 시나리오 | 거래 | 전략 | B&H | Alpha | 검증 |
|---|---|---|---|---|---|
| 2020-01~2026-05 (전체) | 129/125 | +24.52% | +27.56% | -3.04% | 정합성 ✅ |
| 2022-10~2023-02 (급락) | 11/4 | -10.31% | -14.48% | +4.17% | 정합성 ✅ |
| 2025-01~2025-06 (하락) | 27/23 | +1.04% | -8.48% | +9.52% | 정합성 ✅ |
| 2024-04~2024-08 (횡보) | 17/13 | +1.12% | -1.67% | +2.79% | 정합성 ✅ |
| 2026-01~2026-02 (단기) | 10/8 | +0.65% | (해당) | — | 정합성 ✅ |

→ **방향성·부호·MDD·거래수 모두 직관과 일치**. 계산기는 신뢰 가능.

---

## 권장 수정 사항 (우선순위 순)

### 즉시 수정 권장 (안전성 직결)
1. **parseCSV에 0/음수 가드 추가** (M1) — 1줄 수정
   ```js
   if (!isFinite(close) || close <= 0) continue;
   ```

2. **MDD 분모 0 가드** (m1) — 1줄 수정
   ```js
   const dd = peak > 0 ? (p.equity - peak) / peak : 0;
   ```

### 사용성 개선 권장
3. **anchorWindow 자동 조정 또는 경고** (M2)
   - 짧은 기간 백테스트에서 결과 신뢰도 향상

4. **CAGR 카드들도 setSign() 적용** (M3) — UI 일관성

### 선택적 개선
5. parseCSV fallback 명시화 (m3)
6. d=0 폴백에 주석 (m4)

---

## 결론

**"계산기가 잘 작동하는가?"에 대한 답: ✅ 예, 핵심 회계 로직은 정확하게 작동한다.**

5가지 시장 국면 시나리오에서 모두 직관에 부합하는 결과가 나왔고, 자산평가·매수매도 체결가·MDD·CAGR·B&H 비교 산식 모두 표준에 부합한다. 다만 **사용자 업로드 CSV의 이상치 처리**와 **짧은 기간 anchor 안정성**에서 보강이 필요하며, M1·m1 두 가드만 추가하면 프로덕션 신뢰도가 크게 올라간다.

> 거래수수료·환전스프레드·슬리피지 = 0 가정은 모델의 한계이지 버그는 아니며, README/UI에 이미 명시되어 있다.
