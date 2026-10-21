# upplz 플러그인 (Claude Code · Codex)

웹 게임/앱을 iOS·Android로 빌드·배포하는 upplz 워크플로우 스킬 모음(iOS 는 맥 로컬 빌드, Android 는 에이전트가 직접 빌드하고 서명·업로드만 맥·윈도 로컬 스크립트). **같은 디렉토리가 두 호스트에 설치된다** — Claude Code 는 `.claude-plugin/plugin.json`(스킬 + 커맨드 — 2026.10.5 부터 MCP 미번들), Codex 는 `.codex-plugin/plugin.json`(스킬만)을 읽는다.

## 스킬
- `upplz-init` — **배포 설정 원스톱**(Claude Code `/upplz:init` · Codex `$upplz:upplz-init`, 인자 `ios|android|credentials`). upplz 도구가 보이지 않으면 §0 에서 MCP 연결부터 한다. 서버 셋업 상태(`get_setup_status`)와 이 머신의 로컬 상태를 함께 보고 남은 설정을 끝까지 처리한다. `.p8`·Play Service Account 는 **파일 경로만** 받아(본문을 읽지 않는다) 반환된 스크립트가 인증서 저장소에 암호화해 넣는다. 협업자는 owner 전용 단계만 owner 에게 요청하고 빌드·업로드·메타데이터까지 진행한다. 절차는 `references/first-time-setup.md` 하나를 upload 계열 스킬과 공유한다.
- `upplz-upload-build-only` — 테스트용 순수 빌드→스토어 업로드(최초 배포 셋업 게이트 포함)
- `upplz-upload` — 출시용: 메타데이터·아이콘·스크린샷 점검·갱신 + 빌드→업로드(풀 체인)
- `upplz-metadata` — 스토어 메타데이터 pull/apply
- `upplz-icon` — 아이콘 생성·갱신
- `upplz-screenshot` — 스크린샷 생성·프레임

## 커맨드(Claude Code 전용)
- `/upplz:init` — `skills/upplz-init/SKILL.md` 를 읽어 수행하는 얇은 래퍼. 내장 `/init` 과 다르다 — 항상 전체 이름으로 안내한다.

## 온보딩 순서
회원가입 → **프로젝트 생성**(대시보드) → **키 발급**(프로젝트 페이지) → 플러그인 설치(스킬만) → MCP 연결(Claude Code 는 **앱 저장소에서** `claude mcp add` 한 줄 — local 범위, Codex 는 `~/.codex/.env` + `codex mcp add` 두 줄. 둘 다 대시보드 프로젝트 페이지의 호스트 탭이 실제 값으로 보여 주고 `upplz-init` §0 도 같은 절차를 안내한다) → 재시작 → `upplz-init` → 배포.

## 설치

### Claude Code
플러그인은 스킬·커맨드만 설치한다(2026.10.5 부터 MCP 를 번들하지 않는다 — 플러그인 설정값은 사용자당 1개라 프로젝트별 키를 둘 수 없었다). MCP 는 **앱 저장소 디렉토리마다** local 범위로 등록한다 — 키 1개 = 프로젝트 1개이므로 저장소마다 그 프로젝트의 키를 등록하면 디렉토리를 옮길 때 키를 바꿔 끼우지 않는다(같은 URL 의 서버는 local > plugin 순으로 하나만 연결된다 — 2026-10-21 실측). 대시보드 프로젝트 페이지 「Claude Code」탭이 실제 값이 들어간 한 줄을 보여 준다(키가 셸 히스토리와 `~/.claude.json` 에 남는다 — 대시보드 안내 참조). 등록 뒤 그 디렉토리에서 Claude Code 를 재시작한다.

```
/plugin marketplace add jiwoonesoft/upplz
/plugin install upplz@upplz
/reload-plugins
```
```bash
# 앱 저장소 디렉토리에서(교체형 — 재발급 때도 같은 줄)
claude mcp remove upplz --scope local 2>/dev/null; claude mcp add --scope local --transport http upplz <엔드포인트>/mcp --header "Authorization: Bearer <프로젝트 키>"
```

