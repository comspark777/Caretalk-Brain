---
domain: Contribution
date: 2026-06-29
tags: [contribution, backfill, fund-amount, late-activation, aged, collation]
status: done
related: ["[[_INDEX]]"]
---
# 자기부담금 늦은 활성화 누락 백필 (5월분, 7명)

## 한 일
- **배경**: 7명 AGED 고객이 2026-06-26 Budget Manage에서 Contribution(자기부담금)을 뒤늦게 활성화 → 활성화 이전에 마감된 스케줄(Log_Kind=474) 적립대상 건이 무적립으로 누락됨. (메모리 §5 서명 SP 버그와는 다른 케이스 — 고객 설정이 늦게 켜진 것)
- **대상 7명** (UserType=1, ServiceType=478 AGED): youngjonghwang194110(L2), kwangungpark194309(L2), josephsong194412(L3), yanxingzhu195308(L3), xiuyinghuang195212(L3), tianshuyang195012(L3), xuejinlin194804(L2).
- **조사 (실서버)**:
  - 테스트서버엔 이 고객들 없음(ID 체계 다름 — `(HL)AnnKim1948`형). 실서버 데이터로만 점검.
  - `Customer_BudgetInfo_Detail` FundCategoryNo=84 행에 Contribution 설정(IndChk/LivChk=1), `UpdateDate` 전원 2026-06-26 11~12시 = 활성화 시점 확정.
  - 요율 고객별 상이: 대부분 Ind 5% / Liv 17.5%, xiuyinghuang 7.95/7.95, yanxingzhu 8.03/21.71.
  - 474 무적립 **54건 전량**(2025-11~2026-06), HasContrib=0 → 활성화가 트리거임을 데이터로 확정.
  - **5월분 14건 / $282.80** (에릭 지시로 5월만 처리).
  - **636 인보이스는 별개 메커니즘**(CLAIM 카테고리 기반): 활성화와 무관하게 이미 적립 중, 무적립/적립이 같은 기간 공존(무적립 MaxPay 전원 2026-05-31에서 끊김) → 'Clinical supports' 등 정상 비적립. **소급 대상 아님.**
- **적립 공식/버킷 검증** (다른 고객 적립건 역검증):
  - `ContributionPrice = Amounts × Fee%`, 행별 `decimal(5,2)` 반올림 후 합산(마감 SP와 동일). `Amounts = ConsumerPrice − Paid_Point − Paid_Deposit`.
  - 적립 버킷 = `PaymentInfo_History.Contribution_DetailId` → `FundCategoryNo` **91(INDEPEN, Kind=1) / 92(LIVING, Kind=2)** 행의 `Used_FundAmount`에 누적. (설정행 84 ≠ 적립버킷 91/92)
  - 마감 SP는 474 INSERT 시 같은 행에 Contribution 3컬럼 인라인 기입(`20260610_alter_SignatureInfo_add_contribution.sql:248-250`); **백필은 기존 무적립 474 행을 UPDATE**.
- **백필 스크립트**: `CareTalkData/20260626_backfill_late_contribution_may.sql` (프리뷰 기본 @Execute=0 / 감사테이블 `CARETALK_Fix_LateContribution_Audit` / FixRunId 롤백). 기존 `20260610_backfill_signature_contribution.sql` 패턴 재사용.
- **적용·검증 완료**: @Execute=1 실행 → 14건 Contribution 기입 + 감사테이블 14행 + 버킷 7개 `Used_FundAmount` 정합성 **7/7 OK**(Before+Added=Now). 총 **$282.80**.

## 결정/맥락 (왜)
- **5월분만**: 에릭 지시. 4월 이전(youngjonghwang 2025-12, josephsong 2025-11 등) 나머지 40건/약 $1,122는 보류.
- **collation 충돌 트러블슈팅**: `#Targets` 임시테이블(tempdb collation `SQL_Latin1...`) ↔ DB `Korean_Wansung_CI_AS` JOIN 충돌(에러 468). `COLLATE DATABASE_DEFAULT`는 SSMS 연결 컨텍스트에 의존해 불충분 → **`#Targets` 제거하고 `WHERE P.UserId IN (7명)` 인라인 전환**(문자열 리터럴은 컬럼 collation을 따라가 충돌 원천 차단).
- **@Execute 기본 0 안전장치**: 첫 실행은 프리뷰만 돌아 미적용. 프리뷰 SELECT 4개는 @Execute 무관하게 항상 출력되므로, **적용 여부는 메시지 탭(`백필 적용 완료. FixRunId=`)·감사테이블 존재·실데이터로 판별**해야 함(결과 탭만으론 구분 불가).
- 반올림: 행별 합 $282.80이 시스템 실제 동작(저장값). 전체합 반올림 $282.77은 표시용 차이.

## 다음 할 일
- [ ] 4월 이전 나머지 40건(~$1,122) 소급 여부 결정 → 같은 스크립트 `@FromDate`/`@ToDate`만 변경해 재실행.
- [ ] 정산서 재발행 시 11/05·18/05 General house cleaning 등 Contribution 칸 표시 확인(이전 `—` 해소 검증).
