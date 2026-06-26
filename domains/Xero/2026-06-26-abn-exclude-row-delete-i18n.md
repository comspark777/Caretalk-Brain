---
domain: Xero
date: 2026-06-26
tags: [xero, abn, exclude, batch-delete, multi-delete, i18n, budgetclaim, send-to-xero]
status: 코드 완료 (SP 3개 SSMS 대기 · 미머지)
related: ["[[_INDEX]]", "[[2026-06-26-uploaded-batch-row-delete]]"]
---
# Send to Xero — ABN 제외 + 행 다중삭제 + UI 영문화

BudgetClaim **Send to Xero**(`btnXeroHandover` → `HandoverToXero` → `CreateBatchFromBudgetClaimAsync` → SP `usp_CARETALK_XER_Insert_UploadBatch`) 후속 3종 작업.

## 한 일
- **Send to Xero 동작 확인**: 화면 날짜(`#sDate`~`#eDate`) + Member Option(FundType 0/479/478)을 `{sdate,edate,fundType}`로 POST → 배치 생성 → `/Admin/Xero?batchId=N` 이동. (`BudgetClaim.js:2098` handoverToXero)
- **ABN 계약자 제외**: Insert SP RowStatus/StatusMessage CASE에 `SI.ContractType IN (2,13)` → `'EXCLUDED'` 추가. ContractType은 `CARETALK_Users_SupporterInfos`(SP가 이미 `SI` JOIN 중, 신규 JOIN 없음). 우선순위: 이미전송(SENT/LOCKED) > ABN(2/13) > 미잠금 > PENDING.
- **ABN 시각 구분**(프론트): `xeroPayroll.js` `buildEditableItemRow` Status 셀에 `excludeReasonMarkup(it)` — `statusMessage`에 'ABN' 포함 시 빨강 `ABN` 배지, 그 외 EXCLUDED는 info 아이콘 + 전체 사유 title 툴팁. (편집모드에서 사유 안 보이던 한계 해결)
- **EXCLUDED 일괄삭제**(`Delete Excluded` 버튼): SP `usp_CARETALK_XER_Delete_ExcludedBatchItems` + Repo/Service/Controller(`DeleteExcludedItems`, `BatchIdRequest` 재사용) + `deleteExcludedItems()`.
- **선택 다중삭제**(`Delete Selected` 체크박스): SP `usp_CARETALK_XER_Delete_BatchItems`(itemIds CSV, `STRING_SPLIT`, 호환성 160) + 풀 파이프라인 + 헤더 전체선택(`#chkSelectAllRows`)·행 체크박스(`.xero-row-select`) + `deleteSelectedItems()`. 두 삭제 버튼 공존.
- **한글 UI → 영문화**: Send to Xero 모달(`BudgetClaim.js` handoverToXero: title/html/버튼/실패·오류 알럿) + Insert SP statusMessage 3종(ABN/미잠금/중복). 'ABN' 단어 유지로 프론트 판정 보존. (xeroPayroll.js Swal은 이미 전부 영문 — console 한글만 잔존, 화면 무관)
- **빌드**: `dotnet build -t:Compile` 오류 0 (일반 빌드의 MSB3027/3021은 IIS Express DLL 잠금일 뿐 코드 무관).
- **SQL 재실행 안전화**: "이미 존재" 에러(세 SP 모두 DB 존재 확인) → 3파일 모두 `CREATE OR ALTER`로 통일.

## 결정/맥락 (왜)
- **ABN = ContractType 2(Sub-Contractor)+13(Third party Contractor)**. 화면 라벨 `1647=ABN`(Available Job)은 **실데이터 0건**이라 미채택. 실분포: 0=1248,1=534,2=68,11=93,12=44,13=23. 배정 SOI: 2→1061건/13명(효과 실재), 13→0건(미래 대비 포함). TFN(0/1/11/12)=직원=payroll, ABN(2/13)=계약자=인보이스 정산.
- **제외 방식 = EXCLUDED 마킹**(행 유지+사유). 강제차단 아님 — Exclude 체크박스로 수동 재포함 가능(한계②). 시각 `ABN` 배지로 인지하도록 두는 게 의도.
- **삭제 2종 분리**: `Delete Excluded`=EXCLUDED 전부 / `Delete Selected`=체크 선택. 둘 다 UPLOADED 배치 한정(SP 가드), 소프트삭제+집계 재계산.
- **SP 적용은 에릭님 직접**(전체 도메인) — [[../../HOME]] 규칙. Claude는 스크립트 파일만.

## 다음 할 일
- [ ] SP 3개 SSMS 실행(`CREATE OR ALTER`): `..._Insert_UploadBatch_ExcludeABN.sql`(ABN+영문메시지) / `..._Delete_ExcludedBatchItems.sql` / `..._Delete_BatchItems.sql`
- [ ] F5 재실행 후 화면 확인: ABN 배지·사유 툴팁, Delete Excluded/Selected 동작, 영문 모달/메시지
- [ ] 라이브 검증 후 메인 머지(에릭 승인)
