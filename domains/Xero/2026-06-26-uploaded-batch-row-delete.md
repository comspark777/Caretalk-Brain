---
domain: Xero
date: 2026-06-26
tags: [xero, batch, delete, soft-delete, caretalk-pipeline]
status: done
related: ["[[_INDEX]]", "[[2026-06-25-track-c-high-care]]"]
---
# Xero UPLOADED 배치/로우 삭제 기능

UPLOADED 상태 업로드 배치에 대해 (1) 배치 전체 삭제, (2) 배치 내 개별 로우 삭제 추가. caretalk-pipeline 4단계(data→backend→frontend→qa) 협업, QA PASS, 테스트DB 적용·동작 확인.

## 한 일
- **SP 2종 (신규)** — `CareTalkData/20260626_Xero_DeleteBatch.sql`
  - `usp_CARETALK_XER_Delete_UploadBatch(@I_BatchId, @I_ActionUserId)`: 가드(존재+IsDelete=0+Status='UPLOADED') → 트랜잭션으로 하위 아이템 전부 + 배치 헤더 `IsDelete=1`. 반환 스칼라 1/0.
  - `usp_CARETALK_XER_Delete_UploadBatchItem(@I_ItemId, @I_ActionUserId)`: 가드(아이템 존재+IsDelete=0+부모배치 UPLOADED JOIN) → 해당 행만 `IsDelete=1` + 배치 집계(Total/Sent/Skipped/Excluded) 재계산. 반환 1/0.
- **Repository** — `IXeroRepository`/`XeroRepository`: `DeleteUploadBatchAsync`, `DeleteUploadBatchItemAsync` (`ExecuteScalarAsync<int>`, 파라미터 `I_*`).
- **Service** — `IXeroPayrollService`/`XeroPayrollService`: `DeleteBatchAsync`, `DeleteBatchItemAsync` (단순 위임, 가드는 SP 내부).
- **Controller** — `Areas/Admin/Controllers/XeroController.cs`: `POST /Admin/Xero/DeleteBatch {batchId}`(BatchIdRequest 재사용), `POST /Admin/Xero/DeleteBatchItem {itemId}`(신규 DeleteBatchItemRequest). anti-forgery 미적용(기존 POST 액션군과 동일).
- **JS** — `wwwroot/scripts/Xero/xeroPayroll.js`: 좌측 리스트 UPLOADED 배치에만 휴지통 버튼(`.xero-batch-delete`, stopPropagation으로 openBatch 차단), 편집행 끝에 Delete 컬럼(`.xero-row-delete`). 성공판정 `res.returnData === 1`. 배치삭제 성공→`loadBatches()`+패널 닫기, 로우삭제 성공→`refreshCurrentBatch()`. cshtml 변경 없음(전부 JS 동적 렌더).
- **QA**: 분리/대체경로 빌드 3경로 오류 0·신규 경고 0. 경계면(시그니처 체인·camelCase 키·반환 1/0) 전 항목 PASS. 테스트DB 적용 후 동작 정상 확인(에릭).

## 결정/맥락 (왜)
- **소프트 삭제(IsDelete=1)** 채택 — 두 테이블 다 IsDelete 보유, 모든 SELECT가 IsDelete=0 필터라 UI에서 즉시 사라지면서 데이터는 복구 가능. UPLOADED는 미전송 staging이라 안전.
- **서버측 Status='UPLOADED' 가드** — UI 버튼 숨김만으로 부족, SP에서 강제(직접 호출해도 0 반환 차단).
- **로우 삭제 ≠ Exclude 토글** — Exclude(PENDING↔EXCLUDED)는 "보내되 스킵", Delete는 "목록에서 제거".
- **엣지케이스 하드닝**: 배치의 마지막 행 삭제 시 집계 CROSS APPLY의 `SUM(...)`이 빈 집합→NULL, 집계 4컬럼이 INT NOT NULL이라 UPDATE 롤백→삭제 차단됨. SP2에서 SentCnt/SkipCnt/ExclCnt를 `ISNULL(SUM(...),0)`로 감싸 해결(COUNT(*)는 NULL 불가라 제외).

## 다음 할 일
- [ ] 프로덕션 SP 적용(SSMS): `20260626_Xero_DeleteBatch.sql` (테스트DB는 적용 완료)
- [ ] 메인 머지(에릭 승인) — 본 기능 + Track A·B·C·D 함께
- [ ] (선택) 마지막 1행 삭제 = ISNULL 하드닝 핵심 케이스 Playwright E2E
