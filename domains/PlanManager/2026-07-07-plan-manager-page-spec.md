---
domain: PlanManager
date: 2026-07-07
tags: [plan-manager, spec, pipeline, funding-management-type, validation-engine, reconciliation, aba, statement, coi, quickclaim, xero]
status: 플랜매니저 페이지 개발명세 v1 접수 — 인보이스 수신→검증→승인→퀵클레임→대사→ABA→스테이트먼트 전체 파이프라인 설계 문서(별도 신규 페이지). 코드 착수 전.
related: ["[[_INDEX]]", "[[QuickClaim/2026-07-07-quickclaim-caretalk-integration-workflow]]", "[[QuickClaim/2026-06-29-qca-ndis-kickoff-spec-db-mapping]]"]
---
# 플랜매니저 페이지 개발명세 v1 접수 — 전체 파이프라인 설계 (07.07)

원본: `CareTokSolutions/bDocs/플랜매니저_페이지_개발명세_v1.md` (작성일 2026-07-06, 대상=개발자, Downloads 사본에서 리포로 이관). 정부 클레임 단계(퀵클레임/QCA)는 이 파이프라인의 STEP 4 → [[QuickClaim/_INDEX|QuickClaim 도메인]]과 연결.

## 한 일
- 에릭이 넘긴 **플랜매니저 페이지 개발 구현 명세(v1)** 를 신규 PlanManager 도메인에 기록. 코드 작업은 아직 없음(설계/스펙 접수 단계).

## 핵심 설계 (스펙 요지)

### 0. 목표·원칙
- 제공기관 인보이스를 **자동 검증·분류·승인** → 승인건을 **퀵클레임으로 NDIA 클레임** → NDIA 입금 **대사(reconcile)** → **ABA로 제공기관 송금** → **스테이트먼트·밸런스 정산**까지 자동 흐름.
- 원칙: 자동화 최대화 + 사람 개입 최소화. 단 ⑴예외(플래그)와 ⑵실제 은행 송금 실행만 사람이 확인.
- 연동 전제: OCR=**케어톡 자체 내장**(Dext 미사용) / Xero=**연동 완료**(회계·스테이트먼트·지급·대사·ABA 내보내기) / 퀵클레임=**API 계약 확정 예정(공동 개발)**.

### 0-B. ★서비스팀 ↔ 플랜매니저 분리 (핵심)
- 케어톡은 이미 완성돼 **서비스팀(NDIS 케어 제공)** 이 사용 중. 플랜매니저는 그 위에 **완전 별도 신규 페이지**(예: `/Admin/PlanManagement/...`)로 구축, 오로지 인보이스 처리만 담당.
- 분리 3축: ①접근/UI 분리(역할기반 권한, 겸직자는 상단 모드 전환) ②데이터/역할 분리 ③회사의 두 역할 구분(ⓐ서비스 프로바이더=인보이스 발행 / ⓑ플랜매니저=타 제공기관 자금 관리·지급).
- ★**이해상충(COI)**: 우리가 플랜매니저인 참가자가 우리 서비스팀 케어도 받으면 COI 발생 → 시스템 자동 플래그 + arm's length(자기 인보이스 자기가 자동승인 금지, 별도 검토·기록). NDIS COI 규정(겸업 허용되나 관리·공개 필수) 준수.

