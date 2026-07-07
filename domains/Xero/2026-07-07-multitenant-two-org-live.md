---
domain: Xero
date: 2026-07-07
tags: [xero, multitenant, two-org, odc-payroll, one-dream-community, tenantid, uploadbatchpayrun, rotating-refresh, automap, collapse-ui, cutover]
status: 멀티테넌트(2조직) 라이브 가동 — 직원 87/234·요율 31조합 매핑, 잔여=매칭보강 패치·회계3건·PayItem×2조직·골든패스
related: ["[[_INDEX]]", "[[2026-07-02-odc-connect-mapping-plan-2]]"]
---
# Xero 멀티테넌트(2조직) 개편 — 설계→구현→라이브 가동 (06~07.07)

회계팀 메일 한 통("조직당 200명 제한이라 A~J는 ODC Payroll, K~Z는 ODC 계정")이 **Auto-Map 41/234의 근본 원인**이었음을 실측으로 확정하고, 단일 tenant 전제 코드를 N-tenant로 개편해 당일 라이브 가동까지 완료.

## 한 일

### 1) 원인 확정 (라이브 실측)
- 배치 #5 서포터 251명 = A~J 115 / K~Z 132. EmployeeMapping GUID 매칭 41명 **전원 A~J, K~Z 0명** — 연결된 조직(ODC Payroll)에 K~Z가 아예 없던 것. 이메일 불일치(기존 추정)는 부차 원인.
- ★"ODC"는 별도 조직명이 아니라 **ONE DREAM COMMUNITY PTY LTD의 약칭**(Xero 아이콘도 ODC). 에릭 계정에 이미 접근권한 있었음 — 조직 전환 후 Pay Employees에서 First name 정렬 첫 행이 "Kate"(A~J 부재)로 확정. 별도 권한 요청 불필요했음.

### 2) 아키텍처 결정 9개 (플랜 승인)
1. 직원 tenant 소속 = `EmployeeMapping.TenantId` 진실원본 (A~J 규칙 하드코딩 금지). 유니크는 UserId 단독 유지 → 이중 급여 차단
2. `EarningsRateMapping.TenantId NOT NULL` + 유니크 `(TenantId, PayType, Area, ContractType)` 재생성
3. 배치는 tenant 불가지 — 쪼개지 않고 전송 시 행의 직원 매핑에서 tenant 유도 GroupBy (에릭 질문 "두 페이지 vs 한 페이지"에 한 페이지+조직필터로 확정)
4. PayRun은 (BatchId, TenantId)당 1개 — 신규 자식 테이블 `CARETALK_Xero_UploadBatchPayRun`이 권위, 기존 `UploadBatch.XeroPayRunId`는 유지
5. 토큰 tenant당 1행(값은 전 행 동일) + Callback에서 /connections **전체** 저장 + 목록에 없는 조직 소프트삭제. ★rotating refresh token → **single-flight(SemaphoreSlim) 갱신 1회 후 전 활성 행 upsert**
6. 기존 데이터는 ODC Payroll TenantId로 백필
7. 전송 4단계 tenant 루프(멱등 가드: PayRunId 보유 자식 행 스킵 / tenant별 행만 LOCK / 레거시 폴백 합성)
8. Auto-Map 활성 tenant 전체 순회, 복수 tenant 매칭 충돌 시 스킵
9. 배치 Status는 "참여 tenant 전부 완료" 시 전이, tenant별 진행은 자식 행 Status

### 3) 구현 (caretalk-pipeline: data→backend→frontend→qa)
- SQL 2본: `CareTalkData/20260706_xero_multitenant_schema.sql`(테이블+백필) / `_sp.sql`(신규 SP 4 + 변경 6, 신규 파라미터 전부 `=NULL` 기본값 → 배포 창 구앱 호환). 테스트DB 적용·검증 후 라이브 SSMS.
- C#: XeroAuthService(전체 connections·single-flight), XeroPayrollService(`GetTokenAndTenantAsync(tenantId)` 9곳 전파, BuildPreview tenant 키, 4단계 루프, Auto-Map 순회), XeroController(GET Tenants/BatchPayRuns 신규, `CreatePayRunRequest→{batchId, targets:[{tenantId, payrollCalendarId}]}` 파괴적 변경).
- UI: 상태카드 조직 배지, Employee 매핑 optgroup(직원 선택=조직), Rate 탭 tenant 필터, 그리드 Org 배지+**조직 필터**, Send 패널 tenant별 캘린더 행. 200행 페이징·지연 셀렉트 비접촉.
- 검증: QA PASS(E2E 5/5·경계면 10·실전송 0건 보장) → /verify 15항목(FAIL 0) → /simplify 4관점 리뷰 18건 적용(TenantStep 헬퍼 12곳, 이름 룩업 Tokens 진실원본화, 루프 내 재조회 제거, JS 로더 제네릭화·라벨 Map 캐시) → 스모크 PASS. 커밋 `1f071a7e`.

