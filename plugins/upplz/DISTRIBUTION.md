# upplz 플러그인 정식 배포 가이드

고객에게 `upplz` 플러그인(현재 `2026.10.1`)을 배포하는 방법. **같은 디렉토리가 Claude Code 와 Codex 에 설치된다** — Claude Code 는 설치 한 번으로 스킬 + 커맨드 `/upplz:init` + 원격 MCP 연결 + 프로젝트 API 키가 함께 구성되고, Codex 는 스킬만 설치한 뒤 `$upplz-init` §0(또는 대시보드의 명령 두 줄)으로 MCP 를 연결한다.

## 배포 채널: 공개 마켓 저장소 `jiwoonesoft/upplz`

서버 코드와 분리된 **별도 공개 git 저장소**로 배포한다(플러그인만 담기므로 공개해도 서버 노출 없음). 2026-10-12 에 저장소 이름과 마켓 이름을 모두 `upplz` 로 바꿨다 — 그 전에 옛 이름으로 등록한 사용자는 「기존 사용자 이관」을 따른다.

### 저장소 레이아웃

```
upplz/                               # 공개 git 저장소 (마켓플레이스) — github.com/jiwoonesoft/upplz
├── .claude-plugin/marketplace.json  # Claude Code 카탈로그(이름 upplz)
├── .agents/plugins/marketplace.json # Codex 카탈로그(이름 upplz)
└── plugins/
    └── upplz/                       # 이 저장소의 plugin/ 내용을 그대로 복사
        ├── .claude-plugin/plugin.json   # Claude Code: userConfig + 번들 MCP
        ├── .codex-plugin/plugin.json    # Codex: 스킬만
        ├── commands/init.md             # Claude Code 전용 얇은 래퍼
        ├── skills/                      # upplz-init 포함 6종(두 호스트 공용)
        ├── references/
        ├── docker/android/Dockerfile    # web-game-android 이미지(2026.9.3~)
        ├── README.md
        └── DISTRIBUTION.md
```

`plugins/upplz/` 는 본 저장소(upplz.git)의 `plugin/` 디렉토리 내용을 그대로 옮긴 것이다. 릴리스마다 이 디렉토리를 동기화하고 **세 파일의 `version`** 을 올린다(두 매니페스트·`.claude-plugin/marketplace.json`). Codex 카탈로그(`.agents/plugins/marketplace.json`)의 플러그인 항목에는 `version` 필드가 없다 — Codex 가 보는 버전은 플러그인 자신의 `.codex-plugin/plugin.json` 에서 온다.

### 마켓 파일 두 개

`.claude-plugin/marketplace.json`:
```json
{
  "name": "upplz",
  "owner": { "name": "Upplz" },
  "description": "웹 게임/앱을 iOS·Android로 빌드·배포하는 upplz 플러그인 마켓플레이스",
  "plugins": [
    {
      "name": "upplz",
      "source": "./plugins/upplz",
      "description": "배포 설정 원스톱·테스트 업로드(빌드만)·출시 업로드(메타·아이콘·스샷 점검 포함)·메타데이터·아이콘·스크린샷 스킬 + 원격 upplz MCP 서버 연결 번들",
      "version": "2026.10.1",
      "author": { "name": "Upplz" },
      "keywords": ["ios", "android", "build", "deployment", "capacitor", "app-store", "play-store", "fastlane"]
    }
  ]
}
```

`.agents/plugins/marketplace.json`(Codex):
```json
{
  "name": "upplz",
  "interface": { "displayName": "Upplz" },
  "plugins": [
    {
      "name": "upplz",
      "source": { "source": "local", "path": "./plugins/upplz" },
      "category": "Developer Tools"
    }
  ]
}
```

> 두 스키마는 배포 직전 공식 문서로 최종 확인한다 — Claude Code: https://code.claude.com/docs/en/plugin-marketplaces.md · Codex: https://developers.openai.com/plugins/build/plugins . Codex 카탈로그 항목의 `policy` 는 생략해도 설치된다(2026-10-12 실측, codex-cli 0.162.1: `policy` 없는 위 카탈로그로 `codex plugin marketplace add`·`codex plugin add` 가 모두 성공했다).

## 고객 설치 절차

### Claude Code
`/plugin install` 중에 Upplz API Key·Upplz MCP Endpoint 를 입력하고, 마지막 줄 `/upplz:init` 으로 배포 설정을 시작한다(프로젝트당 1회).
```
/plugin marketplace add jiwoonesoft/upplz
/plugin install upplz@upplz
/reload-plugins
/upplz:init
```
- `userConfig` 에 따라 **Upplz API Key**(`sensitive: true`, keychain 저장)와 **Upplz MCP Endpoint** 를 묻는다. 저장된 값은 번들 MCP 서버의 `Authorization: Bearer ${user_config.api_key}` 헤더로 주입된다.

