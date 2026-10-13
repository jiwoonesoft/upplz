# upplz — 플러그인 마켓플레이스 (Claude Code · Codex)

웹 게임/앱을 iOS(맥 로컬 빌드)·Android(맥·윈도 로컬 Docker 하네스)로 빌드·배포하는 **upplz** 플러그인. 같은 플러그인이 Claude Code 와 Codex 에 설치된다.

## 설치

```
# Claude Code — 설치 중 Upplz API Key·Upplz MCP Endpoint 입력(→ 번들 MCP 연결)
/plugin marketplace add jiwoonesoft/upplz
/plugin install upplz@upplz
/reload-plugins
/upplz:init

# Codex — 스킬만 설치. MCP 는 대시보드 프로젝트 페이지 「Codex」탭의 명령 두 줄(~/.codex/.env + codex mcp add)로 연결 후 재시작
codex plugin marketplace add jiwoonesoft/upplz
codex plugin add upplz@upplz
$upplz:upplz-init
```

- **스킬**: `upplz-init`(배포 설정 원스톱)·`upplz-upload-build-only`(테스트 업로드)·`upplz-upload`(출시 업로드)·`upplz-metadata`·`upplz-icon`·`upplz-screenshot`.
- **커맨드(Claude Code)**: `/upplz:init` — `upplz-init` 스킬을 읽는 얇은 래퍼(내장 `/init` 이 아니다).
- **API 키**: upplz 대시보드에서 **프로젝트를 만든 뒤 프로젝트 페이지에서** 발급(키 1개 = 프로젝트 1개).
- 2026-10-12 에 저장소 이름이 `upplz-plugins` → `upplz`, 마켓 이름이 `upplz-tools` → `upplz` 로 바뀌었다. 옛 이름으로 등록한 Claude Code 는 [배포 가이드](./DISTRIBUTION.md)의 「기존 사용자 이관」을 따른다.
- 상세: [플러그인 README](./plugins/upplz/README.md) · [배포 가이드](./DISTRIBUTION.md).

## 저장소 구조

```
.claude-plugin/marketplace.json    # Claude Code 카탈로그(이름 upplz)
.agents/plugins/marketplace.json   # Codex 카탈로그(이름 upplz)
plugins/upplz/                     # 플러그인 (upplz.git 의 plugin/ 동기화본 — .claude-plugin/ + .codex-plugin/)
DISTRIBUTION.md                    # 퍼블리싱 담당자용 가이드
```

플러그인 소스 원본은 upplz 서버 저장소의 `plugin/` 이며, 릴리스마다 `plugins/upplz/` 로 동기화하고 두 매니페스트와 Claude Code 카탈로그(`.claude-plugin/marketplace.json`)의 `version` 을 올린다(Codex 카탈로그 항목에는 `version` 이 없다).
