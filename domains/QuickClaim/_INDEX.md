---
domain: QuickClaim
updated: 2026-07-20
status: ★push 모델 확정(Ben=직접 개발, 지원 없음) — pull(5 API) 폐기, CareTalk이 Quickclaim Public API 직접 호출. REGISTRATION_NUMBER=4050036107 확정했으나 sandbox 여전히 "not valid" → rego는 QC 대시보드 사전 등록 필요(에릭 처리). push 클라이언트 설계 착수 단계.
---
# QuickClaim (QCA NDIS 청구 연동)

## 현재 상태
호주 ODC의 NDIS 정부 수가 청구 자동화. **2026-07-20 push 모델 확정** — Ben이 "구독 있으니 직접 개발" + 지원 없음. 기존 "QCA가 우리 5 API를 pull"하는 서버 모델은 **폐기**. CareTalk이 클라이언트가 되어 Quickclaim Public API를 직접 호출한다: `POST /transactions`(전송) → `GET /transaction/{id}`(폴링) → 상태 반영, Error 시 `DELETE /transaction/{id}`.
- **Sandbox 실증(07-16)**: 인증=`org-id`+`x-api-key` 헤더. participant 등록(68416)·거래 업로드 성공. 자격증명=`D:\CaretokProject\CaretalkSolution\_secrets\quickclaim-api.json`. 전체 엔드포인트 카탈로그=워크스페이스 `bDocs/quickclaim-public-api-endpoints.md`.
- **registrationNumber(07-20)**: `CARETALK_Config.REGISTRATION_NUMBER`=**4050036107** 확정. 그러나 sandbox 재전송 시 여전히 "not valid" — 값 문제 아니라 **QC 대시보드에 rego 사전 등록 필요**("must be provided in the QC dashboard"). sandbox org 131132 미등록 상태 → 에릭이 대시보드 로그인 후 등록 필요.
- **주의**: GitHub README 예시(v2)와 실서버(v3.0.1) body 형식 불일치(POST /participant는 단일 객체만). 같은 customerTransactionId 재전송 시 invalid 행 중복 누적(upsert 아님) → idempotency 설계 반영 필요.
- **push 클라이언트 구성(예정)**: 전송 대상 추출(ClaimExcel SP 매핑) → POST /transactions → 폴링 → 상태 반영. `CareTalk.WebAPI` 신규 5 API는 불필요(폐기).
- **상태 모델**: 실측 `paymentRequestStatus` enum(SUCCESSFUL/INVOICED/ERROR/PENDING PAYMENT/PENDING CANCEL/CANCELLED)과 우리 내부 상태 정합화 필요.
- **transactionId 원칙**: schedule 단위가 아니라 billing line 단위의 stable unique id를 사용해야 한다.
- **거래=2소스**: `ServiceOrderItems` + `Reimbursement`(+`_Detail`).
- **매핑 원본**: `usp_CARETALK_PM_select_ClaimExcel`(이미 NDIS Bulk Upload 엑셀 생성, QCA 필드와 1:1). ndisNumber=`Users.CustomerRegistrationNo`, supportItem=`ClaimCode`, 상수=`CARETALK_Config`.
- **청구 대상**: `ServiceState IN (12,11,8)`(12=완료) + `isDisable=1`(Lock) + `Paid_Price>0`.
- **미정/확인 필요**: Quickclaim sandbox, 최종 sample JSON, HTTP method(`PUT` vs `POST /exported-transactions`), payload shape, field source of truth(PDF table vs OpenAPI schema), payment status enum, ERROR 재제출 규칙, cancellation/refund 처리 방식.

## 세션 로그
<!-- /wiki-save 가 최신 로그를 [[YYYY-MM-DD-slug]] 형태로 여기에 추가 -->
- [[2026-07-20-push-model-confirmed-rego-dashboard-block]] — push 모델 확정(Ben=직접 개발·pull 폐기)·REGISTRATION_NUMBER=4050036107 확정·sandbox 여전히 not valid(rego 대시보드 사전 등록 필요로 판명)·push 클라이언트 설계 착수
- [[2026-07-16-sandbox-connect-participant-invoice-test]] — sandbox 접속 실증(org-id+x-api-key)·participant 등록(68416)·POST /transactions 업로드(registrationNumber만 미해결)·Postman 컬렉션 전체 카탈로그 확보·push vs pull 설계 논점 도출
- [[2026-07-07-quickclaim-caretalk-integration-workflow]] — QCA가 CareTalk API를 호출하는 pull/push-back 워크플로우 정리, 필수 5 API, 상태 모델, idempotency, sandbox/payload 확인 메일 초안 작성
- [[2026-06-29-qca-ndis-kickoff-spec-db-mapping]] — 킥오프: Ben 온보딩 메일+스펙2종 분석, 방향 정정(CareTalk=API서버), DB 조사(2소스·ServiceState·ClaimExcel 매핑), 미정 4가지 도출
