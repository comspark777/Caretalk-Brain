---
domain: QuickClaim
date: 2026-07-16
tags: [quickclaim, qca, ndis, sandbox, api-test, transactions]
status: in-progress
related: ["[[_INDEX]]", "bDocs/quickclaim-public-api-endpoints.md (2026.07.14_QuickClaim 워크스페이스)"]
---
# Quickclaim Sandbox 접속·회원등록·인보이스 업로드 실증

## 한 일
- **자격증명 정리**: `D:\CaretokProject\CaretalkSolution\_secrets\quickclaim-api.json`을 Production/Sandbox 2단 구조로 재구성. Sandbox = stage-app(웹)·stage-api(API)·OrgId·ApiKey (에릭이 값 직접 채움. 키 원문은 vault에 기록하지 않음).
- **인증 방식 규명**: 처음 `apiKey`/`Authorization` 헤더로 401 → 공식 위키에서 확인한 정답은 **`org-id` + `x-api-key` 헤더 쌍**. `GET /public/participant` 200 OK로 인증 검증 완료.
- **회원(참가자) 등록 성공**: `POST /participant` — participantId **68416** (CareTalk TestUser, ndisNumber=430290495=quickclaim-test.json의 테스트 번호).
  - ★GitHub README(v2.0.3)의 `{"participants":[...]}` 배열 형식은 **400 거부**. 실서버(v3.0.1)는 **단일 객체**만 허용. 문서≠구현 실증.
- **인보이스(거래) 업로드 시도**: `POST /transactions` (batchName=CT-TEST-BATCH-0001, plan-managed). import 자체는 200이나 검증 실패 시 `invalidTransaction`으로 분류됨. 에러 메시지를 따라 3회 반복:
  - 해결: `approved`(0/1 필수), 날짜 형식 `YYYY-MM-DD HH:mm`, `billingName` 필수.
  - **미해결 1개**: `registrationNumber` — 필수인데 임의 값(4050000000)은 "not valid", 누락하면 "missing". 조직에 등록된 실제 NDIS rego가 필요한 것으로 추정.
- **전체 API 카탈로그 확보**: api-docs.quickclaim.io가 Postman 문서 사이트임을 발견, 컬렉션 JSON 원본을 통째로 다운로드(204줄 엔드포인트 트리). 정리본을 워크스페이스 `bDocs/quickclaim-public-api-endpoints.md`에 저장.

## 결정/맥락 (왜)
- **★설계 논점 (내일 최대 이슈)**: Public API에 `POST /transactions`(우리→QC push) + `GET /transaction/{customerTransactionId}`(결과 폴링) + `DELETE /transaction`(Error만 삭제)이 실존한다. 즉 기존에 설계하던 **"QCA가 우리 5 API를 pull"하는 모델의 대안으로 "우리가 push"하는 모델이 가능**하다. `POST /transactions`의 필드가 QCA V2 스펙 48필드와 동일하므로 매핑 작업은 양쪽 모델에서 재사용 가능. Ben과 어느 모델로 갈지 확정 필요.
- **idempotency 설계 근거 확보**: 같은 `customerTransactionId`를 재전송하면 invalidTransaction에 **중복 행이 누적**된다(upsert 아님). 우리 쪽 재시도 로직은 전송 전 상태 확인 또는 고유 ID 재발급 전략이 필요.
- **"최종 JSON을 Ben에게 받아야 한다"는 기존 우려가 실증됨**: 공식 README 예시(v2)와 실서버(v3) body 형식이 이미 불일치.
- `GET /transaction` 응답의 `paymentRequestStatus` enum 실측 문서값: SUCCESSFUL / INVOICED / ERROR / PENDING PAYMENT / PENDING CANCEL / CANCELLED — 기존 상태 모델 초안(APPROVED→EXPORTED→…)과 대조할 것.
- Digital Invoice 계열(`api.di.quickclaim.io`, provider→PM 인보이스)은 **별도 인증(tenant-id)** 이라 현재 자격증명으로 사용 불가. Support At Home 계열은 Phase 2.

## 다음 할 일
- [ ] `registrationNumber` 해결: CARETALK_Config의 실제 REGISTRATION_NUMBER 조회(이번 세션 sqlcmd 무응답, 재시도) → sandbox 재전송 → `transaction[]`(유효) 진입 확인. 실패 시 Ben에게 sandbox org rego 세팅 요청.
- [ ] plan/budget mock 데이터 세팅 (stage 앱 또는 Ben 요청) → `GET /plan/430290495` 검증.
- [ ] Ben과 push(`POST /transactions`) vs pull(QCA가 우리 5 API 호출) 모델 확정.
- [ ] invalidTransaction 중복 누적을 반영한 재시도/idempotency 설계.
- [ ] 상태 모델 초안 ↔ 실측 paymentRequestStatus enum 정합화.
