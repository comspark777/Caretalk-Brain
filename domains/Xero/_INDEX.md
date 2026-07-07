---
domain: Xero
updated: 2026-07-07
status: 멀티테넌트 라이브 + 관리자 매뉴얼 docx 완성(실서버 캡처 13장, bDocs). 실서버 배치 #9=unmapped 136·missing 39·period BLOCK 76. 남은 것=매칭보강 패치(승인대기)·회계 3건·PayItem $1.00×2조직·컷오버 합의 후 골든패스·접이식UI 머지
---
# Xero

## 최신 (2026-07-07, 2차)
**관리자 매뉴얼 docx + 검증 UI/요율 구조 진단** — [[2026-07-07-admin-manual-validation-diagnosis]]
- 매뉴얼: `CareTokSolutions/bDocs/CareTok_Xero_Admin_Manual_20260707.docx` (한글, 실서버 캡처 13장, ★처음 세팅 순서 포함). 실서버는 read-only로만 접근.
- 진단 확정: 접이식 카드 미표시 = 전 행 EXCLUDED → active 0건 → **0건 카드 숨김 설계**(버그 아님) / 요율은 조합 테이블 라이브 조인이라 탭만 채워도 반영되나 **직원 매핑 선행 필수**(R.TenantId=EM.TenantId) / ★Exclude 체크박스 = 체크가 "포함"(컬럼명과 반대).
- 실서버 배치 #9(01.07~12.07, 1,171행): Pending 898, 차단 unmapped 136·missing 39·period BLOCK 76(01~05.07 근무=열린기간 밖, 컷오버 이슈).

## 이전 (2026-07-07, 1차) — 다음 세션 시작점
**멀티테넌트(2조직) 개편 설계→구현→라이브 가동 완료** — [[2026-07-07-multitenant-two-org-live]] ★다음 할 일 체크리스트 이 로그에 있음
- 근본원인 확정: Xero 조직당 200명 제한 → A~J=ODC Payroll / K~Z=**ONE DREAM COMMUNITY PTY LTD("ODC"는 약칭)**. Auto-Map 41/234는 K~Z가 연결 조직에 없어서였음.
- 핵심 구조: EmployeeMapping.TenantId 진실원본 / 배치 tenant 불가지·전송 시 GroupBy / (BatchId,TenantId)당 PayRun 자식 테이블 / rotating refresh는 single-flight 갱신 후 전 행 upsert / SP 신규 파라미터 전부 기본값(배포창 호환).
- 라이브: 재연결 1회로 2조직 저장+Demo 자동정리 실증. 직원 87(45+42)·요율 31(23+8) 매핑. 요율 자동매핑 실패 2원인=ATTENDANT CARE 명세밖·ABP 중복이름(유일성 가드). Pay Period BLOCK=열린기간 밖 근무(컷오버: ~05.07 Deputy / 06.07~ CareTalk 제안).
- ★브랜치 사고 교훈: 작업 폴더 공유 시 에이전트 투입 전 브랜치·핵심 코드 grep 확인(구시점 분기 브랜치 체크아웃으로 접이식 UI가 구버전 위에 구현→리셋 후 재적용 d54aaf5a).

## 이전 (2026-07-02, 2차)
**실조직(ODC Payroll) 연결 완료 + 매핑 계획 확정** — [[2026-07-02-odc-connect-mapping-plan-2]] ★잔여 작업 체크리스트(A~D) 이 로그에 있음
- 권한: 앱 연결엔 Payroll admin + **Business and accounting=Standard** 둘 다 필요(전자만으론 "More permission required"). 권한 변경 후 Xero 재로그인 필수.
- 함정 2개 실전 확인: ①상태카드는 DB만 봐서 "Connected" 착시 가능(콜백 유실 시 반쪽연결 — SyncLog CONNECT로 판별) ②동의 화면 "Already connected" 상태에선 드롭다운 안 건드리고 Continue가 정답.
- Auto-Map: 요율 22건=Deputy 명세 F/P표와 일치(데모 잔재 0, DB검증). 미매핑 26조합 실측표+제안 매핑 로그에 기록.
- Pay Items 실사: 캐주얼 항목 기존재(생성 불필요, 매핑만). ⚠️Travel $0.99/HighCare $3.00/BrokenShift $20.82는 Deputy 방식 — **수정 금지, CareTalk 전용 $1.00 신규 3개 생성** 방향. ABP7/8 버전·ATTENDANT CARE·ContractType N은 회계 확인 3건.

