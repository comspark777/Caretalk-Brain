---
domain: Infra-DB
date: 2026-06-26
tags: [claude-md, search-priority, graphify, qmd, tooling-cleanup, branch-sync]
status: done
related: ["[[_INDEX]]", "[[2026-06-25-second-brain-finalize]]"]
---
# CLAUDE.md 검색규칙 청산 — feature 브랜치 동기화 + 글로벌 정리

## 한 일
- **발견**: 작업 브랜치 `claude/zero-accounting-australia-bk8n84`의 `CLAUDE.md`가 **옛 graphify/QMD 검색 우선순위**를 그대로 보유. `/wiki-load`로 Infra-DB 로드 중 적발.
- **사실 확인**: 본류 `VersionUpGrade`는 **이미 어제 청산 완료** — `8bf8995a [26.06.25] 세컨드 브레인 검색규칙 청산 머지 (graphify/QMD 제거 → Obsidian+Serena+grep)`. 즉 체리픽 불필요, feature 브랜치만 뒤처져 있던 것.
- **동기화**: `git checkout VersionUpGrade -- CLAUDE.md` → graphify/QMD 0건 확인 → 단독 커밋 `96905799` (CLAUDE.md만, 진행 중 Xero 작업물 미변동).
- **글로벌 정리**: `~/.claude/CLAUDE.md`의 `# graphify` 섹션 + `/graphify` 트리거 제거 (git 밖, 사용자 설정).

## 결정/맥락 (왜)
- graphify·QMD 도구를 제거(어제) → **Obsidian 세컨드 브레인이 그 역할 대체**. 새 검색 우선순위 = **① wiki(`/wiki-load`, `domains/<도메인>` 로그) → ② Serena MCP(LSP) → ③ Grep/Glob**.
- 메모리(`feedback_search_priority`)는 이미 "Obsidian→Serena→grep"로 갱신돼 있었으나 feature 브랜치 CLAUDE.md만 구세계 → 불일치 해소.

## 다음 할 일
- [ ] (선택) 글로벌 graphify **스킬 폴더**(`~/.claude/skills/graphify/`) 자체 잔존 여부 확인·정리 — 트리거/문서는 제거됨.
- [ ] AGENTS.md는 graphify/QMD 참조 없음(확인 완료) — 조치 불요.
