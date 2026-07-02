---
domain: Xero
date: 2026-07-01
tags: [xero, deploy, production, sp-drift, sync, sqlcmd, hash-compare, ssms]
status: 실서버 배포 + 13SP 동기화 완료 · 서버대서버 0 diff 검증
related: ["[[_INDEX]]", "[[2026-06-26-batch-2month-default-deploy-merge]]", "[[2026-06-26-abn-exclude-row-delete-i18n]]"]
---
# Xero 프로덕션 배포 + 13개 SP 테스트기준 동기화 (드리프트 정정)

`/Admin/Xero` (PayRun) 기능을 실서버에 배포. 배포 파일이 테스트보다 구버전이던 13개 SP를 테스트 기준으로 재정정하고 서버대서버로 0 diff 검증.

## 한 일

### 1) DEPLOY_20260626_Xero_Full.sql 실서버 SSMS 배포 (에릭 실행)
- `CareTalkData/DEPLOY_20260626_Xero_Full.sql` (테이블 9 + SP 29, `CREATE OR ALTER`, 멱등).
- PART0 사전검증 통과(외부의존 `usp_CARETALK_PM_select_ClaimExcel`·`uf_CARETAIL_GET_Split_Index` 등 실존) → 객체 전부 생성/갱신됨.

### 2) 배포 전 대조에서 드리프트 발견 — 파일 vs 테스트DB (16 일치 / 13 불일치)
- 정규화 해시 대조법 확립: **공백 전부 제거 → `CREATEORALTERPROCEDURE`↔`CREATEPROCEDURE` 통일 → 대괄호 제거 → SHA256(UTF-16LE)**. 주석/들여쓰기/공백/CREATE동사 차이를 중화하고 본문 로직만 비교.
- 함정: 테스트DB SP는 **`CREATE PROCEDURE`(평문)로 저장**, 배포파일은 `CREATE OR ALTER`. 초기엔 컷 실패로 29개 전부 DIFF 오탐 → 동사 통일 정규화로 해결.
- 결과: 16개 본문 동일, **13개 상이**. 즉 배포파일이 테스트보다 이 13개에서 구버전.

### 3) 배포 후 실서버 vs 테스트(기준) 서버대서버 대조 → 13개 diff 확인
- `d:/tmp/inv.sql`: XER SP(정규화 해시) + Xero 테이블(컬럼시그니처 해시) 인벤토리. 양쪽 sqlcmd 서버사이드 실행(실서버 SELECT 전용).
- 실서버 = 배포파일 그대로가 되어, **테이블 9 전부 일치 + SP 16 일치 / SP 13 상이**(파일-vs-테스트 13과 정확히 동일 집합).
- 누락/추가 객체 0. 문제는 오직 13개 SP가 실서버에서 구로직.

### 4) 테스트 기준 동기화 스크립트 생성 + 검증 + 재적용
- 테스트 서버의 현재 13개 정의를 **hex 청크 방식**(UTF-16LE→hex 7000자 청크 다중행 → Python 재조립·디코드)으로 무손실 추출. (bcp·sqlcmd `-u`는 8191자에서 절단 → 우회)
- `CareTalkData/20260701_sync_xero_13sp_test_to_live.sql`: 원본 헤더/본문 유지, 헤더만 `CREATE PROCEDURE`→`CREATE OR ALTER PROCEDURE`. SET옵션 상단.
- 로컬 이중검증: 13개 전부 **테스트 기준 해시와 MATCH** + 13개 전부 현재 실서버와 diff.
- 에릭 SSMS 실행 후 재대조: **실서버 vs 테스트 38/38 완전 일치(0 diff)** 확정.

## 결정/맥락 (왜)
- **기준 = 테스트 서버**(에릭 지시). 배포파일이 stale이라 실서버가 구버전이 됐고, 테스트 현재본으로 되돌림.
- **CREATE OR ALTER 배포의 함정**: 파일이 최신이 아니면 실서버에 옛 로직을 덮어씀. 06-26 오후 테스트에서 손본 `Insert_UploadBatch`(13:06)·`Select_UploadBatches`(13:04)·`Delete_*` 등이 파일에 미반영이었음.
- **정규화 해시가 로그 신뢰보다 우선**: 06-26 로그의 "테스트DB 29개 1:1 일치"는 그 시점 기준이었고, 이후 테스트가 갱신돼 지금은 파일과 어긋남 → 실측이 권위.
- **추출 무손실 확보**: 구버전 bcp/sqlcmd 8191자 절단 이슈를 hex 청크로 우회(긴 SP: Insert_UploadBatch 8368자, FromJson 12258자).

## 다음 할 일
- [ ] Xero 포털 수동설정: OAuth Connect / Employee·EarningsRate 매핑 / **Pay Item 단가 RATEPERUNIT $1.00**(Travel·HighCare·BrokenShift, 이중곱 방지).
- [ ] 라이브에서 실제 배치 생성→Send to Xero 골든패스 화면 검증(ABN 제외 배지·Delete Excluded/Selected·영문 모달).
- [ ] 메인 브랜치 머지(에릭 승인).