### Codex
```
codex plugin marketplace add jiwoonesoft/upplz
codex plugin add upplz@upplz
```
그 다음 대시보드 프로젝트 페이지 「Codex」탭의 명령 두 줄(① `~/.codex/.env` 에 `UPPLZ_API_KEY=` 기록 ② `codex mcp add upplz --url <엔드포인트>/mcp --bearer-token-env-var UPPLZ_API_KEY`)을 실행하고 **Codex 를 재시작**한 뒤, 배포할 앱의 저장소 디렉토리에서 `$upplz-init`. 사용자가 명령을 건너뛰었으면 `$upplz-init` §0 이 같은 절차를 안내한다.
- Codex 에는 설치 때 값을 묻는 기능(`userConfig`)이 없고 `${user_config.*}` 치환도 없어 MCP 를 번들하지 않는다(설계 D1). 키는 Codex 가 기동 때 읽는 `~/.codex/.env` 에만 둔다 — 셸 export·config.toml 평문·플러그인 번들 모두 쓰지 않는다.

### 공통
- 서버는 키를 해시 조회해 **`customerId`·`projectId`·역할(`owner`|`collaborator`)을 확정**한다(`src/mcp/auth.ts`) — 도구는 LLM 입력이 아니라 이 값만 신뢰한다.
- **API 키 발급**: 고객은 대시보드에서 **프로젝트를 먼저 만들고 그 프로젝트 페이지에서 키를 발급**받아야 한다.
- **키 1개 = 프로젝트 1개**: 프로젝트를 바꾸려면 Claude Code 는 `/plugin` → upplz 설정에서, Codex 는 `~/.codex/.env` 의 값을 바꾸고 재시작한다.

## 버전 업데이트 배포

1. 본 저장소 `plugin/` 에서 변경 → 리뷰/머지.
2. `plugin/.claude-plugin/plugin.json`·`plugin/.codex-plugin/plugin.json` 의 `version` 상향(같은 값 — `src/mcp/plugin-docs-drift.test.ts` 가 고정).
3. 마켓 저장소(`jiwoonesoft/upplz`)의 로컬 클론(이하 `<클론>`)의 `plugins/upplz/` 에 동기화: `rsync -a --delete plugin/ <클론>/plugins/upplz/`(점 디렉토리 `.claude-plugin/`·`.codex-plugin/` 과 `docker/` 가 함께 간다 — 동기화 뒤 `ls -a <클론>/plugins/upplz` 로 확인). `<클론>/.claude-plugin/marketplace.json` 의 `version` 반영(Codex 카탈로그는 건드리지 않는다) → `diff -r plugin <클론>/plugins/upplz` 가 비어 있는지 확인 → push(사용자 지시 시).
4. 고객 갱신: Claude Code `/plugin marketplace update upplz` 후 재설치/갱신, Codex `codex plugin marketplace upgrade upplz`(Git 마켓 스냅숏을 새로 받으면서 설치된 플러그인 캐시도 새 버전으로 바꾼다 — 재설치 불필요. 2026-10-12 실측은 https URL 을 git `insteadOf` 로 로컬 bare 저장소에 붙인 대역 Git 마켓에서 했고, 고객이 쓰는 `jiwoonesoft/upplz`(owner/repo) 마켓에서는 미검증이다). `upgrade` 뒤에도 `codex plugin list` 의 `upplz@upplz` 버전이 그대로면 `codex plugin remove upplz@upplz` → `codex plugin add upplz@upplz` 로 다시 설치한다. 로컬 경로로 추가한 마켓(아래 검증 체크리스트)은 `upgrade` 대상이 아니다(`is not configured as a Git marketplace`) — `codex plugin add upplz@upplz` 를 다시 실행하면 새 버전으로 교체된다.
5. **게시 순서 — 마켓이 웹 안내보다 먼저 나간다.** 대시보드는 마켓 저장소의 이름·명령(`upplz@upplz`)을 그대로 안내하므로, 웹이 먼저 배포되면 두 호스트 모두 새 설치 명령이 실패한다. 순서는 ⓪ 마켓 저장소 이름 변경(`gh repo rename` — 저장소 이름이 바뀌는 릴리스만. 2026-10-12 의 `jiwoonesoft/upplz` 변경은 완료) → ① 마켓 push(3) → ② 공개 저장소로 두 호스트 설치 확인(Claude Code `/plugin marketplace add jiwoonesoft/upplz` → `/plugin install upplz@upplz`, Codex `codex plugin marketplace add jiwoonesoft/upplz` → `codex plugin add upplz@upplz` — `owner/repo` 형식의 Codex 마켓 추가는 아직 미검증이라 여기서 처음 확인한다) → ③ 웹 배포 → ④ upplz 서버 재시작(서버 코드가 바뀌지 않은 릴리스면 ④ 는 생략 — 서버 문구가 웹 화면을 가리키면 반드시 ③ 뒤). 서버의 `pnpm build` 는 ① 전에 끝내 둔다(타입 오류면 게시를 시작하지 않는다 — 재시작 직전으로 미루지 않는다). 스키마가 바뀌는 릴리스면 ③·④ 를 upplz 서버 저장소 CLAUDE.md 「배포 순서」 1~5 로 대신한다. 마켓 이름이 바뀌는 릴리스면 ② 뒤에 「기존 사용자 이관」 절차를 기존 설치자에게 직접 알린다. 모두 사용자 지시가 있을 때만 한다.

