---
domain: Xero
date: 2026-06-26
tags: [high-care, broken-shift, pay-item, rate-type, grid-ui, budget-claim, xero-handover]
status: 코드완료(Xero포털설정·커밋 대기)
related: ["[[_INDEX]]", "[[2026-06-25-track-c-high-care]]", "[[2026-06-26-high-care-followup-session-guard]]"]
---
# High Care 그리드 금액 표시 + Pay Item 단가($1.00) 정정 + Send to Xero 버튼 정리

## 한 일

### 1. High Care 동작 규명 (SQL 조사, 테스트DB)
- **매칭 기준 = 고객(SOI.CustomerId), 서포터 아님.** `Apply_HighCareLevels` SP가 배치행 → `ServiceOrderItems.CustomerId` → `HighCareCustomer` JOIN으로 레벨을 채움(`20260625_usp_CARETALK_XER_Apply_HighCareLevels.sql:17-31`).
- **Batch 3 사례 분석:** Ae Ran Cho 행들의 실제 고객은 Hae Kuong Lee·Eun Ye Lee. Do Heon Bae(L1 등록) 담당은 Hyung Kyu Park. HighCare가 비어있던 진짜 원인 = **시점 문제** — 배치 생성(06-25 14:23)이 고객 High Care 등록(Do Heon Bae 06-26 09:47 / Eun Ye Lee 09:54 / Hae Kuong Lee 09:58)보다 먼저. 자동채움은 **배치 생성 시 1회만** 돌아 소급 안 됨.
- **실증:** `Apply_HighCareLevels`를 Batch 3에 재실행 → **14행 자동채움 확인**(Do Heon Bae=L1, Hae Kuong Lee=L2, Eun Ye Lee=L2). ItemId 88(Ae Ran Cho/Hae Kuong Lee)=2 등.

### 2. Xero 전송 데이터 구조 확인
- 전송 = Payslip `EarningsLine { EarningsRateID, NumberOfUnits }`, `POST Payslip/{id}` (`XeroPayrollService.cs:1132-1191`).
- **HighCare·BrokenShift·KM = units 자리에 금액($)**, Sleepover만 횟수. **직원(UserId)별 1줄 합산**. HighCare 금액 = `Σ(Hours×레벨단가)` (`HighCareCalculator.cs:17`).
- 기본 근무급은 별도 Timesheet(units=시간)로 전송.

### 3. 가이드/CSV 정정 — HighCare·BrokenShift Pay Item $1.00
- 코드는 이미 **금액**을 보냄 → Xero Pay Item을 `$3/$6`·`FIXED`로 두면 **이중 곱셈**(2h L2: 앱 $12 → Xero ×6 = $72). Travel(KM)과 동일하게 **`RATEPERUNIT $1.00`** 이어야 함.
- 수정: `GUIDE_20260625_Xero_PayItem_RateType_Setup.md`(표 25·26번 Value $1.00 + §8 정정근거), `Xero_PayItems_List.csv`(25·26행 `RATEPERUNIT,1.0`).
- 결론: **Travel(24)·HighCare(25)·BrokenShift(26) 전부 RATEPERUNIT $1.00** 통일. 앱 코드/DB 변경 없음.

### 4. 그리드 HighCare 금액 병기 구현 (`xeroPayroll.js`)
- HighCare 셀에 **레벨 입력칸 유지 + `= $금액`(Hours×레벨단가) 읽기전용 병기**. 입력 중 실시간 갱신(`on input`). 단가는 `state.hcRateLevel1/2` 캐시(Settings 로드/저장 시 갱신).
- 변경: state 캐시, `hcAmount`/`hcAmountLabel` 헬퍼, `data-hours`, HighCare 셀, load/save rate, input 핸들러. `node --check` PASS, 라이브 확인 OK.
- 저장은 여전히 레벨(단가 중앙관리), 금액은 표시 전용(전송값과 동일).

### 5. BudgetClaim 버튼 UI 정리 (`/Admin/PlanManager/BudgetClaim`)
- 한글 `Xero로 넘기기` → **`Send to Xero`**(영문, UI 텍스트 규칙).
- col2 스택에서 빼내 **줄 맨 끝 새 칸으로 이동**(grid `auto 200px 275px 275px max-content` 5열). 회색 기본값 → **Xero 브랜드 블루 `#13b5ea`** 부여.
- 사유: 두 줄 스택이 아래 리스트/스크롤 간섭 → 한 줄 유지. Playwright로 한 줄 배치 검증.
- 파일: `BudgetClaim.cshtml:137`, `wwwroot/css/PlanManager/BudgetClaim.css`(grid 5열 + `#btnXeroHandover` 색).

## 결정/맥락 (왜)
- **앱 코드는 금액 전송이 정답** — 문제는 회계 가이드가 잘못된 Xero 설정을 안내한 것. 단가 1곳(앱 Settings/`HighCareRate` L1=$3/L2=$6) 관리, Xero는 $1.00로 통과.
- 그리드 "레벨 대신 금액 표시" 요구 = 전송값을 화면에서 미리 보이게. 레벨 저장 방식은 유지(단가 중앙관리 이점).
- High Care는 등록을 **배치 생성 전에** 해야 자동채움됨. 기존 배치는 (a) SP 재적용 (b) 그리드 수동 (c) 새 배치 — UI에 "재적용" 버튼 없음.

## 다음 할 일
- [ ] ⚠️ **실제 Xero 포털 Pay Item 설정 변경**(회계담당, 리포지토리 밖): "High Care Allowance"·"Broken Shift Allowance" → `RATEPERUNIT $1.00`. 안 하면 이중곱.
- [ ] 변경 커밋(에릭 승인): `xeroPayroll.js`, `BudgetClaim.cshtml`/`.css`, 가이드/CSV.
- [ ] (검토) 기존 배치 HighCare "재적용" UI 버튼 필요 여부 — 의도된 동작인지 에릭 판단.