### Codex
플러그인은 스킬만 설치한다. MCP 연결은 `~/.codex/.env` 의 `UPPLZ_API_KEY` 와 `codex mcp add` 로 등록한다 — 대시보드 프로젝트 페이지 「Codex」탭이 실제 값이 들어간 명령 두 줄을 보여 주고(첫 줄은 키가 셸 히스토리에 남는다 — 대시보드 안내 참조), `$upplz:upplz-init` §0 도 같은 절차를 안내한다. 등록 뒤 Codex 를 재시작한다.

```
codex plugin marketplace add jiwoonesoft/upplz
codex plugin add upplz@upplz
```

퍼블리싱 담당자용 상세(마켓 저장소 레이아웃·버전 갱신·엔드포인트 주의·기존 사용자 이관)는 [DISTRIBUTION.md](./DISTRIBUTION.md).

### 개발/내부 테스트
- Claude Code: `claude --plugin-dir /경로/upplz.git/plugin` → `/reload-plugins`(스킬 수정 즉시 반영). MCP 는 플러그인과 무관하게 앱 저장소의 local 등록을 쓰므로 그대로 연결된다. 전체 흐름은 `/plugin install` 로 검증.
- Codex: 마켓 저장소 로컬 클론을 `codex plugin marketplace add /경로/upplz` 로 등록하고 `codex plugin add upplz@upplz`. 캐시(`~/.codex/plugins/cache`)로 복사되므로 수정 뒤에는 `codex plugin remove upplz@upplz` 후 다시 설치한다.

## 연결·인증
- 서버는 Streamable HTTP(`<엔드포인트>/mcp`) + `Authorization: Bearer <API Key>` 다. API 키는 upplz 대시보드의 **프로젝트 페이지**에서 발급한다.
- Claude Code: 앱 저장소에서 `claude mcp add --scope local … --header` 로 등록한 서버가 헤더를 싣는다(`~/.claude.json` 의 그 프로젝트 경로 아래, 평문 0600).
- Codex: `codex mcp add upplz --url <엔드포인트>/mcp --bearer-token-env-var UPPLZ_API_KEY`. 키는 Codex 가 기동 때 읽는 `~/.codex/.env` 에 둔다(셸 export·config.toml 평문 불필요).
- **키 1개 = 프로젝트 1개.** 다른 프로젝트를 배포하려면 그 프로젝트의 키로 교체한다 — Claude Code 는 `/plugin` → upplz 설정, Codex 는 `~/.codex/.env` 의 값 교체 후 재시작.
- 스킬·참조 문서는 서로를 **플러그인 루트 기준 상대 경로**로 가리킨다 — 플러그인 루트 = SKILL.md 의 두 단계 위(`skills/<이름>/SKILL.md`). Claude Code 의 플러그인 루트 변수는 커맨드(`commands/init.md`)에서만 쓴다(Codex 에는 그 변수가 없다).

## 전제
- upplz MCP 서버 접속(원격) — 프로젝트 식별자·비밀번호 보관과 빌드·업로드·등록 스크립트 제공(서버는 Apple·Google 을 직접 부르지 않는다).
- 인증서 저장소(owner 소유 private GitHub)의 collaborator 권한 — 인증서·프로파일·keystore·Apple 팀 키·Play Service Account 가 모두 여기 있다.
- 로컬 toolchain(**빠진 것은 에이전트가 설치한다** — 비밀번호·GUI 확인·로그인만 사용자에게 요청): iOS 는 맥의 Xcode/fastlane/openssl(ImageMagick 은 아이콘·프레임 합성 시), Android 는 git/openssl + JDK(keytool·jarsigner — Android 를 빌드한 JDK 그대로, Capacitor 8 은 JDK 21) + Node.js(Play API) — 윈도는 Git Bash + JDK + Node.js(미검증). Docker 는 쓰지 않는다. 절차는 `references/first-time-setup.md` 「빌드 환경 설치」.
