---
domain: QuickClaim
updated: 2026-07-07
status: QCA 연동 워크플로우 정리 완료 — CareTalk=API 서버, QCA=pull/push-back 클라이언트. Sandbox와 final JSON payload 확인 후 DTO/상태/idempotency 설계 필요.
---
# QuickClaim (QCA NDIS 청구 연동)

## 현재 상태
호주 ODC의 NDIS 정부 수가 청구 자동화. **CareTalk이 REST API를 제공(서버), QCA(QuickClaim Connector App, NDIS Digital Partner 미들웨어)가 호출(클라이언트)**.
- **방향**: QCA가 CareTalk `GET /transactions`로 승인 거래를 pull하고, `PUT /exported-transactions`, `PUT /payment-result`, `PUT /cancel-payments` 등으로 결과를 push-back한다. PRODA B2B Activation은 QCA↔NDIS 구간.
- **필수 API**: GET /transactions, PUT /exported-transactions, PUT /payment-result, GET /deleted-transactions, PUT /cancel-payments (인증 헤더 `apiKey`, 전부 `CareTalk.WebAPI` 신규).
- **상태 초안**: APPROVED → EXPORTED → SUCCESSFUL/INVOICED/ERROR, 후속 CANCELLED/DELETED. `PUT /exported-transactions`는 duplicate claim 방지를 위해 idempotent해야 한다.
- **transactionId 원칙**: schedule 단위가 아니라 billing line 단위의 stable unique id를 사용해야 한다.
- **거래=2소스**: `ServiceOrderItems` + `Reimbursement`(+`_Detail`).
- **매핑 원본**: `usp_CARETALK_PM_select_ClaimExcel`(이미 NDIS Bulk Upload 엑셀 생성, QCA 필드와 1:1). ndisNumber=`Users.CustomerRegistrationNo`, supportItem=`ClaimCode`, 상수=`CARETALK_Config`.
- **청구 대상**: `ServiceState IN (12,11,8)`(12=완료) + `isDisable=1`(Lock) + `Paid_Price>0`.
- **미정/확인 필요**: Quickclaim sandbox, 최종 sample JSON, HTTP method(`PUT` vs `POST /exported-transactions`), payload shape, field source of truth(PDF table vs OpenAPI schema), payment status enum, ERROR 재제출 규칙, cancellation/refund 처리 방식.

## 세션 로그
<!-- /wiki-save 가 최신 로그를 [[YYYY-MM-DD-slug]] 형태로 여기에 추가 -->
- [[2026-07-07-quickclaim-caretalk-integration-workflow]] — QCA가 CareTalk API를 호출하는 pull/push-back 워크플로우 정리, 필수 5 API, 상태 모델, idempotency, sandbox/payload 확인 메일 초안 작성
- [[2026-06-29-qca-ndis-kickoff-spec-db-mapping]] — 킥오프: Ben 온보딩 메일+스펙2종 분석, 방향 정정(CareTalk=API서버), DB 조사(2소스·ServiceState·ClaimExcel 매핑), 미정 4가지 도출
