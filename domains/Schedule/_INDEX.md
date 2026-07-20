---
domain: Schedule
updated: 2026-07-20
status: active
---
# Schedule

## 현재 상태
- Aged Care(루트 ServiceItemId=1) 미래 Ready 스케줄 가격/ClaimCode 일괄 보정 실서버 적용 완료 — 16:30 컷오프(`@I_MinFromTime` 파라미터 신규) 기준 **233 스케줄 / 281 Detail, +$4,494.75**(RunId `8EFFFC8C-...`). 남은변경 0·478 198.00·695 ClaimCode SERV-0020 채움·백업233/281 검증. 07-06 SP를 컷오프 파라미터화해 재사용.
- (이전) 자동마감 JOB 8일 실패(07-06~13) 해결 — 자기부담금 `decimal(5,2)` 오버플로(176187·$1020) + 단일트랜잭션 전체롤백. JOB SP+수평 5개 SP (10,2) 확장 실서버 적용, 백로그 506건 마감. 남은 확인: 07-14 자정 실행·`Daily_Increase_Fund` 빨간X.
- (이전) 서비스 아이템 가격 변경 후 미래 Ready 스케줄 가격 일괄 보정 — 190,219 하위, 15,198 스케줄/19,915 Detail, 롤백 백업 검증.

## 세션 로그
<!-- /wiki-save 가 최신 로그를 [[YYYY-MM-DD-slug]] 형태로 여기에 추가 -->
- [[2026-07-20-agedcare-price-batch-update]]
- [[2026-07-13-autofinish-job-decimal-overflow]]
- [[2026-07-06-service-item-price-batch-update]]
- [[2026-07-01-supporter-gps-depart-arrive]]
