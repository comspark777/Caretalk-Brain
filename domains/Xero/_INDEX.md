---
domain: Xero
updated: 2026-06-26
status: Track A·B·C·D + 배치/로우 삭제 + ABN 제외·행 다중삭제·UI 영문화 + 배치목록 기본2개월 + DEPLOY 29SP 통합(테스트DB 1:1 검증) — 프로덕션 SSMS·메인 머지 대기
---
# Xero

## 현재 상태
실무자 의견 매핑표 4트랙 **전부 코드 완료** (A·B·C 이번, D=Broken Shift 이전).
- **Track A**: Pay Item RateType 정합화(CSV 8줄 + 회계 가이드). 연차 메커니즘 기준 Casual=RATEPERUNIT / Part·Full근무=MULTIPLE. **앱 코드 무관**(Xero 설정값).
- **Track B**: Travel(KM) 금액을 CareTalk가 SiteConfig(`*_SUPPORTER_KM_CLAIM`) 단가로 계산해 전송(옵션2). 빌드·QA PASS.
- **Track C**: High Care 수당. 고객별 레벨 등록→배치 생성 시 자동채움(SOI.CustomerId JOIN)→레벨별 단가($3/$6) 금액 전송. 신규 테이블 2+SP 5, UI 2곳(admin/xero·Customer Detail). **QA 전체 PASS**(빌드·단위·E2E·디자인), 9커밋 미머지.
- **대기**: 프로덕션 SP(SSMS — Track C 6개 포함), **Xero 포털 Pay Item 정정 → RATEPERUNIT $1.00** (Travel $0.99→$1.00, **HighCare $3/$6→$1.00, BrokenShift FIXED→$1.00** — 코드가 금액 전송하므로 이중곱 방지), 라이브 대조, 메인 머지(에릭 승인).
- **그리드 UI(06-26)**: 배치행 HighCare 셀에 레벨+`$금액` 병기(`xeroPayroll.js`), BudgetClaim 'Send to Xero' 버튼 줄끝 이동·브랜드색. 라이브 확인.
- **ABN·삭제·영문화(06-26)**: Send to Xero에서 ABN 계약자(ContractType 2/13) EXCLUDED 마킹+ABN 시각배지/사유툴팁. 배치행 삭제 2종 추가(`Delete Excluded`=EXCLUDED 전부 / `Delete Selected`=체크박스 다중선택, 둘 다 UPLOADED 한정·소프트삭제). UI 한글→영문(Send to Xero 모달+SP statusMessage). **빌드 오류0, SP 3개(CREATE OR ALTER) SSMS 대기·미머지**.

## 세션 로그
<!-- /wiki-save 가 최신 로그를 [[YYYY-MM-DD-slug]] 형태로 여기에 추가 -->
- [[2026-06-25-track-ab-ratetype-travel]] — Track A(RateType 정합화) + Track B(Travel 금액 CareTalk 계산)
- [[2026-06-25-track-c-high-care]] — Track C(High Care 레벨별단가·고객별자동채움·금액전송, QA PASS·미머지)
- [[2026-06-26-uploaded-batch-row-delete]] — UPLOADED 배치/로우 소프트 삭제(SP가드·집계재계산·ISNULL하드닝, QA PASS·테스트DB적용)
- [[2026-06-26-high-care-followup-session-guard]] — High Care Settings탭 분리+레벨가드(0~2)+리뷰승인 / Admin 세션만료 가드(CareTalkClientController, qa PASS)
- [[2026-06-26-high-care-grid-amount-payitem-rate]] — High Care 그리드 금액병기(레벨+$) + Pay Item $1.00 정정(HighCare·BrokenShift, 이중곱방지) + Send to Xero 버튼 줄끝 정리(라이브 검증)
- [[2026-06-26-abn-exclude-row-delete-i18n]] — Send to Xero: ABN 계약자 제외(ContractType 2/13 EXCLUDED)+ABN 시각배지 / 행 다중삭제 2종(Delete Excluded·Selected 체크박스) / UI 영문화(모달+SP메시지). 빌드OK·SP 3개 SSMS 대기
- [[2026-06-26-batch-2month-default-deploy-merge]] — 배치목록 조회제한無 진단→기본 최근2개월 필터(batchId직접조회 미적용 함정회피, C#무변경) / 인덱스 점검=보강불필요 / DEPLOY_Full.sql 27→29 SP 통합(ABN·2개월·Delete 2종) / 테스트DB 실시간조회로 SP 29 1:1 검증·백업테이블 제외
