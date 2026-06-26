---
domain: Xero
date: 2026-06-26
tags: [xero, upload-batch, select-uploadbatches, deploy, pagination, index, mssql-verify]
status: deploy 통합 완료 · 프로덕션 SSMS 실행 대기
related: ["[[_INDEX]]", "[[2026-06-26-abn-exclude-row-delete-i18n]]", "[[2026-06-26-uploaded-batch-row-delete]]"]
---
# 배치 목록 기본 2개월 필터 + DEPLOY 스크립트 29 SP 통합(테스트DB 권위검증)

## 한 일

### 1) /Admin/xero "Uploaded Batches" 목록 — 조회 제한 없음 진단
- `loadBatches()`(`xeroPayroll.js:430`) → `/Admin/Xero/Batches` → `GetBatchesAsync` → SP `usp_CARETALK_XER_Select_UploadBatches`.
- **SP에 TOP/OFFSET 없음** → `IsDelete=0` 전건을 `ORDER BY BatchId DESC`로 전부 반환. JS도 받은 리스트 전부 DOM 렌더링(슬라이스/페이징 없음).
- 결론: 배치가 수백~수천 건 쌓이면 첫 진입에 전건이 내려와 성능 저하. 현재 테스트DB는 active 2건뿐이라 안 보였던 것.

### 2) 해결 — SP에 기본 최근 2개월 필터(에릭 결정: 페이징 X)
- `CareTalkData/20260626_usp_CARETALK_XER_Select_UploadBatches_Default2Months.sql` (ALTER).
- 로직: `@EffectiveFrom = @I_PeriodFrom; IF @I_BatchId IS NULL AND @I_PeriodFrom IS NULL → DATEADD(MONTH,-2,CAST(GETDATE() AS DATE))`. 필터는 `PayPeriodTo >= @EffectiveFrom`.
- **함정 회피**: `@I_BatchId` 지정 시 기본필터 미적용 → 오래된 배치를 batchId로 직접 조회하는 4곳(`XeroPayrollService.cs:214,1104,1226,1340` — 핸드오버 복귀/락 체크)에서 누락 안 됨.
- 사용자가 PeriodFrom 입력 시 그 값 사용(더 옛날 조회 가능). **C# 무변경(SP만), View/JS 무변경**(Period From/To 필드 기존 존재).
- 테스트DB 시뮬레이션 2케이스 통과(기본=EffectiveFrom 2026-04-26 적용 / batchId 지정=NULL 미적용). 에릭 SSMS 적용 확인(modify 13:04).

### 3) 인덱스 점검 — 보강 불필요 판단
- 기존: `PK_CARETALK_Xero_UploadBatch`(CLUSTERED, BatchId=정렬 커버) + `IX_Xero_Batch_Status_Period`(Status,PayPeriodFrom,PayPeriodTo=필터 커버).
- 데이터 규모 작음(페이롤 배치 주/격주 → 연 수십 건). 2개월 필터로 결과셋 더 작아짐 → 추가 인덱스는 INSERT/UPDATE 비용만 늘고 실익 없음(투기적). 미래 수천 건+첫진입 지연 시에만 `WHERE IsDelete=0` 필터드 인덱스(키 `PayPeriodTo DESC`) 재검토.

### 4) DEPLOY_20260626_Xero_Full.sql 통합 (SP 27 → 29)
- `Insert_UploadBatch` → **ABN 계약자(ContractType 2/13) EXCLUDED** 로직 + 영문 사유 메시지로 교체(구버전 한글·ABN無).
- `Select_UploadBatches` → **기본 2개월 필터** 버전으로 교체.
- `Delete_ExcludedBatchItems`, `Delete_BatchItems` **2개 SP 신규 삽입**(Delete_UploadBatchItem 뒤).
- 헤더(제목·PART2·PART3·SP목록) + PART3 사후검증 개수 **27→29** 갱신.

### 5) 테스트DB 실시간 조회 = 권위 기준 교차검증 (★사용자 지시: 폴더만 보지 말 것)
- `sys.procedures` Xero SP = **29개**, deploy CREATE OR ALTER = **29개** → 이름 집합 1:1 일치.
- `Insert_UploadBatch`(modify 13:06)·`Select_UploadBatches`(modify 13:04) 테스트DB 정의가 이미 ABN·2개월 최신본 = deploy와 동일.
- `Delete_*` 2개 시그니처 검증(@I_ItemIds·STRING_SPLIT·UPLOADED 가드) 일치.
- 테이블 10개 = 정식 9개(deploy 포함) + 백업 1개 `CARETALK_Xero_Batch1_WorkDate_bk_20260624`(06-24 마이그레이션 임시본, 배포 제외 정당).
- **수확**: 파일(`CareTalkData/`)만 봤다면 deploy 미포함이던 `Delete_BatchItems`를 놓칠 뻔 → 실시간 조회로 포착.

## 결정/맥락 (왜)
- **페이징 대신 기본 2개월**: 페이롤 배치는 연 수십 건 규모라 정식 페이징(OFFSET/FETCH+UI)은 과함. 기본 2개월 + 기간필터로 충분.
- **batchId 직접조회 시 기본필터 미적용**: 목록용 가드가 단건 조회를 깨면 안 됨(Service 4곳 의존).
- **deploy 권위 = 테스트DB 실존 객체**: deploy 헤더 철학과 동일. 파일 폴더는 누락/구버전 가능 → 실시간 조회로 확정.
- **백업 테이블 `_bk_` 배포 제외**: 일회성 마이그레이션 백업본.

## 다음 할 일
- [ ] 프로덕션 SSMS에서 `CareTalkData/DEPLOY_20260626_Xero_Full.sql` 전체 F5 실행 (끝 메시지 `테이블 9/9, SP 29/29 ✔` 확인). 멱등(테이블=IF NOT EXISTS, SP=CREATE OR ALTER, 데이터 보존).
- [ ] 배포 후 수동설정: OAuth 연결(Connect), Employee/EarningsRate 매핑, **Xero 포털 Pay Item 단가 RATEPERUNIT $1.00**(Travel/HighCare/BrokenShift — 앱이 금액 전송하므로 이중곱 방지).
- [ ] 메인 브랜치 머지(에릭 승인).
