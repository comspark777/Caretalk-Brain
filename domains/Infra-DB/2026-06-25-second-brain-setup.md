---
domain: Infra-DB
date: 2026-06-25
tags: [second-brain, obsidian, wiki, karpathy, cleanup]
status: done
related: ["[[_INDEX]]", "[[../Xero/_INDEX]]"]
---
# CareTalk Second Brain 구축 + Karpathy 가이드라인 도입

## 한 일
- Obsidian 세컨드 브레인 vault 스캐폴딩: `D:\CaretokProject\CareTalk-Brain` (HOME.md + 도메인 6 _INDEX.md + templates/session-log.md)
- SessionStart hook: `~/.claude/hooks/session-brain-loader.js` (HOME.md 자동 주입) + settings.local.json 등록(SessionStart + additionalDirectories)
- `/wiki-save` 스킬: `.claude/skills/wiki-save/SKILL.md`
- QMD/Graphify 청산: settings.local.json에서 qmd 5종 제거, graphify-out(355MB) 삭제
- CLAUDE.md 검색 우선순위 교체(graphify/QMD → Obsidian+Serena+grep) — claude/second-brain-obsidian 커밋 5e40425c
- Karpathy 코딩 가이드라인 반입(.claude/rules/karpathy-guidelines.md) + CLAUDE.md 연결 → VersionUpGrade 푸시(389a2d31)

## 결정/맥락 (왜)
- MEMORY.md(Claude 전용)·handoff.md(덮어쓰기) 한계 극복 → 도메인별 영구 누적 + 사람이 Obsidian으로 직접 열람·편집.
- OMC wiki 스킬(git-ignored .omc/wiki)이 아니라 순수 Obsidian 볼트로 결정(레포 밖, 사람 친화).
- [메인직접] 항목은 즉시 적용(untracked/레포밖/로컬), CLAUDE.md만 별도 브랜치 격리(Xero PR 보호).

## 다음 할 일
- [ ] Claude Code 재시작 후 SessionStart hook 주입 + /wiki-save 인식 확인
- [ ] claude/second-brain-obsidian → VersionUpGrade 머지(PR) 시점 결정
- [ ] (옵션) MEMORY.md의 graphify/QMD 언급 정리 — 스펙 §7엔 있으나 상세계획 범위 외
- [ ] (옵션) wiki-save 스킬 dual-location 미러 필요 여부 확인
