---
domain: Infra-DB
date: 2026-07-01
tags: [userid, data-fix, live-server, ndis, zerosum]
status: done
related: ["[[_INDEX]]"]
---
# 실서버 UserId '430219792-' → '430219792' (하이픈 제거) 정정

한 유저(UserName=Seung Kwan Choi, UserType=1)의 UserId 끝 하이픈을 떼서, 흩어진 전 참조를 일관되게 정정한 작업.

## 한 일
- **전수 스캔**: 실서버에서 `sys.columns` 문자열 컬럼(varchar/nvarchar/char/nchar) 전체를 커서로 돌며 `= '430219792-'` 완전일치 집계 → **28테이블 / 29컬럼**에서 발견(총 ~1,288행). 최다: `CARETALK_Statements.UserId` 536, `CARETALK_Log.UserId` 389, `CARETALK_PaymentInfo_History.UserId` 187, `CARETALK_Reimbursement.ConsumerName` 74(★이름 컬럼이지만 UserId 저장이 의도됨), `CARETALK_Document_Manage.UserId` 60.
- **사전 점검**: 대상값 `430219792`(하이픈 없) 사전 존재 0건(PK 충돌 없음) / `CARETALK_Users.UserId`는 PK / 이를 참조하는 FK 4개(`RefreshToken`·`ReferralTree`·`Points`·`PointHistory`)는 모두 `ON UPDATE NO_ACTION`.
- **정정 스크립트 작성**: `CareTalkData/20260701_fix_userid_430219792_remove_hyphen.sql`
  - `@Commit` 토글(0=dry-run 자동 롤백 / 1=apply), `XACT_ABORT ON`, PK 충돌 가드.
  - FK 4개 `NOCHECK` → 전 컬럼 UPDATE → `WITH CHECK CHECK`로 재활성화(참조 무결성 재검증).
  - 재검증용 전수 재스캔은 락 유지 최소화 위해 트랜잭션 밖(커밋 후 read-only)으로 분리.
- **/review 반영**: FK 참조 4개 테이블의 `UserId`도 방어적 UPDATE에 포함(dry-run과 apply 사이 로그인 토큰 등 생기면 재활성화 실패 방지). 컬럼 커버리지 29/29 대조 확인.
- **에릭 SSMS `@Commit=1` 실행 후 검증**(read-only 재스캔): 옛값 잔존 **0건**, 새값 `430219792` **1,378행 / 32컬럼**.

## 결정/맥락 (왜)
- **원래 두 표기가 공존했다**: 새값 총량(1,378) > 옮긴 양(1,288). 차이 ~90행은 **원래부터 하이픈 없는 `430219792`를 갖던 데이터**. 우리가 안 건드린 3곳이 대표적:
  - `CARETALK_File_BulkUpload_Detail.NDIS` 20건, `CARETALK_ZeroSum.UserId` 2건, `CARETALK_Users.CustomerRegistrationNo` 1건.
  - `430219792`는 **9자리 NDIS 참여자번호 형식**. UserId는 NDIS번호에 하이픈 붙인 형태였고, 하이픈 제거로 **UserId = NDIS번호 = CustomerRegistrationNo**로 일치 → 정합성 개선. 에릭 "의도한대로 잘 맞어" 확인.
- **`CARETALK_ZeroSum`은 Xero와 무관**(에릭 확인). 이름(Zero~/Xero)·브랜치명(zero-accounting) 때문에 혼동 주의. → 메모리 `reference_zerosum_not_xero`.
- **실행 주체**: 실서버는 SELECT-only 규칙이라 Claude는 스캔·검증만(read-only), UPDATE는 스크립트만 작성하고 에릭이 SSMS에서 직접 실행.

## 다음 할 일
- [ ] (선택) 스크립트 `CareTalkData/20260701_fix_userid_430219792_remove_hyphen.sql` 커밋 여부 에릭 판단 — 작업 자체는 완료.
