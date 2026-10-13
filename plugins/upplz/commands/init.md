---
description: upplz 배포 설정을 처음부터 끝까지 안내·처리한다(프로젝트 키로 MCP 연결 후 1회). 협업자도 owner 전용 4단계를 뺀 전부를 끝까지 진행한다.
argument-hint: "[ios|android]"
---

<!--
allowed-tools 를 일부러 두지 않는다 — 기본값은 대화 세션의 권한을 그대로 따른다.
이 키는 "커맨드가 쓸 수 있는 도구를 제한" 하는 화이트리스트라, 이름이 하나라도 어긋나면
그 도구를 아예 못 쓰게 된다. 그런데 upplz MCP 도구의 정식 이름은 **설치 경로에 따라 달라진다** —
플러그인 번들 서버면 `mcp__plugin_upplz_upplz__<도구>`, 사용자가 따로 등록한 서버면
`mcp__upplz__<도구>` 다. 하나로 못 박으면 다른 쪽 설치에서 커맨드가 통째로 막힌다.
목록을 비워 두면 최악의 경우 사용자가 도구 승인을 몇 번 더 할 뿐, 절대 막히지 않는다.
-->

<!--
이 파일은 Claude Code 전용 진입점이다. 절차 본문은 `skills/upplz-init/SKILL.md` 하나이고
(Codex 는 `$upplz:upplz-init` 으로 같은 파일을 쓴다), 여기서는 그 파일을 읽어 수행하라고만 말한다.
`${CLAUDE_PLUGIN_ROOT}` 는 커맨드에서만 쓴다 — Claude Code 는 커맨드 파일의 위치를 모델에
알려 주지 않는 대신 이 변수를 치환해 주고, 스킬은 두 호스트 모두 SKILL.md 경로를 알려 준다.
`$ARGUMENTS` 토큰은 **반드시 남겨 둔다** — Codex 는 commands/*.md 를 스킬로 자동 변환하는데
`$ARGUMENTS` 가 있는 파일은 건너뛰므로, 이 한 토큰이 upplz-init 스킬의 중복 생성을 막는다
(src/mcp/plugin-docs-drift.test.ts 가 고정한다).
-->

# `/upplz:init` — 배포 설정 원스톱

`${CLAUDE_PLUGIN_ROOT}/skills/upplz-init/SKILL.md` 를 `Read` 로 읽고, 인자 `$ARGUMENTS`(`ios`|`android`|없음)를 그 문서의 「인자」로 삼아 **§0 부터 끝까지 그대로 수행**한다. 단계 내용을 여기에 다시 적지 않는다.
