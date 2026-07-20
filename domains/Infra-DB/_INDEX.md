---
domain: Infra-DB
updated: 2026-07-20
status: done
---
# Infra-DB

## 현재 상태
`CARETALK_Roles`에 RoleId 310 / COORDI_ASSISTANT 추가(테스트·실서버 양쪽, `IF NOT EXISTS` 패치). 실사용 시 UserRoles·Menu_Role 매핑 필요. 권한 3축=Roles→UserRoles+RoleClaims+Menu_Role.
- (이전) Codex 스킬 카탈로그 점검 완료 — 139개(프로젝트 52/전역 40/시스템 5/플러그인 42), 2% 축약은 장애 아님. `wiki-save` `.agents`/`.claude` PlanManager 규칙 동기화·해시 검증.
- 세컨드 브레인 구축·마무리 완료 — vault 활성화(dataview/templater/obsidian-git), GitHub 백업(Caretalk-Brain), VersionUpGrade 머지(8bf8995a), 메모리 청산.
- 01.07.2026: 실서버 UserId `430219792-` → `430219792`(하이픈 제거) 정정 완료·검증(28테이블/29컬럼→1,378행/32컬럼, 옛값 잔존 0). 원래 하이픈 없던 NDIS번호 데이터와 통합. ZeroSum은 Xero 무관 확인.
- 26.06.26: `/wiki-load` 읽기 스킬 신설(.claude+.agents 미러) + CLAUDE.md 검색규칙 graphify/QMD 청산 feature 브랜치 동기화.

## 세션 로그
- [[2026-06-25-second-brain-setup]] — 세컨드 브레인 구축 + Karpathy 가이드라인 도입
- [[2026-06-25-second-brain-finalize]] — 볼트 활성화 + git 백업 + 머지 + 메모리 청산
- [[2026-06-26-claude-md-search-rule-sync]] — graphify/QMD 검색규칙 청산: feature 브랜치 동기화(96905799) + 글로벌 CLAUDE.md 정리 (본류는 8bf8995a로 이미 완료)
- [[2026-07-01-fix-userid-remove-hyphen]] — 실서버 UserId 하이픈 제거 정정(전수 스캔→안전 스크립트→검증), NDIS번호와 표기 통합
- [[2026-07-14-codex-skill-inventory-wiki-save-sync]] — Codex 스킬 139개·플러그인 현황 점검, 2% 축약 의미 확인, `wiki-save` 양쪽 미러 동기화
- [[2026-07-20-role-310-coordi-assistant]] — CARETALK_Roles에 RoleId 310 COORDI_ASSISTANT 추가(테스트·실서버), 권한 테이블 9종 파악
<!-- /wiki-save 가 최신 로그를 [[YYYY-MM-DD-slug]] 형태로 여기에 추가 -->