### 1-B. ★STEP 0 자금관리 유형 라우팅 (intake 분기)
인보이스 유입 시 참가자 **fundingManagementType** 으로 파이프라인 분기:
| 케이스 | 우리 역할 | 처리 | 퀵클레임 | ABA 송금 |
|--------|-----------|------|----------|----------|
| ① 타 플랜매니저 관리 (PM_OTHER) | 서비스 제공자 | 인보이스 이메일 발송 | ✗ | 없음 |
| ② NDIA 관리 (AGENCY) | 서비스 제공자 | 직접 클레임 | ✓ `PROVIDER_DIRECT` | ✗ (NDIA가 우리 계좌 직접 입금→매출 기록) |
| ③ 우리가 플랜매니저 (PM_US) | 플랜매니저 | 검증·승인 후 클레임 | ✓ `PLAN_MANAGED` | ✓ 제공기관 송금 |
| ④ 자기관리 (SELF) | 서비스 제공자 | 참가자에게 직접 인보이스 | ✗ | 없음 |
- **②·③ 클레임 방식 동일** — 둘 다 myplace/PACE **Bulk Payment Request** 형식(support item·NDIS#·수량·단가). 퀵클레임엔 **같은 payload + `claimType` 플래그만** 다르게 전달. → 기존 QuickClaim 연동 설계와 정합.
- endorsement 주의: ②Agency-managed는 provider가 NDIA에 endorse('my provider') 필요(등록 2~3일, 미등록 ~10일 지급). 시스템에서 endorsement 상태 체크·경고.

### 상태 머신 (파이프라인)
`RECEIVED → VALIDATING → {AUTO_APPROVED | FLAGGED→APPROVED/REJECTED | REJECTED} → CLAIM_QUEUED → CLAIM_ACCEPTED/REJECTED → NDIA_PAID(대사) → PROVIDER_PAY_QUEUED → ABA_GENERATED → PAID → REMITTANCE_SENT → STATEMENTED`. 전이는 전부 AuditLog 기록(7년 보존).

### 데이터 모델 10 엔티티
Participant(★fundingManagementType) / Plan·BudgetCategory(allocated·spent·committed·remaining) / Provider(abn·bsb·account·riskFlag) / ServiceAgreement / Invoice / InvoiceLine / Claim(quickClaimRef·responseCode) / Remittance(NDIA) / Payment(ABA batch) / Statement / AuditLog.

### STEP 2 검증 규칙엔진 12체크 (핵심)
코드존재/유닛금액정합/가격상한(Price Guide)/서비스계약/예산잔액/참가자검증/날짜/ABN/중복/지원적격성/고위험/**COI**. 판정: hasReject→REJECTED(이메일 반송) / hasFlag→FLAGGED(예외 큐) / else→AUTO_APPROVED. 고액(임계값 초과)·신규 제공기관은 강제 FLAGGED. Smart Match(코드 추천)·Smart Resolve(사소 오류 자동수정 후 재검증)로 자동통과율↑.

### STEP 5 대사 (★경쟁사 핵심강점)
NDIA remittance + 은행 입금내역 ↔ Claim AI/규칙 매칭 → `NDIA_PAID`/`PARTIALLY_PAID`. 미매칭=예외 큐. **대사 통과분만 제공기관 송금**(오지급 방지).

### STEP 6 ABA (③만 해당)
`NDIA_PAID` & `claimType=PLAN_MANAGED` 건만 배치→ABA 파일(Type0 헤더+Type1 credit×N+Type7 합계, 120자 고정폭·센트 단위). ⚠️자금 이체 실행은 사람 최종 승인. 별첨 `플랜매니저_자동지급_파이프라인_구상.md` 4·5장 참조.

### STEP 7 스테이트먼트·밸런스·Xero
송금통지 이메일 / 참가자별 월 명세 / `category.spent`·`remaining` 재계산(committed=승인·미지급 / spent=지급완료 분리) / 인보이스·지급·NDIA 입금 **Xero 자동 반영**(케어톡=운영·청구, Xero=회계 원장).

## 결정/맥락 (왜)
- 플랜매니저를 별도 페이지로 재구축하는 이유: 우리가 **플랜매니저 + NDIS 케어 서비스 겸업**이라 두 역할이 섞이면 COI·데이터·권한이 엉킴. 물리적 분리가 설계 1원칙.
- 지급 안전장치: **NDIA 입금(대사 완료) 후에만** 제공기관 송금(승인 즉시 지급 금지). 참가자 자금=신탁/분리계좌. ★**제공기관 은행정보 변경=자동지급 보류+재검증**(사기 벡터 차단).
- 참고 업체: MYP×Profenso(PIA), Careview SyncPro, Wiise, Plan Partners, My Plan Manager, NDSP, Brevity, FlowLogic.

## 케어톡 기존 자산 재사용 (스펙 §14)
| 단계 | 재사용 | 기존 소스 |
|------|--------|-----------|
| 수신·OCR | ♻ | `ReimbursementV2Ndis` + 케어톡 자체 추출 |
| 검증엔진 | 🆕 | Price Guide·Balance 연동 |
| 클레임 | ♻ 확장 | `ClaimBulkupload`, `tmt`(Transmitter) → 퀵클레임 |
| 대사 | 🆕 | — |
| ABA·송금 | 🆕 | — |
| 스테이트먼트·밸런스 | ♻ | `Mailing`, `Balance*`, `TransactionsList`, Xero |

## 화면(IA) 10개 (스펙 §11)
대시보드 / 인보이스 수신함 / 예외 큐 / 클레임 / 대사 / 지급·ABA / 스테이트먼트·밸런스 / 참가자·플랜·제공기관·서비스계약(마스터) / 규정·연동 설정 / 감사로그.

## 다음 할 일
- [ ] 구현 우선순위(스펙 §13): **MVP=수신(OCR)→12체크 검증엔진→통과/플래그/반려→예외 큐→수동 승인** 먼저. 이후 클레임연동→대사→지급(ABA)→스테이트먼트/대시보드→고도화(Smart Match/Resolve).
- [ ] 퀵클레임(STEP 4)은 기존 QCA 연동과 통합 — `claimType`(PROVIDER_DIRECT/PLAN_MANAGED) 플래그로 ②③ 분기, payload는 Bulk Payment Request 공통. sandbox/최종 JSON 확인은 QuickClaim 미정 항목과 동일([[QuickClaim/2026-07-07-quickclaim-caretalk-integration-workflow]]).
- [ ] 통합 확정 필요: 퀵클레임(payload·응답·인증) / 은행 ABA 규격·업로드 / NDIS·PACE PRODA 접근 / 이메일·SMS 알림.
- [ ] 별첨 문서 `플랜매니저_자동지급_파이프라인_구상.md`(ABA 4·5장) 존재 여부·위치 확인 후 함께 보관.
