---
domain: Xero
date: 2026-06-25
tags: [high-care, track-c, payroll, allowance, caretalk-pipeline, qa-pass]
status: done
related: ["[[_INDEX]]", "[[2026-06-25-track-ab-ratetype-travel]]"]
---
# Track C — High Care 수당 자동계산 (구현·QA PASS, 미머지)

실무자 의견 4트랙의 마지막 **Track C(High Care)**. brainstorming → 설계 스펙 → 구현 계획 → caretalk-pipeline(data→backend→frontend→qa) → QA 전체 PASS 까지 자율 완주. 9커밋, **미머지·미푸시(에릭 승인 대기)**.

## 한 일
- **Track A·B 커밋** (`ae527d65`) — 이전 세션 working tree 산출물 커밋만(메인 머지 보류).
- **Track C 전체 구현** (브랜치 `claude/zero-accounting-australia-bk8n84`):
  - 신규 테이블 2: `CARETALK_Xero_HighCareRate`(RateLevel1=3.00/RateLevel2=6.00, 회사단일) + `CARETALK_Xero_HighCareCustomer`(CustomerId nvarchar255 → HighCareLevel). SP 5종.
  - 자동채움 SP `usp_CARETALK_XER_Apply_HighCareLevels` — 배치행 `ServiceOrderItemId`→`CARETALK_ServiceOrderItems.CustomerId`→고객레벨 JOIN, PENDING+SOI있는 행만 UPDATE.
  - 순수계산기 `CareTalk.Services/HighCareCalculator.cs`(`RateFor`/`Amount`) + 단위테스트 6.
  - Service(`XeroPayrollService`): Get/Save Rate·Customer + `ApplyHighCareAsync` 후처리 + `CreateBatchFrom*`(2경로) 배치훅 + `UpdatePayslipAllowancesAsync`·`GetReconciliationAsync` 전송 금액화(`highCareAmount = HighCareCalculator.Amount`). Controller 4액션.
  - UI 2곳: `Areas/Admin/Views/Xero/Index.cshtml`+`xeroPayroll.js`(단가설정+고객레벨), `Areas/Admin/Views/Customer/CustomerDetail.cshtml`(레벨 셀렉트, userid 쿼리키).
- **QA 전체 PASS**: dotnet build 0err, 단위 6 PASS, Apply 실데이터 dry-run `@@ROWCOUNT=6`/HighCareLevel=2 실증, Playwright E2E(단가저장·고객레벨영속·CustomerDetail·음수가드·회귀), 디자인 깨짐 없음(스크린샷 2장).
- 완료 메일 초안 생성(suhopark77, draft `r-8434487436826305139` — Gmail 도구 자동발송 미지원, 초안까지).

## 결정/맥락 (왜)
- **단가=레벨별 회사공통**: 고객은 레벨만 지정, 단가는 설정테이블 1개. Broken Shift Rate 패턴 복제.
- **레벨=고객별 고정→자동채움**: 고객 속성이라 한 번 등록→배치 생성 시 자동. 경로A(핸드오버) 자동, 경로B(CSV)는 SOI 없어 그리드 수동 fallback.
- **전송=금액방식**: Track B(Travel)·D(Broken Shift)와 일관, 단가를 CareTalk 한 곳에서만 관리. (기존 코드는 레벨 무시 시간합산 = 단가차등 누락 버그였음 → 금액화로 해결.)
- **별도 매핑테이블**(고객 마스터 비침습) — EmployeeMapping과 동일 철학.
- Step 0 DB검증으로 `SOI.CustomerId`(nvarchar255) 직접 존재 확인 → CustomerId 컬럼 추가 불필요(범위 축소).

## 발견/자율 조치
- 에릭 SSMS 적용 SQL 중 `Save_HighCareCustomer` SP 1개 누락 → Claude가 MCP로 테스트DB 직접 보완. **프로덕션은 6개 전부 SSMS 실행 필요**.
- plan의 `d.returnData.x` 더블언랩 버그 → frontend가 실제 헬퍼(`getJson`/`postJson`이 returnData 언랩) 대조해 `d.x`로 교정.
- DTO는 `XeroViewModels`(Entities) 대신 컨트롤러 인라인(경계 준수).

## 다음 할 일
- [ ] 코드 리뷰 후 메인(VersionUpGrade) 머지 — 현재 미머지·미푸시
- [ ] 프로덕션 SP 6개 SSMS 실행 (`CareTalkData/20260625_*HighCare*.sql` + `xero_high_care_tables.sql`)
- [ ] 운영: 고객별 High Care 레벨 등록 + 단가($3/$6) 확인
- [ ] 루트 QA 스크린샷 png 2개(`hc-xero-card-initial.png`/`hc-customerdetail-card.png`) 정리
