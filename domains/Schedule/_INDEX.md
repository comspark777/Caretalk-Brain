---
domain: Schedule
updated: 2026-07-06
status: active
---
# Schedule

## 현재 상태
- 서비스 아이템 가격 변경 후 미래 Ready 스케줄 가격을 고객 브랜치 가격표 기준으로 일괄 보정하는 빠른 SQL 배치를 적용 완료했다.
- 적용 대상은 NDIS parent `ServiceItemId = 190,219` 하위이며, 15,198개 스케줄 / 19,915개 Detail 업데이트 및 롤백 백업 검증까지 완료했다.

## 세션 로그
<!-- /wiki-save 가 최신 로그를 [[YYYY-MM-DD-slug]] 형태로 여기에 추가 -->
- [[2026-07-06-service-item-price-batch-update]]
- [[2026-07-01-supporter-gps-depart-arrive]]
