---
domain: QuickClaim
date: 2026-07-20
tags: [quickclaim, qca, ndis, sandbox, push-model, registration-number, design-decision]
status: in-progress
related: ["[[_INDEX]]", "[[2026-07-16-sandbox-connect-participant-invoice-test]]", "bDocs/quickclaim-public-api-endpoints.md (2026.07.14_QuickClaim 워크스페이스)"]
---
# push 모델 확정 · registrationNumber는 대시보드 등록 필요로 판명

## 한 일
- **설계 모델 확정 (push)**: Ben이 "구독이 있으니 직접 개발하라"고 정리 + Ben 측 지원 없음. → 기존에 검토하던 **"QCA가 우리 5 API(GET /transactions 등)를 pull"하는 모델은 폐기**. CareTalk이 클라이언트가 되어 Quickclaim Public API를 직접 호출하는 **push 모델**로 간다.
- **REGISTRATION_NUMBER 값 확정**: 07-16 무응답이던 sqlcmd 재시도 성공. `CARETALK_Config.Config_Key='REGISTRATION_NUMBER'` = **4050036107** (= `ONE_DREAM_COMMUNITY_NDIS_NUMBER`와 동일). 조직 상수 참고: `SITEINFO_ABN`=88 625 363 091, `SITEINFO_NAME`=One Dream Community.
- **sandbox 재전송 실증**: 실제 rego 4050036107로 `POST /transactions` 재전송 → 여전히 `transaction[]` 비어있고 `invalidTransaction`에 **"registrationNumber is not valid!"**. 문서 샘플 rego(4050000201, 4000000001)도 동일 거부.
- **근본 원인 규명**: Postman 컬렉션 원문에서 확인 — rego는 값 형식 문제가 아니라 **QC 대시보드에 사전 등록되어 있어야** 함. 공식 문서 문구: *"Must be provided in the QC dashboard"* / *"For agency-managed transactions, the registrationNumber ... is used to map the regoId from our internal registration system. You need to define them in quickclaim dashboard before using this API."* → sandbox org 131132에 rego 미등록 상태가 원인.
- **문서 갱신**: `bDocs/quickclaim-public-api-endpoints.md`에 설계 확정(push) 섹션 추가, registrationNumber 검증 규칙에 "대시보드 사전 등록 필요" 실측 반영, 할 일 목록 갱신.

## 결정/맥락 (왜)
- **push 모델 채택 근거**: Ben이 개발 주체를 우리로 못박음 + 지원 없음. pull 모델은 QCA(미들웨어)가 우리 서버를 호출하는 구조라 상대 측 개발·조율이 전제인데, 그 전제가 사라졌다. push는 Quickclaim Public API가 이미 `POST /transactions`(전송)·`GET /transaction/{id}`(폴링)·`DELETE /transaction/{id}`(Error만 삭제)를 제공하므로 우리 단독으로 완결 가능.
- **설계 전환 영향**: `CareTalk.WebAPI`에 5개 신규 API를 만드는 작업은 불필요. 대신 **QuickClaim 클라이언트 서비스**(전송 대상 추출 → POST → 폴링 → 상태 반영) 구현으로 방향 전환. 필드 매핑(48필드)은 양쪽 모델 공통이라 재사용 가능 — `usp_CARETALK_PM_select_ClaimExcel`이 원본.
- **rego "not valid"는 우리가 못 푸는 단계**: 대시보드 조직 설정에 rego 등록이 선행돼야 함. 로그인(비밀번호 입력)은 규칙상 대신 불가 → 에릭 처리 필요.

## 다음 할 일
- [ ] **(에릭)** stage-app.quickclaim.io 로그인 → 조직 설정에 rego `4050036107` 등록. 또는 Chrome에 Claude 확장 연결(로그인된 세션)해주면 대시보드 직접 탐색 가능. 등록 후 재전송하여 `transaction[]`(유효) 진입 확인.
- [ ] participant 430290495에 plan/budget mock 세팅 → `GET /plan/430290495` 검증.
- [ ] **push 모델 클라이언트 설계 착수**: 전송 대상 추출(ClaimExcel SP 매핑 = ndisNumber·supportItem·상수), 상태 모델(paymentRequestStatus enum 정합), idempotency(같은 customerTransactionId 재전송 시 invalid 행 중복 누적 → 전송 전 상태 확인 또는 고유 ID 전략).
