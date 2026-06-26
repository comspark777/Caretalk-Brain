---
domain: Xero
date: 2026-06-26
tags: [high-care, settings-tab, session-guard, code-review, auth]
status: done
related: ["[[_INDEX]]", "[[2026-06-25-track-c-high-care]]"]
---
# High Care 후속 — Settings 탭 분리 + 레벨가드 + 코드리뷰 + Admin 세션가드

Track C(High Care) 머지 전 후속 정리. UI 분리 + 리뷰 권고 반영 + 그 과정에서 발견된 로그인(세션) 에러 정석 해결까지. 전부 커밋, **미머지·미푸시**.

## 한 일
- **Settings 탭 분리** (`754f2ab9`): admin/xero `Index.cshtml`의 **Broken Shift Rates + High Care Allowance** 카드를 첫 탭(Batches/Send)에서 빼서 신규 **Settings 탭**(4번째)으로 이동. Batches/Send엔 CSV Upload가 맨 위로. JS 미변경(id 보존). Playwright 디자인 검증 PASS.
- **High Care 레벨 범위 가드** (`c1212019`): `XeroPayrollService.SaveHighCareCustomerAsync`에 `if(level<0||level>2)` 1줄. API 직접 호출로 레벨 3+ 들어오면 전송금액이 조용히 0 되는 것 차단(리뷰 권고 #1).
- **코드 리뷰** (`/review`): Track C 전체를 독립 code-reviewer가 체크리스트 10항목 검토 → **승인**(블로커 0, LOW 3: 레벨범위·UPSERT동시성·INSERT사문). 빌드 0오류, 단위테스트 13/13, 라이브 DB JOIN 정합 확인.
- **Admin 세션 만료 가드** (`00239e34`): `CareTalk.Web/Helpers/CareTalkClientController.cs:OnActionExecuting`. Admin Area + `userInfo==null`(쿠키valid+세션empty) → `RedirectToAction("LogOut","Account")`. qa-verifier Playwright PASS(세션만료→302 LogOut→로그인, 정상로그인 회귀, 루프없음, AJAX정상).

## 결정/맥락 (왜)
- **단가 관리 위치**: High Care 단가는 admin/xero(레벨별 회사공통 1세트), 고객 화면(Customer Detail)은 레벨만 지정. 단가는 자주 안 바꿔서 → Settings 탭으로 분리(매번 노출 불필요).
- **레이아웃 가드는 불가**(시도→FAIL→원복): `_Layout.cshtml`에서 `userInfo==null` 시 `return`하면 자식뷰 `@section Styles/Scripts` 미렌더 → `InvalidOperationException` 500. NullReference 500을 다른 500으로 바꿀 뿐. → **컨트롤러 단계 리다이렉트가 정석**(뷰 렌더 전).
- **세션 가드 설계**: 완전 미로그인은 `[Authorize]`+`LoginPath=/Account/loginV2`가 먼저 처리. 가드는 "인증쿠키 valid + 세션 만료" 엣지(앱 재시작/IIS Express 세션 초기화)만 잡음. `Account`(Area="")는 조건 미충족 → LogOut 무한루프 없음. `IUserSession`은 `RequestServices.GetService`로 해결(베이스 생성자 변경 시 파생 컨트롤러 27개 수정 회피).
- **c1212019에 다른 작업 혼입**: `git add` 파일전체로 "배치/로우 삭제" 작업의 `DeleteBatchAsync`/`DeleteBatchItemAsync`가 레벨가드 커밋에 섞임. 에릭 결정 **그대로 두기**(둘 다 같은 브랜치 필요 코드).

## 다음 할 일
- [ ] 메인(VersionUpGrade) 머지 — 전체 미머지·미푸시
- [ ] 프로덕션 SP SSMS 실행 (High Care 6개 + `20260626_Xero_DeleteBatch.sql`)
- [ ] "배치/로우 삭제" 작업 나머지 커밋 (IXeroRepository/XeroRepository/IXeroPayrollService/XeroController/xeroPayroll.js/SP — working tree 미커밋, Service 구현은 c1212019에 이미 포함)
- [ ] 루트 QA 스크린샷 png 4개 정리
