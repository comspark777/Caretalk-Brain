---
domain: QuickClaim
updated: 2026-06-29
status: 킥오프 — 스펙 분석+DB 매핑 조사 완료, Ben 협의하며 진행(미정 4가지). 구현 전 설계 단계.
---
# QuickClaim (QCA NDIS 청구 연동)

## 현재 상태
호주 ODC의 NDIS 정부 수가 청구 자동화. **CareTalk이 5개 REST API를 제공(서버), QCA(QuickClaim Connector App, NDIS Digital Partner 미들웨어)가 호출(클라이언트)**.
- **방향**: CareTalk(API 제공자) → QCA → PRODA(B2B Device 인증) → NDIS. 메일의 PRODA B2B Activation은 QCA↔NDIS 구간(ODC 수동 온보딩).
- **5 API**: GET /transactions, PUT /exported-transactions, PUT /payment-result, GET /deleted-transactions, PUT /cancel-payments (인증 헤더 `apiKey`, 전부 `CareTalk.WebAPI` 신규).
- **거래=2소스**: `ServiceOrderItems` + `Reimbursement`(+`_Detail`).
- **매핑 원본**: `usp_CARETALK_PM_select_ClaimExcel`(이미 NDIS Bulk Upload 엑셀 생성, QCA 필드와 1:1). ndisNumber=`Users.CustomerRegistrationNo`, supportItem=`ClaimCode`, 상수=`CARETALK_Config`.
- **청구 대상**: `ServiceState IN (12,11,8)`(12=완료) + `isDisable=1`(Lock) + `Paid_Price>0`.
- **미정 4가지(Ben 협의 대기)**: ① claimType 변환 ② paymentType 구분 ③ EXPORTED 신규상태 ④ 청구단위(집계 vs 개별).

## 세션 로그
<!-- /wiki-save 가 최신 로그를 [[YYYY-MM-DD-slug]] 형태로 여기에 추가 -->
- [[2026-06-29-qca-ndis-kickoff-spec-db-mapping]] — 킥오프: Ben 온보딩 메일+스펙2종 분석, 방향 정정(CareTalk=API서버), DB 조사(2소스·ServiceState·ClaimExcel 매핑), 미정 4가지 도출
