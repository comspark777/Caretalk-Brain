---
domain: QuickClaim
date: 2026-07-07
tags: [quickclaim, qca, ndis, api-integration, billing]
status: draft-knowledge-note
related: ["[[_INDEX]]", "C:/Users/박수호/Documents/Codex/2026-07-07/new-chat/outputs/quickclaim-caretalk-integration-workflow.md"]
---
# Quickclaim - CareTalk API Integration Workflow

## 한 일
- Quickclaim Connector App(QCA) 연동 방향을 정리했다.
- 핵심 결론: **CareTalk이 Quickclaim을 직접 호출하는 방식이 아니라, QCA가 CareTalk API를 호출한다.**
- QCA 흐름을 `GET /transactions`로 승인 거래를 가져가고, `PUT /exported-transactions`, `PUT /payment-result`, `PUT /cancel-payments`로 결과를 다시 CareTalk에 반영하는 pull-and-push-back 구조로 정리했다.
- CareTalk가 제공해야 할 필수 endpoint를 정리했다.
  - `GET /transactions`
  - `PUT /exported-transactions`
  - `PUT /payment-result`
  - `GET /deleted-transactions?fromDate=...`
  - `PUT /cancel-payments`
- 내부 거래 상태 모델 초안을 정했다.
  - `APPROVED`
  - `EXPORTED`
  - `SUCCESSFUL`
  - `INVOICED`
  - `ERROR`
  - `CANCELLED`
  - `DELETED`
- `transactionId`는 schedule 단위가 아니라 billing line 단위의 stable unique id여야 한다는 원칙을 기록했다.
- Quickclaim에 보낼 sandbox/API payload 확인 메일 초안을 작성했다.

## 결정/맥락 (왜)
- QCA V2 billing specification 기준으로 연동 주체는 Quickclaim Connector App이다. CareTalk은 외부 연동 API 서버 역할을 해야 한다.
- 중복 청구 방지를 위해 `PUT /exported-transactions`는 idempotent하게 처리해야 한다.
- `PUT /payment-result`는 최종 처리 상태뿐 아니라 Quickclaim reference, claim export file, submission date, paid amount, invoice number, error message 등 후속 추적 정보를 저장해야 한다.
- `GET /deleted-transactions`와 `PUT /cancel-payments`는 void, non-billable, cancellation, refund 이후 Quickclaim 쪽 receivable 정리에 필요하다.
- 보안은 HTTPS, sandbox/production 분리 API key, 선택적 Quickclaim IP allowlist, request/response logging, request id 추적을 기본 전제로 둔다.
- PDF field table과 OpenAPI schema가 완전히 일치하지 않으므로 구현 전에 Quickclaim의 최종 JSON payload sample을 받아야 한다.

## Quickclaim 확인 필요
- Sandbox QCA 환경 제공 여부, sandbox/production API key 분리 여부, fixed IP allowlist 필요 여부.
- 모든 endpoint의 최종 sample JSON payload.
- `PUT /exported-transactions`와 문서 일부의 `POST /exported-transactions` 중 실제 구현해야 할 HTTP method.
- `/exported-transactions` payload가 단순 transaction id 배열인지, `transactionId` 배열을 담은 object인지.
- `GET /transactions` response field source of truth: PDF table vs OpenAPI schema.
- `programCode`, `programCostcode`, `locationCode`, `categoryXcode`, `categoryYcode` 필드의 필수 여부.
- `paymentType`별 필수 필드와 QCA pagination 지원 여부.
- 날짜/시간 timezone 및 ISO 8601 offset 필요 여부.
- `unit`, `rate`, `paidAmount`의 JSON number/string 표현 방식.
- GST 없음 표현 방식: `0`, `0.00`, `null`.
- `paymentRequestStatus` 전체 enum, 중복 payment-result 가능 여부, `ERROR` 이후 재노출/재제출 규칙.
- cancellation/refund가 agency-managed와 plan-managed 모두 가능한지, 원거래 취소 vs adjustment 생성 중 어떤 방식이 필요한지.

## 다음 할 일
- [ ] Quickclaim에 sandbox setup 및 final JSON payload sample 요청 메일을 발송한다.
- [ ] Quickclaim 답변 기준으로 endpoint request/response DTO와 field mapping을 확정한다.
- [ ] `EXPORTED` 상태와 idempotency 저장 구조를 설계한다.
- [ ] CareTalk sandbox API endpoint를 구현한다.
- [ ] agency-managed, plan-managed, self-managed, error, corrected resubmission, deleted transaction, cancellation/refund, duplicate retry 시나리오를 QCA sandbox에서 검증한다.
