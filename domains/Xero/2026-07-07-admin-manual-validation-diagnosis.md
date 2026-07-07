---
domain: Xero
date: 2026-07-07
tags: [xero, manual, docx, validation, collapse-ui, rate-mapping, exclude-checkbox, playwright, live-capture]
status: 관리자 매뉴얼 docx 완성(실서버 캡처 13장) + 접이식 카드 미표시=0건 숨김 설계 확인 + 요율매핑 구조(라이브 조인) 문서화
related: ["[[_INDEX]]", "[[2026-07-07-multitenant-two-org-live]]"]
---
# Xero 관리자 매뉴얼 작성 + 검증 UI/요율 매핑 구조 진단 (07.07, 2차)

## 한 일

### 1) "접이식 검증 카드가 안 보인다" 진단 — 버그 아님, 설계 동작
- 에릭 문의: 배치 #8(전 행 EXCLUDED)에서 어제 만든 접이식 카드가 안 보임.
- 원인 체인 확정: 검증 4종(Supporter Summary/Unmapped/Missing Rates/Period Warnings)은 전부 `active = PENDING+SKIPPED` 행 기준으로 계산(`XeroPayrollService.cs:295,302,342,364`) → 전 행 EXCLUDED면 4종 모두 0건 → **0건 카드 숨김 규칙**(`xeroPayroll.js:697-704,767,803`)으로 전부 숨음 → 남는 건 `BlockReasons`의 "no rows available to send" 한 줄뿐(`XeroPayrollService.cs:435-436`).
- 전 행 EXCLUDED의 유력 원인 = 업로드 자동 제외 규칙(이미 SENT/LOCKED 중복, ABN 2/13, 스케줄 미잠금 `SOI.isDisable≠1` 등). 행 ⓘ 툴팁(StatusMessage)으로 확인 가능.

### 2) 요율 매핑 구조 확인 — "탭만 채워도 되나?" → 된다(전제 2개)
- 행 그리드의 Xero Earnings Rate는 저장값이 아니라 **조회 시마다 매핑 테이블 라이브 조인**으로 계산: `usp_CARETALK_XER_Select_UploadBatchItems`의 OUTER APPLY(`20260706_xero_multitenant_sp.sql:353-368`).
- 행 인라인 요율 드롭다운도 행 오버라이드가 아니라 **같은 조합 테이블에 저장하는 다른 입구**(`xeroPayroll.js:1348` → SaveEarningsRateMapping). 전송 시 해석도 동일 테이블(`ResolveRate`, 정확→Area공통→Type공통→완전공통).
- ★전제: ①조인 조건 `R.TenantId = EM.TenantId` — **직원 미매핑 행은 tenant 미정이라 요율도 영원히 Unmapped**(직원 매핑 선행 필수) ②해당 조직에 조합 실존(공통 매핑 폴백 가능).

### 3) 관리자 매뉴얼 docx 작성 (에릭 요청)
- 산출물: `CareTokSolutions/bDocs/CareTok_Xero_Admin_Manual_20260707.docx` (1.39MB, python-docx, 한글).
- **실서버 caretok24.com 화면 기준** 스크린샷 13장 임베드(에릭 지시로 테스트서버→실서버 전환). 조회/탭 전환만, 변경행위 0.
- 구성: 개요(2조직 구조) / ★처음 세팅 순서(연결→Settings 단가→직원매핑→요율매핑→배치→전송, 순서 이유 포함) / 탭 4개 상세 / 검증 4종 원인→해결 표 / 4단계 전송 / 행 그리드 17컬럼 표 / FAQ 11 / 용어집 17.
- 문서화한 함정: **Exclude 체크박스는 체크=포함(PENDING)/해제=제외(EXCLUDED)**(`xeroPayroll.js:1088,1113` — 컬럼명과 반대로 읽힘), Supporter Summary의 Area/Type=첫 행 대표값, 0건 카드 숨김.

### 4) 실서버 관찰 (07.07 시점)
- 연결: 2조직 정상. 배치 목록(기본 2개월 필터)엔 **#9 (01.07~12.07, 1,171행)** 1건.
- 배치 #9: Pending 898 / Excluded 273. 차단 = unmapped 136 / missing rate 조합 39 / period BLOCK 76(전원 근무 01.07~05.07 = 열린기간 06.07~19.07 밖 → 컷오버 이슈 그대로), 212 supporters.
- EXCLUDED 273행 다수 = ABN 배지 행(Org "—").

## 결정/맥락 (왜)
- 매뉴얼을 실서버 기준으로 재캡처: 에릭이 실제로 보는 화면과 동일해야 매뉴얼 가치가 있음(테스트서버 진행 중 중단 지시).
- Playwright 실서버 작업 시 **read-only 원칙**(선택/탭/접기펼치기/필터만, Save·Delete·Send·Auto-Map 금지).

## 운영/기술 팁 (재사용)
- caretok24.com admin 템플릿은 window 스크롤 프로그램 제어가 안 먹음(scrollTop 고정, zoom 추정) → 긴 그리드 캡처는 **행 trim(display:none) 후 element screenshot** 우회.
- admin 템플릿이 마지막 활성 탭을 복원함(localStorage) → 자동화 시 탭 상태를 가정하지 말고 명시 클릭.
- 로그인은 `/Account/LogIn`(구) 경로가 자동화에 안정적, loginV2는 로딩 오버레이가 클릭 가로챔. 로그인 완료까지 10초+ 걸릴 수 있음.

## 다음 할 일
- [ ] 에릭 매뉴얼 검토 → 컷오버 정책 확정되면 Pay Period 섹션 갱신
- [ ] 배치 #9 실전송 전: unmapped 136(매칭보강 패치 승인 대기와 연동)·missing 39 조합 해소
- [ ] (승계) 1차 로그의 체크리스트: 매칭보강 패치·회계 3건·PayItem $1.00×2조직·컷오버 합의 후 골든패스·접이식UI 머지