### 4) 라이브 가동 (에릭 SSMS+머지+배포 후)
- 라이브 검증: 스키마·SP 10·백필 NULL 0·새 JS 서빙 확인.
- 재연결 1회로 **2조직 연결 + Demo Company 잔존 행 자동 정리**("removed on reconnect" SyncLog) — 새 콜백 로직 첫 실행에 실증.
- Auto-Map 누적: 직원 **87/234**(ODC Payroll 45 + ONE DREAM 42), 요율 **31조합**(23+8). 모달의 "9 mapped/151 unmatched"는 실행 증가분/잔여 표기.

### 5) 운영 지식 (에릭 질문에서 정리)
- **요율 자동매핑 안 되는 이유 2가지(설계된 안전 스킵)**: ①ATTENDANT CARE는 Deputy 명세(NDIS/AGED만) 밖 → 매핑처 미정 ②같은 이름 Pay Item이 ABP7/ABP8 버전 중복 → 유일성 가드가 오급여 방지 위해 스킵. 둘 다 회계 확인 3건에 걸림.
- **Pay Period BLOCK**: Xero 타임시트는 "현재 열린 기간"(당시 06.07~19.07)에만 입력 가능. 근무 29.06~05.07은 직전 마감 기간 → 전송 불가가 정상. **컷오버 제안: ~05.07 근무=Deputy, 06.07~=CareTalk** (이중지급 방지, 회계 합의 필요).
- **Supporter Summary Preview** = 서포터별 집계(Rows/Hours/Km/HighCare행수/BrokenShift금액/PayType분해). ★Area·Employment Type은 합계 아닌 **첫 행 대표값**(XeroPayrollService BuildPreview, `first.Area`).
- **배치 #6 미매핑 165명 분류**(라이브 실측): ABN 27(매핑 불필요—자동 EXCLUDED)/Admin 11(대상 여부 확인 필요)/코드접두 5/괄호장식 13/일반 109(개인메일 vs 근무메일 불일치).

### 6) 접이식 UI + 브랜치 사고
- 검증 섹션 4종(Unmapped/Missing Rates/Period Warnings/Supporter Summary)을 건수 배지 접이식 카드(기본 접힘, 40vh 내부 스크롤, 배너 라인 클릭→펼침 점프, BLOCK 빨강 배지)로 — 에릭 "스크롤 압박" 요청.
- ★사고: 에릭이 만든 `2026.07.06-XeroMultiModify` 브랜치가 **멀티테넌트 머지 전(26.07.03) 시점 분기**라 체크아웃 시 작업 트리가 구버전으로 회귀 → 접이식 UI가 구버전 위에 구현됨. 패치 백업 후 브랜치를 VersionUpGrade로 리셋, 멀티테넌트 최신 위에 재적용(`d54aaf5a`). **교훈: 다른 세션/사용자와 작업 폴더 공유 시, 에이전트 투입 전 브랜치·핵심 코드 존재를 grep으로 확인할 것.**
- `.gitignore`에 `/xero-*.png` `/smoke-*.png` `/.claude/worktrees/` 추가(`e360ebe3`).

## 결정/맥락 (왜)
- **한 페이지+조직필터(두 페이지 아님)**: 급여 업무 단위는 조직이 아니라 기간(격주 배치). 배치에 두 조직 직원이 섞이고, 미매핑 직원은 조직을 모름(닭-달걀). 실제로 갈라지는 건 PayRun 단계뿐이라 거기만 명시 분기.
- **refresh 전파를 전 행 upsert로**: 토큰은 앱+사용자 단위로 전 조직 공용인데 행은 tenant별 — 한 행만 갱신하면 다른 행의 refresh token이 rotating으로 무효화됨.
- **SP 신규 파라미터 전부 기본값**: 스크립트를 앱 배포보다 먼저 실행해도 구앱이 정상 동작(배포 창 안전).

## 다음 할 일
- [ ] **직원 매칭 보강 패치**(괄호 `(C)/(P)/(F)`·`05C-` 접두 제거 + 공백무시 비교, 유일 매칭 가드 유지) — 에릭 승인 대기. 잔여 127명 중 상당수 자동화 기대
- [ ] 회계 확인 3건: ATTENDANT CARE 매핑처 / ContractType N 의미 / ABP7·8 현행 버전 → 요율 잔여 매핑
- [ ] **Pay Item $1.00 3종을 두 조직 각각 생성**(기존 Deputy 항목 수정 금지 — 이중곱 방지)
- [ ] Admin 11명 급여 대상 여부 확인, 회계팀에 Xero 직원 명단(이름+이메일) 요청 → 대조 일괄 매핑
- [ ] 컷오버 합의(~05.07=Deputy / 06.07~=CareTalk) 후 열린 기간 배치로 **골든패스 4단계**(수당 이중곱 대조)
- [ ] 접이식 UI(`d54aaf5a`)+`.gitignore`(`e360ebe3`) — 에릭 로컬 확인 → VersionUpGrade 머지·재배포
