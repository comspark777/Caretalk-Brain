---
domain: Xero
date: 2026-07-02
tags: [xero, odc-payroll, real-org, standard-role, automap, pay-items, deputy-spec, mapping-plan, remaining-tasks]
status: 실조직(ODC Payroll) 연결 완료 + Auto-Map 22/48·41/234 — 매핑 마무리·$1.00 항목 생성·전송 테스트 대기
related: ["[[_INDEX]]", "[[2026-07-02-oauth-area-fix-grid-perf]]"]
---
# 실조직(ODC Payroll) 전환 완료 + 매핑 현황과 잔여 작업 총정리

Demo → 실조직 전환의 권한 이슈를 풀고 연결 완료. Auto-Map 1차 실행 후 Pay Items 실사와 Deputy 명세 대조로 **남은 매핑 작업 계획을 확정**. 다음 세션은 아래 "잔여 작업" 섹션부터 시작하면 됨.

## 한 일

### 1) 실조직 전환 — 권한 2단계가 관건이었다
- **Payroll admin만으론 앱 연결 불가.** Xero 정책: 앱 연결(authorise)엔 ① Payroll admin(payroll.* 스코프) + ② **Business and accounting 역할 Standard 이상** 둘 다 필요. ②가 빠져 동의 화면에서 "More permission required"(회색) 지속 → 관리자에게 Standard 요청해 해결. (권한 변경 후 Xero 로그아웃/재로그인 필요 — 세션에 옛 역할 캐시)
- **반쪽 연결 사고**: 동의는 성사됐는데 CareTalk 세션 만료로 콜백 유실 → Xero엔 연결 존재 / DB엔 데모 죽은 토큰. 상태 카드는 [DB만 확인](XeroAuthService.cs GetConnectionStatusAsync)해서 "Connected" **착시** — DB(SyncLog CONNECT 부재)로 판별했음. CareTalk 로그인 상태에서 Reconnect 재실행으로 해결.
- 동의 화면에서 ODC Payroll이 "Already connected"일 때 **드롭다운 안 건드리고 Continue with 1 organisation** → 연결이 1개뿐이라 `GetFirstConnectionAsync`(첫 연결 저장)가 안전하게 ODC Payroll 저장.
- **최종 확인(DB)**: SyncLog `CONNECT SUCCESS 16:59:12 — ODC Payroll`. 토큰 테이블에 데모 옛 행(TokenId 1) 잔존하나 최신 행을 읽어 무해(선택적 정리 대상).

### 2) Auto-Map 1차 (16:59:32) — 직원 41/234, 요율 22/48
- 요율 22건 전부 ODC Payroll 연결 "후" 생성 — 데모 잔재 0 (DB 타임스탬프 검증).
- 22건은 **Deputy 명세 §3-2/3-3(F/P 매핑표)과 정확 일치** — 실조직 Pay Item 이름이 Deputy 시절 그대로라 이름 매칭 성공.
- 직원 41명만 매칭 = 이메일 불일치 다수 추정(CareTalk 개인메일 vs Xero 근무메일). 잔여 193명은 수동 매핑 또는 Xero 직원 미등록 확인 필요.

### 3) ODC Payroll Pay Items 실사 (Payroll → Payroll settings → Pay Items → Earnings)
- Deputy 시절 항목 풍부: `Casual - Daytime(1.25x)`~`Public Holiday (2.75x)`, `Aged care-casual ...` 시리즈, `Nursing Casual/F/P ...` 시리즈, `Inactive Sleepover - Weekday/Saturday/Sunday/Public Holiday`, `F/P - ...`(배수형) — **캐주얼 항목 신규 생성 불필요, 매핑만 하면 됨**.
- ⚠️ 같은 이름의 **ABP7/ABP8/ABP8-12 버전 중복** 다수 → 어느 버전이 현행인지 회계 확인 필요.
- ⚠️ **수당 3종 단가가 Deputy 방식**: `Travel Allowance (per KM)` **$0.99**/unit, `High Care Allowance` **$3.00**/unit, `First Broken Shift` **$20.82**/unit. 우리 코드(Track B/C, 6/25 설계변경)는 **금액을 units로 전송** → $1.00 항목 필요. **기존 항목 단가 수정 금지**(현행 Deputy 급여 흐름이 사용 중) → **CareTalk 전용 신규 항목 3개($1.00) 생성**이 결정 방향.

