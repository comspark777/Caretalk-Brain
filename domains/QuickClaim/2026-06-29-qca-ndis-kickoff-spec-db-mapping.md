---
domain: QuickClaim
date: 2026-06-29
tags: [QuickClaim, QCA, NDIS, 청구, API연동, PRODA]
status: in-progress
related: ["[[_INDEX]]"]
---
# QCA NDIS 연동 킥오프 — 스펙 분석 + DB 매핑 조사

## 한 일
- **Ben(QC) 온보딩 메일 분석**: NDIS DPO가 ODC의 Digital Partner 신청 승인 → PRODA **B2B Device Activation** 단계(사람이 quickclaim 웹앱+PRODA에서 수동 활성화). QCA=NDIS 인증 Digital Partner 미들웨어(Salesforce 기반).
- **스펙 문서 2종 정독**: `quickclaim Connector App - V2 (billing).pdf`(12p, OpenAPI 포함) + `QCA_V2_NDIS_Spec_for_Suho.xlsx`(Elly/Jasmin 작성: 5 API + Transaction 48필드 + claimType 8종 + Status Flow + Ben 질문 22개).
- **★방향 정정**: 처음 "CareTalk가 QC에 업로드"로 이해 → 실제는 **CareTalk이 5개 REST API를 노출(서버), QCA가 호출(클라이언트)**. PDF: "APIs provided by the customer".
- **DB 조사(테스트DB, MCP)**: 거래 2소스 식별, ServiceState 코드맵 확정, 기존 청구 SP 분석.
- **매핑 후보표 작성**: ClaimExcel SP 출력이 QCA 필드와 1:1 대응 확인.

## 결정/맥락 (왜)
- **자세한 사양은 Ben과 협의하며 진행하기로** (에릭 결정, 2026-06-29). 미정 항목은 Ben 답변 후 확정.
- **5개 API** (전부 `CareTalk.WebAPI` 신규, 인증 헤더 `apiKey`):
  1. `GET /transactions` — 청구대상 & 미EXPORTED 거래 배열 (QCA polling)
  2. `PUT /exported-transactions` — EXPORTED 마킹(중복방지)
  3. `PUT /payment-result` — NDIS 결과(SUCCESSFUL/INVOICED/ERROR) 통보
  4. `GET /deleted-transactions` — fromDate 이후 삭제거래 ID
  5. `PUT /cancel-payments` — NDIA/PM 취소·환불 통보
- **거래 = 2소스**: `CARETALK_ServiceOrderItems`(transactionId=ServiceOrderItemId) + `CARETALK_Reimbursement`(+`_Detail`, ReimSeq).
- **ServiceState 코드맵** (`CARETALK_Codes` CategoryId=22): 1=Ready(대기), **12=Service Finished(완료=청구기준)**, 8=Cancel-CC, 11=Cancel-SC, 18=Refund(→API5), 16=Paid, 9=In Service. → 에릭 말씀(1=대기/12=완료) 코드로 검증 완료.
- **청구 대상 필터** = `ServiceState IN (12,11,8) AND isDisable=1`(Lock=청구확정) `AND Paid_Price>0` — 기존 `usp_CARETALK_PM_select_ClaimList`/`_ClaimExcel`이 쓰는 그대로.
- **★매핑 원본 = `usp_CARETALK_PM_select_ClaimExcel`**: else 분기가 이미 NDIS Bulk Upload 청구 엑셀 생성. 출력 ↔ QCA 1:1:
  - ndisNumber=`Users.CustomerRegistrationNo`(9자리 검증) / supportItem=`ClaimCode` / unit=Σ`TimeUnitCount` / rate=Σ`ConsumerPrice` / GST=P2(NDIS GST-free) / registrationNumber·ABN=`CARETALK_Config`(REGISTRATION_NUMBER, SITEINFO_ABN) / paymentType 단서=`Users.PlanManager`.
  - 즉 QCA 연동 = 기존 "엑셀 다운로드→NDIS 포털 수동 업로드"를 API 자동화로 대체.

## 미정 4가지 (Ben 협의/에릭 판단 대기)
1. **claimType 변환**: CareTalk `DRIVE/Hour/km` ↔ QCA `TRAN/NF2F/CANC-NSDO`(8종). +Ben enum 정의 필요.
2. **paymentType**(agency/plan/self): `Users.PlanManager` 추론 vs 명시 컬럼.
3. **EXPORTED 상태**: `isDisable`(Lock) 외 "QCA 수령" 표시 신규 컬럼/테이블 필요.
4. **청구 단위**: ClaimExcel=고객+supportItem 집계행 vs QCA=개별 transactionId 추적 — GET row 단위 결정.

## 다음 할 일
- [ ] Ben과 미정 4가지 + 22개 질문(Sandbox/prod URL·키, sync 주기, program/cost/location code 등) 협의
- [ ] 협의 결과 반영해 설계 문서(spec) 정리
- [ ] 5개 API 구현 (SP → Repository → Service → Controller, GET /transactions부터)

## 참조
- 스펙 추출본(텍스트): 세션 scratchpad `pdf_text.txt` / `xlsx_dump.txt`
- 글로벌 메모리: `project_quickclaim_qca_integration.md`
- 연락: Ben(QC COO) / Elly(ODC) / Jasmin Min(ODC Plan Manager). Phase1=NDIS, SaH=Phase2.
