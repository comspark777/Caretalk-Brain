---
domain: Infra-DB
date: 2026-06-25
tags: [second-brain, obsidian, git, cleanup, merge]
status: done
related: ["[[_INDEX]]", "[[2026-06-25-second-brain-setup]]"]
---
# 세컨드 브레인 마무리 — 볼트 활성화 + git 백업 + 머지 + 메모리 청산

(이어서: [[2026-06-25-second-brain-setup]] 구축 완료 후 마무리)

## 한 일
- **Obsidian 볼트 활성화**: dataview·templater 플러그인을 기존 볼트(`D:\Obsidian_Mybusiness\Obsidian-vault`)에서 복사, 제한 모드 해제 확인. templater startup 에러 수정(`templates_folder` `_Templates`→`templates`, `startup_templates` 비움).
- **obsidian-git 추가**: Obsidian 내에서 commit-and-sync(자동 백업) 가능.
- **볼트 git 저장소화**: `git init`(main) + .gitignore(.obsidian workspace·.omc·.trash 제외) + 첫 커밋. GitHub **private** repo로 push → `https://github.com/comspark777/Caretalk-Brain`. 추적 21파일(노트+플러그인).
- **#2 머지**: `claude/second-brain-obsidian` → `VersionUpGrade` 충돌 없이 머지(8bf8995a) + push. CLAUDE.md에 Karpathy + 새 검색규칙(graphify/QMD 제거) 둘 다 반영.
- **#3 메모리 청산**: `project_graphify_setup` 삭제, `feedback_search_priority` 재작성(Obsidian→Serena→grep), `project_second_brain` 신규, MEMORY.md 인덱스 갱신, MCP 서버 카운트 7로 정정, 끊긴 링크 제거.
- **wiki-save 스킬**: "새 도메인은 임의 생성 금지, 확인 후 승인 시에만 생성" 규칙 명시.

## 결정/맥락 (왜)
- 볼트는 코드 레포 밖 **독립 git repo**로 백업(설계상 레포밖 + 브랜치 무관). 다른 PC는 `git clone`으로 플러그인까지 동기화됨.
- 새 도메인 난립 방지 위해 **"확인 후 생성"** 유지(완전 자동 X).
- graphify/QMD는 유지비 대비 효용 낮아 은퇴, Obsidian 세컨드 브레인으로 통합.

## 다음 할 일
- [ ] **Claude Code 재시작** → SessionStart hook(HOME.md 주입) + `/wiki-save` 인식 검증
- [ ] 볼트 로그 쌓일 때마다 obsidian-git **Commit-and-sync**로 백업
- [ ] (옵션) wiki-save 스킬은 [메인직접]이라 git 미커밋 상태 — 필요 시 커밋/미러
