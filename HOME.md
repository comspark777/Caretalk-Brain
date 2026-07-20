# CareTalk Second Brain

> 세션 시작 시 SessionStart hook이 이 파일을 자동 주입합니다.
> 특정 도메인을 깊이 다룰 땐 `domains/<Domain>/_INDEX.md`와 날짜별 로그를 Read 하세요.

## 도메인 지도

| 도메인 | 현재 상태 | 최근 갱신 |
|--------|-----------|-----------|
| [[domains/Xero/_INDEX\|Xero]] | ★골든패스 Step2 차단 버그 2건 수정·실서버 배포(캘린더 선택 초기화·기전송 재실행 차단) + 캘린더 셀렉트 개선(열린기간·N in this batch·자동선택). Step1 타임시트 approved 회계팀 실증. 15.07 전체 데이터 락 테스트=Step2~4 첫 통과 확인(Posting 전 DRAFT 수당 이중곱 주의). 남은 것=커밋 2건 머지(QuickClaim 브랜치)·매칭보강 패치·회계 3건·PayItem $1.00×2조직·접이식UI 머지 | 2026-07-14 |
| [[domains/Referral/_INDEX\|Referral]] | (기록 없음) | — |
| [[domains/Schedule/_INDEX\|Schedule]] | ★Aged Care(루트1) 미래 스케줄 가격/ClaimCode 일괄보정 실서버 적용 — 16:30 컷오프(@I_MinFromTime 신규) 233스케줄/281Detail·+$4,494.75(RunId 8EFFFC8C)·검증완료(478→198·695 ClaimCode채움·남은변경0·백업233/281). 07-06 SP 컷오프 파라미터화 재사용. (이전) 자동마감 JOB 8일실패 해결·백로그506마감 | 2026-07-20 |
| [[domains/Contribution/_INDEX\|Contribution]] | 늦은 활성화 누락 백필 5월분 적용·검증 완료(7명·14건·$282.80)·636은 정상 제외·4월이전 40건(~$1,122) 보류 | 2026-06-29 |
| [[domains/Intake/_INDEX\|Intake]] | (기록 없음) | — |
| [[domains/Infra-DB/_INDEX\|Infra-DB]] | CARETALK_Roles에 RoleId 310 COORDI_ASSISTANT 추가(테스트·실서버, IF NOT EXISTS 패치)·권한 테이블 9종 파악. (이전) Codex 스킬 139개 점검·`wiki-save` 규칙 동기화 | 2026-07-20 |
| [[domains/QuickClaim/_INDEX\|QuickClaim]] | ★push 모델 확정(Ben=직접 개발, 지원 없음) — pull(5 API) 폐기, CareTalk이 Public API 직접 호출. REGISTRATION_NUMBER=4050036107 확정했으나 sandbox 여전히 "not valid" → rego는 QC 대시보드 사전 등록 필요(에릭 처리 대기). push 클라이언트 설계 착수 | 2026-07-20 |
| [[domains/PlanManager/_INDEX\|PlanManager]] | 개발명세 v1 접수(설계·코드 착수 전) — 인보이스 수신→검증12체크→승인→퀵클레임(STEP4)→대사→ABA→스테이트먼트 전체 파이프라인을 별도 신규 페이지로 재구축. 서비스팀/PM 분리(COI)·자금관리 유형 라우팅(claimType) | 2026-07-07 |

## 사용법
- 저장: `"오늘 거 저장해줘"` 또는 `/wiki-save [도메인]`
- 로드: 새 세션이 이 인덱스를 자동 인지. 상세는 도메인 파일 Read.