> Claude Code 의 `userConfig` 는 **설치 시점에만** 프롬프트된다. 업데이트만으로는 재입력되지 않으므로, 키 변경이 필요하면 재설치를 안내한다. Codex 는 `.env` 값만 바꾸고 Codex 를 재시작하면 된다(MCP 서버는 기동 때 로드된다).

## 기존 사용자 이관 (2026-10-12 이름 변경)

Claude Code 에 옛 이름으로 등록된 마켓은 **제거 후 재추가**한다 — 등록 이름과 파일 안 이름이 어긋난 마켓을 Claude Code 가 어떻게 다루는지 확인되지 않아 이것을 표준으로 둔다. `/plugin marketplace remove upplz-tools` 가 설치된 플러그인 때문에 거부되면 첫 줄(`/plugin uninstall upplz@upplz-tools`)을 먼저 실행한다. `/plugin install upplz@upplz` 는 키·엔드포인트를 다시 묻는다.
```
/plugin uninstall upplz@upplz-tools
/plugin marketplace remove upplz-tools
/plugin marketplace add jiwoonesoft/upplz
/plugin install upplz@upplz
/reload-plugins
```
`upplz-plugins` 주소는 GitHub 리다이렉트로 계속 닿지만 새 안내에는 쓰지 않는다. 대시보드는 새 명령만 보여 주고 이 절차로 가는 단서가 없으므로, 이름이 바뀐 릴리스를 게시할 때 기존 설치자에게 이 절차를 직접 알린다(「버전 업데이트 배포」 5).

## ⚠️ MCP 엔드포인트 설정

`plugin.json`(Claude Code)의 `api_endpoint` 기본값은 자리표시자(`https://YOUR-UPPLZ-ENDPOINT`)다 — **실제 엔드포인트 주소는 공개 저장소에 박지 않는다**(개인/내부 인프라 노출 방지). 실제 주소는 별도(로컬 메모리·온보딩 채널)로 관리한다.

- Claude Code 고객은 설치 시 `userConfig` 프롬프트에서, Codex 고객은 `codex mcp add --url` 에, upplz 온보딩(대시보드 **프로젝트 페이지**)으로 안내받은 **실제 엔드포인트**를 넣는다. 대시보드는 `NEXT_PUBLIC_MCP_SERVER_URL` 이 설정돼 있을 때만 실제 값을 채워 보여 준다.
- 정식/대규모 배포용 **안정적 프로덕션 엔드포인트**(고정 도메인·이중화)가 정해지면 두 호스트 모두 기본값으로 박을 수 있다. 개인 머신 주소는 공개 기본값으로 두지 않는다.
- Streamable-HTTP 는 헤드리스/대화형 인증(MFA·패스프레이즈 프롬프트) 불가 — 인증은 Bearer 토큰(API 키) 단일 경로를 유지한다.

## Codex 설치 흐름 검증 (게시 전 체크리스트)

마켓 저장소 로컬 클론에 변경을 반영한 상태에서, Codex CLI 가 설치된 맥에서 로그인 없이:
1. `codex plugin marketplace add <클론 절대 경로>` → `codex plugin list` 에 `upplz@upplz` 가 보인다.
2. `codex plugin add upplz@upplz` → `find ~/.codex/plugins/cache -maxdepth 6 -name SKILL.md | grep upplz-init` 이 1건 이상, 같은 캐시(`~/.codex/plugins/cache/upplz/upplz/<버전>/`)에 `references/`·`docker/android/Dockerfile`·`.codex-plugin/plugin.json` 이 있고 `.codex-plugin/migrated-command-skills/` 는 **없다**(`commands/init.md` 의 `$ARGUMENTS` 가 Codex 의 커맨드→스킬 변환을 막는다 — 생기면 같은 절차의 스킬이 둘이 된다).
3. `codex mcp add upplz --url http://127.0.0.1:3100/mcp --bearer-token-env-var UPPLZ_API_KEY` → `codex mcp get upplz` 에 `bearer_token_env_var` 가 찍힌다.
4. 원복: `codex mcp remove upplz`, `codex plugin remove upplz@upplz`, `codex plugin marketplace remove upplz`.

## 서버 env 전제

플러그인이 연결하는 upplz 서버에 필요한 값은 `DATABASE_URL`·`ENCRYPTION_KEY`(필수)와 `MCP_PORT`(선택)뿐이다 — 브라우저 크레덴셜 업로드 폼(Upload 서비스)은 2026-09-25 에 삭제돼 공개 URL·포트 설정이 없다. 전체 목록은 루트 `.env.example` 참조.
