---
domain: Schedule
date: 2026-07-20
tags: [schedule, service-order, service-item-price, agedcare, claimcode, sql, rollback]
status: complete
related: ["[[_INDEX]]", "[[2026-07-06-service-item-price-batch-update]]", "CareTalkData/20260720_fast_update_agedcare_consumer_prices_by_price_table.sql"]
---
# Aged Care Price Batch Update (16:30 cutoff)

## 한 일
- 07-06 NDIS(190,219) 가격 일괄 보정과 **동일한 방식**으로, 이번엔 **Aged Care(루트 ServiceItemId=1)** 하위 미래 Ready 스케줄의 고객 부담 가격/ClaimCode를 고객 브랜치 가격표 기준으로 일괄 보정했다.
- 저번 SP 3종을 그대로 재사용하되 **날짜 컷오프를 파라미터화**(`@I_MinFromTime datetime = NULL`, NULL이면 기존 `GETDATE()`)한 새 스크립트를 만들었다: `CareTalkData/20260720_fast_update_agedcare_consumer_prices_by_price_table.sql`.
  - 미리보기 SP: `usp_CARETALK_CUS_Select_Schedule_ConsumerPriceFastPreview`
  - 적용 SP: `usp_CARETALK_CUS_Update_Schedule_ConsumerPrices_Fast`
  - 롤백 SP: `usp_CARETALK_CUS_Rollback_Schedule_ConsumerPrices_Fast`
  - 롤백 백업 테이블 3종은 07-06 것 그대로 재사용(`IF NOT EXISTS` 가드).
- 에릭이 실서버 SSMS에서 직접 배포·실행(`@I_ParentServiceItemIds=N'1'`, `@I_MinFromTime='2026-07-20 16:30:00'`, `@I_ConfirmApply=1`, `@I_ActionUserId=N'admin'`).

## 결정/맥락 (왜)
- **컷오프를 16:30으로 고정한 이유**: 코덱스는 `FromTime >= 오늘 0시`로 235건(대상 4,282)을 냈고, 기존 SP는 `FromTime > GETDATE()`(당시 17:5x)로 233건(대상 4,274)을 냈다. 차이 8건(대상)/2건(변경)/$53은 **오늘 낮 이미 지나간 스케줄** 때문. CUS 조인·IsDisable은 무관(검증). 이미 지난 스케줄을 소급 보정하지 않기 위해 16:30 컷오프를 파라미터로 넣어 233건 기준으로 확정.
- **가격표 없으면 대상 아님**: Aged Care 하위 미래 Ready 스케줄은 수천 건이지만, 대부분 현재 가격 = 가격표라 변동 0. 실제 변경은 가격표가 새로 반영된 것만.
- ClaimCode는 헤더(`ServiceOrderItems`)와 상세(`ServiceOrderItemDetails`) **두 벌 모두** 저장/갱신. Pay(서포터 지급가)·CostUnit·PayType·TimeUnitCount는 건드리지 않음(고객 부담가·청구코드만).

## 검증/결과
- 실행 RunId: `8EFFFC8C-E1B7-4B5D-A946-B4B77C0318EC`, 18:35:11, Status=`Applied`, 오류 없음.
- 적용: **233 스케줄 / 281 Detail**, 가격 증가 총 **+$4,494.75**.
  - 변경 유형 분류: Price만 199 detail / ClaimCode만 82 detail / 둘 다 0.
- 대상 분해(브랜치·아이템):
  - BRISBANE 478(6건, +$168) / BRISBANE 695(41스케줄·82디테일, ClaimCode 공백→SERV-0020)
  - SYDNEY 451(98건, +$1,960) / SYDNEY 478(57건, +$1,638) / SYDNEY 532(31건, +$728.75)
  - 그 외 브랜치(CANBERRA·MELBOURNE 등) 변경 없음. 영향 고객 23명.
- 사후 검증(실서버 SELECT):
  - 남은 변경 대상: **0건**(미리보기 재실행 빈 결과).
  - 478 샘플(OrderItemDetailId 535250~52): ConsumerPrice `198.00`, UpdateDate 18:35, ActionUserId=admin.
  - 695 ClaimCode: Filled=82, StillEmpty=0.
  - 롤백 백업: Sched 233 / Detail 281.
- 참고: SERV-0020 = "Assistance with self-care and activities of daily living" 계열(478·536·608·695) 공통 청구코드. 별도 코드설명 마스터 테이블은 DB에 없음(가격표 BudgetCode 컬럼에만 존재). BRISBANE 695 가격표는 오늘이 아니라 04-14 생성·07-07 수정.

## 다음 할 일
- [ ] 문제 발생 시 RunId `8EFFFC8C-...`로 `usp_CARETALK_CUS_Rollback_Schedule_ConsumerPrices_Fast` 롤백.
- [ ] 향후 같은 배치는 `@I_MinFromTime`으로 컷오프를 명시(누락 시 GETDATE()=실행시각 이후만).