## 이전 (2026-07-02, 1차)
OAuth 실서버 연결 + Area 잘림 정정 + 배치 그리드 먹통 해소 — [[2026-07-02-oauth-area-fix-grid-perf]]
- OAuth: 서버 env var(`setx XeroSettings__* /M`+iisreset) 주입, invalid redirect_uri=www 불일치→포털에 www URI 추가로 해결.
- Area: 원천 255자 vs 컬럼 VARCHAR(10) 잘림으로 핸드오버 실패 → `20260702_fix_xero_area_width.sql`(라이브 적용, **테스트 미적용 드리프트**) → 핸드오버 성공.
- 그리드: 3,317행 편집그리드가 메인스레드 30초+ 블로킹(실측) → 지연 셀렉트옵션 + 200행 클라이언트 페이징(`xeroPayroll.js`만) → 2.7초/전환 0.49초 실측 검증. 미커밋.

## 이전 (2026-07-01)
실서버 SSMS 배포 후, 배포파일이 테스트보다 구버전이던 **13개 SP 드리프트 발견 → 테스트 기준으로 재정정**.
- 정규화 해시(공백제거+CREATE동사통일+대괄호제거+SHA256/UTF-16LE) 서버대서버 대조. 테스트DB SP는 `CREATE PROCEDURE`(평문) 저장이라 초기 오탐 → 동사통일로 해결.
- 배포 직후: 테이블 9 + SP 16 일치 / **SP 13 상이**(=파일 stale 집합). 누락/추가 0.
- `CareTalkData/20260701_sync_xero_13sp_test_to_live.sql`(테스트 현재정의 hex 무손실 추출→`CREATE OR ALTER`) 적용 → **38/38 완전 일치(0 diff)** 확정.
- 교훈: CREATE OR ALTER 배포는 파일이 최신 아니면 옛 로직 덮어씀. 로그의 "1:1 일치" 기록보다 실측 해시가 권위.

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
- [[2026-07-01-prod-deploy-13sp-sync]] — 실서버 배포(DEPLOY 29SP) / 배포파일 stale로 13SP 드리프트 발견(정규화 해시 서버대서버 대조, 평문 CREATE PROCEDURE 오탐 회피) / 테스트 현재정의 hex 무손실 추출→`20260701_sync_xero_13sp_test_to_live.sql`로 정정 / 재대조 38/38 0diff 검증
- [[2026-07-02-oauth-area-fix-grid-perf]] — OAuth 실서버 연결(env var 주입·www redirect_uri 해결·★Demo Company 연결됨) / Area VARCHAR(10) 잘림 정정(`20260702_fix_xero_area_width.sql`, 라이브만 적용·테스트 드리프트) / 배치그리드 30초+ 프리즈 실측→지연옵션+200행 페이징으로 2.7초(라이브 검증) / 배치#2 삭제=에릭 추정
- [[2026-07-02-odc-connect-mapping-plan-2]] — 실조직 ODC Payroll 연결(Standard 역할 필요 발견·반쪽연결 진단·CONNECT 16:59 DB확인) / Auto-Map 요율 22(명세 F/P 일치)·직원 41 / Pay Items 실사(캐주얼 기존재·수당 3종 $0.99·$3·$20.82=Deputy 방식→$1.00 신규 생성 방향) / 미매핑 26조합 실측표 / ★잔여 작업 체크리스트 A~D
- [[2026-07-07-multitenant-two-org-live]] — 멀티테넌트(2조직) 개편 전체 사이클: 200명 제한 근본원인 확정(K~Z=ONE DREAM COMMUNITY) / 아키텍처 결정 9(TenantId 진실원본·배치 불가지·PayRun 자식테이블·single-flight refresh) / SQL 2본+4레이어 구현+QA·verify·simplify / 라이브 2조직 연결·Demo 자동정리·직원 87·요율 31 / 운영지식(요율 스킵 2원인·Pay Period 컷오버·미매핑 165 분류) / 접이식 UI+브랜치 사고 교훈
- [[2026-07-07-admin-manual-validation-diagnosis]] — 관리자 매뉴얼 docx(실서버 캡처 13장, 처음 세팅 순서 포함, bDocs) / 접이식 카드 미표시=0건 숨김 설계 확인 / 요율=조합 테이블 라이브 조인·직원 매핑 선행(R.TenantId=EM.TenantId) / Exclude 체크=포함 주의 / 실서버 배치 #9 차단 현황(136·39·76) / Playwright 실서버 캡처 팁(스크롤 고정 우회·탭 복원·LogIn 경로)
