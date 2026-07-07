# CareTalk Second Brain

> 세션 시작 시 SessionStart hook이 이 파일을 자동 주입합니다.
> 특정 도메인을 깊이 다룰 땐 `domains/<Domain>/_INDEX.md`와 날짜별 로그를 Read 하세요.

## 도메인 지도

| 도메인 | 현재 상태 | 최근 갱신 |
|--------|-----------|-----------|
| [[domains/Xero/_INDEX\|Xero]] | ★멀티테넌트 라이브 + 관리자 매뉴얼 docx 완성(bDocs, 실서버 캡처 13장·처음 세팅 순서). 실서버 배치 #9 차단=unmapped 136·missing 39·period BLOCK 76(컷오버). 남은 것=매칭보강 패치(승인대기)·회계 3건·PayItem $1.00×2조직·컷오버 합의 후 골든패스·접이식UI 머지 — 체크리스트는 07-07 로그 | 2026-07-07 |
| [[domains/Referral/_INDEX\|Referral]] | (기록 없음) | — |
| [[domains/Schedule/_INDEX\|Schedule]] | 서비스 아이템 가격 변경 후 미래 Ready 스케줄 가격 일괄 보정 완료(190,219 하위·15,198 스케줄/19,915 Detail·롤백 백업 검증) | 2026-07-06 |
| [[domains/Contribution/_INDEX\|Contribution]] | 늦은 활성화 누락 백필 5월분 적용·검증 완료(7명·14건·$282.80)·636은 정상 제외·4월이전 40건(~$1,122) 보류 | 2026-06-29 |
| [[domains/Intake/_INDEX\|Intake]] | (기록 없음) | — |
| [[domains/Infra-DB/_INDEX\|Infra-DB]] | 세컨드 브레인 완료 + 실서버 UserId `430219792-`→`430219792` 하이픈 제거 정정 완료·검증(NDIS번호와 통합) | 2026-07-01 |
| [[domains/QuickClaim/_INDEX\|QuickClaim]] | QCA 연동 워크플로우 정리 완료 — QCA가 CareTalk API를 pull/push-back 호출, 필수 5 API·상태 모델·idempotency 초안, sandbox/final JSON 확인 필요 | 2026-07-07 |
| [[domains/PlanManager/_INDEX\|PlanManager]] | 개발명세 v1 접수(설계·코드 착수 전) — 인보이스 수신→검증12체크→승인→퀵클레임(STEP4)→대사→ABA→스테이트먼트 전체 파이프라인을 별도 신규 페이지로 재구축. 서비스팀/PM 분리(COI)·자금관리 유형 라우팅(claimType) | 2026-07-07 |

## 사용법
- 저장: `"오늘 거 저장해줘"` 또는 `/wiki-save [도메인]`
- 로드: 새 세션이 이 인덱스를 자동 인지. 상세는 도메인 파일 Read.
