---
domain: Infra-DB
date: 2026-07-14
tags: [codex, skills, plugins, wiki-save]
status: done
related: ["[[_INDEX]]"]
---
# Codex 스킬 등록 현황 점검과 wiki-save 동기화

## 한 일
- GPT-5.6 Sol/Terra/Luna의 용도와 선택 기준을 현재 Codex 모델 메타데이터 및 공식 모델 안내를 기준으로 확인했다.
- 스킬 설명 축약 경고가 전체 컨텍스트의 최대 2%만 초기 스킬 카탈로그에 사용하는 제한이며, 실제 `SKILL.md` 본문이 잘리는 것은 아님을 확인했다.
- 현재 프로젝트 세션의 스킬을 출처별로 집계했다: 프로젝트 52개, 전역 사용자 40개, Codex 시스템 5개, 플러그인 제공 42개로 총 139개다.
- 플러그인은 17개 설치, 16개 활성, 1개(`ralph-loop`) 비활성 상태임을 확인했다.
- Codex용 `.agents/skills/wiki-save/SKILL.md`에 Claude 미러의 PlanManager 도메인 및 QuickClaim 구분 규칙을 반영했다.
- 두 `wiki-save` 파일의 내용과 줄바꿈을 동기화하고 SHA-256 해시 일치를 확인했다.
- Codex에서 스킬을 확실히 호출하는 방식은 `$wiki-save`이며, vault 템플릿과 `HOME.md`, QuickClaim/PlanManager 인덱스 접근이 가능함을 확인했다.

## 결정/맥락 (왜)
- 현재 경고는 스킬 139개의 이름·설명·경로가 초기 카탈로그 예산을 초과해 설명이 축약된 상태다. 모든 스킬은 여전히 인식되므로 즉시 기능 장애로 보지 않는다.
- 자동 선택 정확도가 떨어질 때만 사용하지 않는 전역 스킬이나 Figma/Superpowers 플러그인을 우선 정리한다. 명시적 `$스킬명` 호출은 축약 영향이 작다.
- 프로젝트 규칙에 따라 로컬 스킬은 `.agents/skills`와 `.claude/skills` 양쪽을 동일하게 유지한다.

## 다음 할 일
- [ ] 자동 스킬 선택이 자주 빗나갈 경우 전역 스킬 40개와 Figma/Superpowers 플러그인의 실제 사용 빈도를 확인해 비활성화 대상을 정한다.
- [ ] `wiki-save`를 수정할 때마다 `.agents`와 `.claude` 미러의 해시 일치를 검증한다.
