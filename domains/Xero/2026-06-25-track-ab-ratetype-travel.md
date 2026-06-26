---
domain: Xero
date: 2026-06-25
tags: [payroll, ratetype, travel, payitem, siteconfig, pipeline]
status: done
related: ["[[_INDEX]]"]
---
# Xero PayItem RateType 정합화(Track A) + Travel 금액 CareTalk 계산(Track B)

실무자 의견(원본 매핑표 이미지)을 4트랙 A/B/C/D로 분해. Track D(Broken Shift)는 이전 세션 완료. 이번 세션은 **Track A·B 완료**.

## 한 일

### Track A — Xero Pay Item RateType 정합화 (앱 코드 무관, 명세 + 가이드)
- **핵심 발견:** RateType은 CareTalk 앱 코드/DB에 없음. `Xero_PayItems_List.csv`의 RateType/Value는 이전 세션이 PayItem 이름에서 추론한 파생 데이터. 실제 RateType은 Xero Pay Item 설정값. CareTalk는 시간/금액(units)만 전송.
- **연차(leave) 메커니즘이 RateType 결정 근거:** Xero는 기본급(MULTIPLE 기반 ordinary earnings)에서만 연차 적립 → Casual(연차 없음)=`RATEPERUNIT`, Part/Full-time 근무(연차 있음)=`MULTIPLE`.
- **CSV 8줄 수정** (`CareTalkData/Xero_PayItems_List.csv`):
  - 1~6 NDIS Casual: `MULTIPLE`→`RATEPERUNIT` (AGED Casual과 통일 — #1 통일 방향=RATEPERUNIT)
  - 13 Homecare Hourly: `RATEPERUNIT,TBD`→`MULTIPLE,1.0` (Part/Full-time 기본급, 연차 적립)
  - 19 Time in lieu: `PAYMENT,-`→`RATEPERUNIT,TBD` (PAYMENT는 이름 `(Payment)`의 일부였음, overtime 시간 표시)
- **회계담당용 가이드 신규:** `CareTalkData/GUIDE_20260625_Xero_PayItem_RateType_Setup.md` (26 PayItem RateType/Value 표 + 변경 이유 + TBD 단가 설정 안내).

### Track B — Travel(KM) 금액 CareTalk 계산 (옵션2, caretalk-pipeline)
- **방향:** Xero 전송 전 CareTalk가 SiteConfig 단가로 `ActualKm × 단가 = Travel 금액` 계산해 units=금액 전송 (= Broken Shift 패턴). 에릭 결정: "업로드 전에 CARETALK_Config 가격으로 계산해놓자".
- **단가 출처:** `CARETALK_Config`의 `{AREA}_SUPPORTER_KM_CLAIM` (AGED/NDIS/NDIS_VEHICLE/ICARE_VEHICLE, 전부 $0.99). 화면 `/Admin/Manage/SiteConfigManage` (키-값 설정 테이블, SP `usp_CARETALK_MNG_SELECT_Configuration`). 고객용 `*_CUSTOMER_KM_CLAIM`(1.1/1.27/2.76)은 청구용·무관.
- **구현 (data→backend→qa 파이프라인):**
  - SP 신규: `CareTalkData/20260625_usp_CARETALK_XER_Select_SupporterKmRate.sql` (`SUPPORTER_KM_CLAIM` 키만 반환, SSMS 실행, 테스트DB 적용됨)
  - Repository: `XeroRepository.SelectSupporterKmRatesAsync()` → `Dictionary<string,decimal>`
  - Service: `XeroPayrollService` — `UpdatePayslipAllowancesAsync` Travel 금액화, `GetReconciliationAsync` KM 금액 정합, `KmRateFor` 헬퍼(키 `{AREA}_SUPPORTER_KM_CLAIM`, 폴백 0 + `LogWarn`)
  - 문서: 가이드 §6/§2 갱신 + CSV 24행 `RATEPERUNIT,0.99`→`1.0`
- **QA PASS:** 전체 솔루션 빌드 0오류, 배치행 Area=NDIS(43)/AGED(82) 2종 단가 100% 매칭(폴백 위험 없음), BS/HighCare/Sleepover/HOURS 무회귀. 실데이터 검산 $21.78/$19.80.

## 결정/맥락 (왜)
- **Track A 코드 무관:** EarningsRateMapping은 `(PayType,Area,ContractType)→XeroEarningsRateId+이름`만 저장, RateType 보관 안 함. 전송 units는 RateType 무관. → Track A = CSV 명세 + Xero 설정 가이드만.
- **#1 NDIS/AGED 통일 방향 = RATEPERUNIT:** 초기 "AGED→MULTIPLE" 검토했으나 원본 이미지에 AGED Casual 배율 표기 없음 + Casual은 연차 무관 → Casual 전체(1~12) RATEPERUNIT 통일.
- **Travel = BS 패턴:** CareTalk가 금액 계산 → units=금액 → Xero Pay Item을 금액 받게(RATEPERUNIT $1.00). SiteConfig에 km 단가가 이미 있어 BS보다 작업 적음(단가·화면 기존).

## 다음 할 일
- [ ] **프로덕션 배포:** SP `20260625_usp_CARETALK_XER_Select_SupporterKmRate.sql` SSMS 실행 (현재 테스트DB만)
- [ ] **★Xero 설정(회계):** "Travel Allowance (per KM)" RATEPERUNIT `$0.99`→`$1.00` (안 바꾸면 이중 곱셈으로 금액 오류)
- [ ] **라이브 대조:** 실 Xero 전송 후 Reconciliation `Difference < 0.01` 확인 (E2E는 라이브 OAuth 의존이라 미검증)
- [ ] **Track C (High Care):** 고객별 시간당 $3/$6, 현재 Caretok 미추출 → BS처럼 실 구현 필요 가능성(가장 큼, 별도 brainstorming)
- [ ] **메인 머지:** 에릭 승인 대기 (브랜치 `claude/zero-accounting-australia-bk8n84`, main=VersionUpGrade)
- [ ] 잠재 리스크: 배치행 `Area varchar(10)` — 향후 `NDIS_VEHICLE`(11자)/`ICARE_VEHICLE`(13자) 유입 시 truncation·폴백 재검증