### 4) Deputy 스와핑 명세 대조 (`CareTalkData/Xero_Deputy_Mapping_Spec.md`, 6/17)
- **적용 확인**: 동일 원천(#src=ClaimExcel), 근무+수당 행 분해→4단계 전송 구조, F/P 매핑표(22건 일치).
- **명세 밖 신규 케이스** (배치 #5 실측): ① **ATTENDANT CARE** area(~480행 — 명세는 NDIS/AGED만) ② **ContractType N**(~92행 — 명세는 C/P/F만, Nursing 추정) ③ PayType 표기 `SOHOUR/SOSAT/SOSUN`(명세는 SO/...).
- 매핑 지식: 매핑은 **전역 테이블**(배치 무관) — 한 번 등록하면 이후 모든 배치에 자동 적용. 재작업은 신규 인원/신규 조합/Xero측 변경 시에만.

### 5) 배치 #5 미매핑 26조합 실측 (행수 포함)
| 조합 | 행수 | 제안 매핑 (명세 §3-1) |
|---|---|---|
| HOUR/NDIS/C | 596 | Casual - Daytime(1.25x) |
| HOUR/AGED/C | 459 | Aged care-casual daytime |
| SAT/NDIS/C | 66 | Casual - Saturday (1.75x) |
| SUN/NDIS/C | 39 | Casual - Sunday (2.25x) |
| NOON/NDIS/C | 32 | Casual - Afternoon (1.38x) |
| NIGHT/NDIS/C | 28 | Casual - Night (1.40x) |
| SAT/AGED/C | 27 | Aged care-casual saturday |
| SUN/AGED/C | 18 | Aged care-casual sunday |
| SOHOUR/*(C·F) 16 + SOSAT/NDIS/F 2 + SOSUN/* 4 | 22 | Inactive Sleepover - Weekday/Saturday/Sunday |
| **ATTENDANT CARE/C** (HOUR 234·NOON 93·SAT 69·SUN 56·NIGHT 23·SO* 6) | ~481 | ❓ 회계 확인 (Casual 일반 시리즈 폴백?) |
| **ContractType N** (HOUR/NDIS 47·HOUR/AGED 25·SAT/NDIS 10·기타 10) | ~92 | ❓ N=Nursing? → Nursing 시리즈? |

## 결정/맥락 (왜)
- **기존 Deputy 항목 단가 불변경**: Deputy 병행 기간 중 단가 변경은 현행 급여를 깨뜨림 → CareTalk 전용 신규 $1.00 항목으로 분리. Deputy 컷오버 후 정리.
- **매핑은 회계와 함께**: ABP 버전 선택·ATTENDANT CARE·N 의미는 급여 분류 결정이라 개발 판단 금지.
- 상태 카드 착시·GetFirstConnectionAsync 첫연결 저장은 **개선 후보**로 기록만 (지금은 운영 우선).

## 다음 할 일 (다음 세션 시작점)

### A. Xero 포털 (에릭, Payroll admin 권한으로 직접 가능)
- [ ] **CareTalk 전용 Pay Item 3개 생성** — Payroll settings → Pay Items → Earnings → Add, 전부 `Rate per unit $1.00`: ①Travel(KM)용 ②High Care용 ③Broken Shift용 (이름에 CareTalk 등 구분자 권장). **기존 $0.99/$3.00/$20.82 항목 수정 금지.**
- [ ] Calendars 탭에서 급여 캘린더(격주) 존재 확인

### B. 회계 담당 확인 (질문 3개)
- [ ] ABP7/ABP8 중복 항목 중 **현행 버전**은?
- [ ] **ATTENDANT CARE** 캐주얼 근무 → 어느 Pay Item? (Casual 일반 시리즈로 대체 가능?)
- [ ] **ContractType N**의 의미? (Nursing이면 Nursing Casual/F/P 시리즈 매핑?)

### C. CareTalk /Admin/Xero (매핑 마무리)
- [ ] 요율 매핑 **~19조합 수동 Save** (위 표의 확정분 — Earnings Rate Mapping 탭 드롭다운)
- [ ] 신규 $1.00 항목 생성 후 Travel/HighCare/BrokenShift 매핑
- [ ] 회계 답변 후 ATTENDANT CARE·N 조합 매핑
- [ ] **직원 잔여 193명**: Auto-Map 재실행 → 잔여는 이메일 대조 후 수동 매핑 (Xero 미등록 직원은 회계에 등록 요청)
- [ ] 검증 통과(Missing 0) → **소량 기간 배치로 전송 4단계 골든패스** (POSTED 전 금액 대조: Travel/HighCare/BrokenShift 이중곱 여부 필히 확인)

### D. 개발 잔여 (코드/DB)
- [ ] `xeroPayroll.js` 성능 수정(지연옵션+200행 페이징) **커밋 + 메인 머지** (에릭 승인) — 실서버엔 이미 배포됨
- [ ] **테스트 서버에 `20260702_fix_xero_area_width.sql` 실행** (Area 드리프트 해소, 미실행 시 테스트 핸드오버 잘림 재현)
- [ ] (선택) 데모 토큰 잔존행 정리 / 상태카드 Xero 실검증 / GetFirstConnectionAsync 조직선택 로직 — 개선 후보
