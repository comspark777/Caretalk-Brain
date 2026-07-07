---
domain: Schedule
date: 2026-07-06
tags: [schedule, service-order, service-item-price, sql, rollback]
status: complete
related: ["[[_INDEX]]", "D:/CaretokProject/CaretalkSolution/CareTokSolutions/CareTalkData/20260706_fast_update_schedule_consumer_prices_by_price_table.sql"]
---
# Service Item Price Batch Update

## 한 일
- `/Admin/Manage/ServiceItem`에서 서비스 아이템 가격 변경 후, 미래 Ready 스케줄의 고객 부담 가격을 고객 브랜치 가격표 기준으로 일괄 보정하는 SQL을 정리했다.
- 최종 대상은 NDIS parent `ServiceItemId = 190,219` 하위 서비스 아이템으로 확정했다.
- 일정 시간 변경이 없다는 전제를 확인하고, `usp_CARETALK_CRT_Insert_CalculateServiceTime` 재계산 방식은 사용하지 않기로 했다.
- 빠른 적용용 SQL 스크립트 `CareTalkData/20260706_fast_update_schedule_consumer_prices_by_price_table.sql`을 만들었다.
  - 미리보기 SP: `usp_CARETALK_CUS_Select_Schedule_ConsumerPriceFastPreview` (`20260706_fast...sql:168`)
  - 적용 SP: `usp_CARETALK_CUS_Update_Schedule_ConsumerPrices_Fast` (`20260706_fast...sql:395`)
  - 롤백 SP: `usp_CARETALK_CUS_Rollback_Schedule_ConsumerPrices_Fast` (`20260706_fast...sql:810`)
- 롤백용 영구 테이블을 추가했다.
  - `CARETALK_ServiceOrderPriceUpdateRollbackRuns` (`20260706_fast...sql:73`)
  - `CARETALK_ServiceOrderItemPriceRollback` (`20260706_fast...sql:108`)
  - `CARETALK_ServiceOrderItemDetailPriceRollback` (`20260706_fast...sql:135`)
- `ClaimCode` 관련 collation 충돌(`Korean_Wansung_CI_AS` vs `SQL_Latin1_General_CP1_CI_AS`)이 발생해 `COLLATE DATABASE_DEFAULT`를 명시하도록 수정했다 (`20260706_fast...sql:304`, `20260706_fast...sql:530`).

## 결정/맥락 (왜)
- 가격표 브랜치는 서포터 브랜치가 아니라 **고객 브랜치 기준**으로 확정했다.
- 스케줄의 시간대 분할/일정 변경은 없으므로, 기존 `CARETALK_ServiceOrderItemDetails.PayType`과 `TimeUnitCount`를 유지한 채 `ConsumerPrice`와 `ClaimCode`만 새 가격표에 맞춘다.
- 대량 업데이트 후 되돌릴 수 있어야 하므로 적용 직전 원본값을 `RunId` 기준으로 저장한다.
- `@I_AllOrNothing = 0` 배치 커밋 방식을 사용해 락과 로그 부담을 줄였다. 완료 후 롤백은 SQL 트랜잭션 롤백이 아니라 백업 테이블 기반 복구 SP로 처리한다.

## 검증/결과
- 스크립트 문법 검사: `Parsed 14 batches OK`.
- 실제 적용 RunId: `B381DB94-76C2-4FF9-BD70-D851A67A0DEE`.
- 적용 결과:
  - 후보 스케줄: 15,198건
  - 후보 Detail: 19,915건
  - 업데이트 스케줄: 15,198건
  - 업데이트 Detail: 19,915건
  - 소요 시간: 5초
  - 상태: `Applied`
  - 오류: 없음
- 롤백 백업:
  - `ServiceOrderItems` 백업: 15,198건
  - `ServiceOrderItemDetails` 백업: 19,915건
- 사후 검증:
  - 남은 변경 대상 스케줄: 0건
  - 남은 변경 대상 Detail: 0건
  - 누락 가격표 Detail: 0건

## 다음 할 일
- [ ] 같은 방식의 가격 대량 변경이 다시 발생하면 미리보기 SP로 변경 대상/누락 가격표를 먼저 확인한다.
- [ ] 적용 후에는 `RunId`를 기록하고, `CARETALK_ServiceOrderPriceUpdateRollbackRuns`의 `Status = Applied` 및 백업 건수를 확인한다.
- [ ] 가격 적용 결과에 문제가 있으면 `usp_CARETALK_CUS_Rollback_Schedule_ConsumerPrices_Fast`에 해당 `RunId`를 넣어 복구한다.
