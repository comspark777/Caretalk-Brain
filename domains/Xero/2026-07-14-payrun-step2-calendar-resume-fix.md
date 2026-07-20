---
domain: Xero
date: 2026-07-14
tags: [xero, payrun, step2, payroll-calendar, resume, timesheet, token-expiry, golden-path, calendar-select]
status: 골든패스 Step2 진입 차단 버그 2건 수정·실서버 배포 + 캘린더 셀렉트 식별성 개선(기간·인원·자동선택). 15.07 회계팀 전체 데이터 락 테스트 예정
related: ["[[_INDEX]]", "[[2026-07-07-multitenant-two-org-live]]", "[[2026-07-07-admin-manual-validation-diagnosis]]"]
---
# PayRun Step2 차단 버그 2건 수정 + 캘린더 셀렉트 개선 (07.14)

회계팀 첫 골든패스 실전 피드백(배치 #14, 06.07~19.07, 전 행 ODC Payroll)에서 **Step 2 진입이 구조적으로 불가능**했던 버그 2건을 확정·수정·실서버 배포. 멀티테넌트 개편 이후 골든패스가 한 번도 Step 2를 통과한 적 없던 이유였음(#11·#13 전부 TIMESHEET_SENT 정지로 방증). Step 1(타임시트→Xero approved)은 회계팀 실행으로 첫 실증됨.

## 한 일

### 0) 연결 카드 "(expired)" 문의 — 정상 동작 확인 (버그 아님)
- 상태 카드는 순수 DB 조회(`XeroAuthService.cs:202` GetConnectionStatusAsync, IsExpired = ExpiresAtUtc≤now). **Refresh Status 버튼도 토큰 갱신 안 함**(재조회만).
- access token 수명 30분, 갱신은 실제 API 작업 때 on-demand(`GetValidAccessTokenAsync` single-flight). 30분+ 무활동이면 (expired) 표시가 정상. 진짜 문제는 refresh token 사망(60일 미사용/revoke) 때뿐 — 작업 시 "re-authentication required" 에러로 구분, 그때만 Reconnect(1회로 2조직 갱신). 두 조직 동시 만료는 토큰이 앱+사용자 공용이라 정상.

### 1) 버그① Step2 "Please select a Payroll Calendar" — 뭘 골라도 100% 실패
- 원인: Step 1 성공 직후 `renderTenantCalendarRows()` 재렌더(`xeroPayroll.js:1558`)가 empty() 후 **자식 행 저장값만** 복원하는데, 자식 행 CalendarId는 PAYRUN_CREATED 후에야 저장(`XeroPayrollService.cs:1043-1045` null 보존) → Step 2 직전엔 항상 빈 값 → missing 판정 → 노티스. 닭-달걀 구조라 사용자 선택과 무관하게 첫 실행 반드시 실패.
- 수정: empty() 전 현재 화면 선택값(prevSel) 캡처 → 자식 행 값 null이면 폴백 복원(서버 저장값 존재 시 그쪽 우선).

### 2) 버그② 재전송 시 Step 1 Failed — 매뉴얼 "완료 단계 스킵" 미구현
- 원인: `SubmitTimesheetsAsync`가 무조건 preview 재검증 → 전 행 SENT면 active 0 → "no rows available" 차단(`XeroPayrollService.cs:435-436`→916 Message) → JS 에러 경로로 Step 1 Failed. **"전부 이미 전송"(재개 OK)과 "전부 제외"(진짜 차단)를 같은 사유로 묶어 차단**한 게 핵심.
- 수정: 차단 사유가 구조적으로 no-rows뿐(unmapped 0·missing rate 0·period BLOCK 0)이고 BatchPayRun 자식 행이 존재하면 → "Timesheets already sent - step skipped" 성공 응답으로 통과 → Step 2부터 재개. 전 행 EXCLUDED(자식 행 없음)는 기존대로 차단 — 오탐 없음. 판정을 문자열 매칭이 아닌 구조(검증 카운트+자식 행)로 해 문구 변경에 안전.
- 커밋 `39ca38e2` (2026.07.13-QuickClaim 브랜치).

### 3) 캘린더 셀렉트 식별성 개선 (커밋 `8ce2d44a`)
- 문제: ODC Payroll 조직에 동명 "Fortnightly (FORTNIGHTLY)" 캘린더 5개 → 회계팀이 어느 항목인지 알 수 없음.
- 서버: `GetPayrollCalendarsAsync(tenantId, batchId=0)` 확장 — ①열린 기간 계산(프리뷰/전송 검증과 동일 로직·DateTime.Today, 표기와 차단 기준 불일치 제거) ②batchId 지정 시 배치 직원(EXCLUDED 제외, distinct XeroEmployeeId)×직원 PayrollCalendarID 대조 → 캘린더별 인원(BatchEmployeeCount). 인원 계산 실패해도 목록은 반환(try-catch). Ref VM에 OpenPeriodStart/End·BatchEmployeeCount 3필드 추가.
- JS: 라벨 `Fortnightly (FORTNIGHTLY, 06.07–19.07) — 3 in this batch` / **소속 캘린더가 정확히 1개면 자동 선택**(2개+면 인원만 표시, 운영자 판단) / 캘린더 캐시 openBatch마다 초기화(인원이 배치별로 다름) / ensureTenantCalendars가 batchId 동봉.
- 정답 원칙 문서화: 캘린더는 고르는 옵션이 아니라 **직원의 Xero 배정 소속**. 잘못 고르면 DRAFT PayRun에 payslip 없음 → 3단계 실패(DRAFT 삭제로 복구 가능).

### 4) 검증·배포·커뮤니케이션
- CareTalk.Web 전체 체인 빌드 오류 0 + node --check OK. camelCase(openPeriodStart/batchEmployeeCount)·payrollCalendarID 끝대문자 정합 확인. 런타임 E2E는 미실행(Step 2+는 실제 Xero PayRun 생성이라 자동화 부적합) — 배포 후 첫 실배치가 최종 검증.
- 에릭 실서버 배포 완료. 회계팀 답장 초안 작성(새로고침 후 #14 그대로 재실행 → 1단계 스킵→2단계 진행 / **Posting 확정 전 DRAFT 수당 금액 확인** 당부 — Pay Item $1.00 정정 완료 전이면 Travel/HighCare/BrokenShift 이중곱 위험).

## 결정/맥락 (왜)
- 재개 판정을 구조 기반으로(문자열 X): BlockReasons 문구가 바뀌어도 안전하고, "전부 제외" 배치가 오통과할 수 없음.
- 자동 선택은 1개일 때만: 배치 직원이 2개+ 캘린더에 걸치면 단일 PayRun으로 커버 불가한 비정상 신호 — 자동으로 덮지 않고 노출.
- 인원 계산을 서버에 둠: 직원→캘린더 매핑(Xero API)이 JS에 없고, 기간 계산을 전송 검증과 같은 코드 경로에 묶어야 표기≠차단 불일치가 안 생김.

## 운영/기술 팁 (재사용)
- 연결 카드 "(expired)"는 대부분 무해(유휴 만료). 실제 작업 성공이 곧 토큰 갱신 겸 확인. Reconnect는 작업이 "re-authentication required"로 실패할 때만.
- ODC Payroll 조직에 동명 Fortnightly 5개 + "Weekly (FORTNIGHTLY)"(이름·타입 불일치) 존재 — Xero 측 캘린더 정리 대상 후보.

## 다음 할 일
- [ ] **15.07 회계팀 전체 데이터 락 테스트** = 골든패스 Step 2~4 첫 통과 확인(#14 재실행 포함). 4단계 Posting 전 DRAFT 수당 대조(Pay Item $1.00×2조직 완료 여부와 연동)
- [ ] 커밋 2건(`39ca38e2`·`8ce2d44a`)이 `2026.07.13-QuickClaim` 브랜치에 있음 — 머지 정리 필요(에릭 승인)
- [ ] (승계) 매칭보강 패치 승인 대기·회계 확인 3건·컷오버 합의·접이식UI 머지
