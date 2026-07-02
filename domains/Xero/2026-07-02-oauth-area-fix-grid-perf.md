---
domain: Xero
date: 2026-07-02
tags: [xero, oauth, redirect-uri, env-var, area-truncation, grid-performance, paging, lazy-options, playwright-measure, demo-company]
status: OAuth 연결·Area 정정·그리드 페이징 실측검증 완료 — 실조직 전환·매핑·머지 대기
related: ["[[_INDEX]]", "[[2026-07-01-prod-deploy-13sp-sync]]"]
---
# OAuth 실서버 연결 + Area 잘림 정정 + 배치 그리드 먹통 해소(페이징)

실서버 Xero 실가동 준비 3종: OAuth 연결 성공, 핸드오버 Area 잘림 수정, 3,317행 배치 그리드 30초+ 프리즈를 실측으로 잡아 페이징으로 해소.

## 한 일

### 1) OAuth 실서버 설정 + invalid redirect_uri 해결
- 키 주입 구조: 로컬=`dotnet user-secrets`(**Development에서만 로드**, csproj `UserSecretsId` 존재), 서버=**머신 env var** `setx XeroSettings__ClientId/__ClientSecret/__RedirectUri /M` + `iisreset`(env var가 appsettings 덮음, `Host.CreateDefaultBuilder` 로드 순서). 실서버 appsettings 직접 수정 불필요.
- Connect 시 Xero 에러 `invalid_request: Invalid redirect_uri` → **포털 등록은 non-www**(`caretok24.com`), env var는 **www** 불일치가 원인. 포털에 `https://www.caretok24.com/Admin/Xero/Callback` + 테스트용 www.onedreamrs.com 추가로 해결(포털 저장 즉시 반영).
- Connect→동의→콜백 성공. 연결은 앱 단위 1회(토큰 DB저장+offline_access 자동갱신) — 관리자마다 로그인 불필요. 60일 미사용 시에만 재연결.
- ⚠️ **라이브가 Demo Company (AU)에 연결됨** — 실조직 전환 필요. [XeroAuthService.cs:69](CareTalk.Services/XeroAuthService.cs#L69) `GetFirstConnectionAsync`가 **첫 번째 연결을 무조건 저장**하므로 반드시 **Demo 연결 해제(Xero Connected apps) → Reconnect(실조직 선택)** 순서. 조직 전환 시 직원/요율 매핑 GUID 전부 무효 → 재등록 + 실조직 Pay Item($1.00) 생성 필요.

### 2) 핸드오버 Area 잘림 정정 (실서버 배포 후 첫 실데이터 오류)
- Send to Xero 실패: `UploadBatchItem 열 'Area' ... 잘립니다. 잘린 값: 'ATTENDANT '`. 원천(#src/ClaimExcel)=NVARCHAR(255) vs 대상 컬럼=**VARCHAR(10)**. 테스트 데이터는 AGED/NDIS(4자)라 미발현 — 실서버 실데이터('ATTENDANT CARE' 등)에서만 터짐.
- `CareTalkData/20260702_fix_xero_area_width.sql`: UploadBatchItem.Area / EarningsRateMapping.Area(UX_Xero_RateMap_Combo DROP→재생성) → NVARCHAR(255) + SP 2종(`Upsert_EarningsRateMapping` @I_Area 조용한잘림, `Insert_UploadBatch_FromJson` $.area) CREATE OR ALTER. DEPLOY_Full 4곳도 패치.
- 라이브 적용 → 핸드오버 성공(배치 #3, 3,317행). ⚠️ **테스트 서버 미적용 드리프트 발견**(Area 아직 varchar(10)) — 실행 필요.
- 인덱스 서버대서버 검증: 라이브 16/16 테스트와 구조 일치(자동생성 PK 이름 차이만, 무해). sys 카탈로그 문자열 결합 시 **COLLATE DATABASE_DEFAULT** 필요(Latin1 vs Korean_Wansung 충돌).

### 3) 배치 그리드 먹통("사이트 전체 느려짐") — 실측 → 2단 조치
- Playwright 라이브 실측: Xero API 전부 <1.1s(캘린더 0.45~1.07s, 직원 0.45s), BatchPreview 0.8s, BatchItems 2MB/0.76s — **서버·API 무죄**. 병목 = 3,317행 편집그리드 DOM: 행당 폼컨트롤 7개 + **행마다 직원/요율 셀렉트 전체옵션 인라인** + 이중 렌더(`loadBatchPreview` 캐시 전후 2회) → **메인스레드 30초+ 블로킹 실측**(evaluate조차 실행불가). 같은 사이트 탭들이 렌더러 프로세스 공유 → "케어톡 전체가 느려짐"으로 체감.
- 조치(모두 `xeroPayroll.js`만, 서버/SP/뷰 무변경):
  ① **지연 옵션**: 셀렉트에 선택옵션 1개만 렌더, mousedown/focusin 시 전체 채움(`lazySelectMarkup`/`fillLazyOptions`, 플래그는 .attr — .data 캐싱 회피)
  ② **클라이언트 페이징 200행/페이지**: `state.batchItems` 보관+현재 페이지만 렌더(`renderBatchItemsPage`/`renderItemsPager`, 페이저 컨테이너 JS 동적 생성으로 cshtml 무변경). 오픈/검색=1페이지 리셋, 편집 후 새로고침=페이지 유지.
- 배포 후 실측 검증: 열기 ~2.7s(프리즈 0), 페이지 전환 0.49s, 셀렉트 옵션 1→16 지연 채움 확인. 캐시버스터가 요청마다 랜덤이라 파일 복사만으로 즉시 반영.

### 4) 기타
- 배치 #2(3,308행) 소프트삭제 확인: 13:39:59 bebe@naver.com — 자동화 클릭(13:40:28~) 이전이라 에릭 본인 삭제로 추정(미확인). IsDelete=1 복구 가능.
- 라이브 로그인 자동화: loginV2 페이지(`/Account/loginV2`), 네이티브 fill+click 후 **네비게이션 없이 대기**해야 성공(즉시 이동하면 로그인 AJAX 끊김).

## 결정/맥락 (왜)
- **서버 env var 방식**: 재배포로 appsettings가 덮여도 생존, 비밀값 소스 미노출. user-secrets는 Development 전용이라 서버에서 무의미.
- **페이징(서버 아닌 클라이언트)**: 데이터 전송(2MB/0.76s)은 문제없고 DOM만 병목 → SP/API 무변경으로 최소 수정. 200행=페이지당 폼컨트롤 1.4천개 수준.
- **측정 우선**: "매핑/Xero API가 느린가?" 가설을 실측으로 배제(전부 <1.1s) 후 진짜 병목(DOM)에만 손댐.

## 다음 할 일
- [ ] 테스트 서버 SSMS: `20260702_fix_xero_area_width.sql` 실행(드리프트 해소)
- [ ] Demo Company 연결 해제 → 실조직 Reconnect → Tenant 표시 확인
- [ ] 실조직 Pay Item 생성(Travel/HighCare/BrokenShift = RATEPERUNIT $1.00) + Auto-Map/요율 매핑
- [ ] 소량 기간 배치로 전송 4단계 골든패스 검증
- [ ] `xeroPayroll.js` 성능 수정 커밋 + 메인 머지(에릭 승인)
