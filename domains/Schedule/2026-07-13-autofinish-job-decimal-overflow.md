---
domain: Schedule
date: 2026-07-13
tags: [incident, autofinish-job, sp-patch, decimal-overflow, contribution]
status: done
related: ["[[_INDEX]]", "[[2026-07-06-service-item-price-batch-update]]"]
---
# 자동마감 JOB 8일 연속 실패 — decimal(5,2) 오버플로 장애 분석·패치·복구

## 한 일
- **장애 확인**: 실서버 `Daily_Auto_Finish_Schedules` JOB(매일 00:02)이 2026-07-06~07-13 **8일 연속 failed**. 자정 일괄마감 클러스터(평소 73~156건/일) 소멸, 미마감 백로그 506건 누적.
- **원인 확정**: `ServiceOrderItemId 176187` (ConsumerPrice **1020.00**, ServiceItemId 478 = 인디 ContributionKind=1, ToTime 2026-07-05). `usp_CARETALK_SOIS_Upsert_Job_ServiceStateAutoFinish`의 `@V_ContributionPrice decimal(5,2)`(최대 999.99)에 1020.00 대입 → **Arithmetic overflow**. 래퍼(`usp_CARETALK_CUS_Job_Upsert_ServiceSchedule_Autofinish`)가 전 건을 단일 트랜잭션으로 돌려서 1건 에러 = 그날 배치 전체 롤백. 커서가 ToTime 오름차순이라 매일 같은 건에서 사망 → 자가회복 불가.
- **패치 1** (JOB 경로): `CareTalkData/20260713_fix_ServiceStateAutoFinish_contribution_decimal_overflow.sql` — `@V_ContributionPrice` (5,2)→(10,2), `@V_Result` 내부 CAST (5,2)→(10,2). 실서버 적용·검증 완료.
- **패치 2** (수평 전개 5개 SP): `CareTalkData/20260713_fix_contribution_decimal_overflow_other_paths.sql` — 같은 자기부담금 블록 복제 경로 일괄 수정. 실서버 적용·검증 완료.
  - `CUS_Upsert_Manual_Closing_Schedule`(수동 마감) / `SOIS_Upsert_ServiceOrderItemSignatureInfo`(서명 마감) / `SOIS_Upsert_ServiceState_Canclellation`(취소): 선언+CAST 각 2곳
  - `SOIS_Upsert_Schedule_Delete`(삭제/환불): 저장된 ContributionPrice를 (5,2)로 읽는 경로 — 선언+음수 CAST
  - `CUS_Update_ServiceOrder_ExtraCharge_Payband`: CAST 3곳만 (@V_ContribAmt는 원래 (7,2))
- **복구**: JOB 수동 실행으로 백로그 **506건 → 0건**, 12시대 515건 일괄 마감. 176187도 State 12 + Paid_Price 1020.00 + GOV FUND 차감·히스토리 정상 통과.

## 결정/맥락 (왜)
- **패치 베이스 = 실서버 `OBJECT_DEFINITION` 추출 원문** (저장소 파일 아님). 대조 중 **드리프트 발견**: 실서버는 저장소 `20260513_..._ToFundAmount.sql`과 달리 자기부담금 UPDATE가 `Used_FundAmount` 유지본(헤더 주석엔 05-13 블록이 있는데 코드는 구버전). 실서버 현행 동작을 정본으로 보존하고 타입 확장만 수행.
- msdb 조회 권한이 없어 JOB 이력 대신 **데이터 흔적**(자정 State=12 클러스터 + 백로그 누적 패턴)으로 진단 → SSMS Job History로 교차 확인.
- 07-06 "가격 일괄 보정" 작업과는 **무관** (176187은 07-02 일반 생성 건, 시기만 우연히 겹침).
- 176187 고객은 인디 자기부담금 플래그 미활성이라 Contribution 자체는 미적용 — 쓰이지도 않는 중간 계산값 때문에 JOB이 죽어 있었음.
- 테이블 `ConsumerPrice`는 `decimal(7,2)`인데 SP 지역변수만 (5,2)였던 정밀도 불일치가 근본 원인. 자기부담금 단건 순액이 $1,000을 처음 넘으면서 잠재 결함 발현.

## 다음 할 일
- [ ] 내일(07-14) 아침 자정 00:02 정기 실행 succeeded 확인 (자정 클러스터 부활 여부)
- [ ] SQL Agent의 `Daily_Increase_Fund_A...` JOB 아이콘 빨간 X 상태 확인 (별건 의심)
- [ ] (선택) 래퍼 SP "1건 실패=전체 롤백" 구조를 건별 TRY/CATCH + 실패 로깅으로 개선 검토
- [ ] (선택) 저장소 20260513 파일 vs 실서버 Used_FundAmount 드리프트 — 어느 쪽이 정책 정본인지 확정해 기록
