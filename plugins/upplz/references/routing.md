# 라우팅 — 머신(맥/윈도) × 대상(iOS/Android)

모든 upplz 스킬은 첫 단계에서 이 문서를 로드해 **머신**과 **대상**을 확정한 뒤 진행한다.

## 머신 감지

빌드가 실행되는 곳은 **사용자 로컬 머신**(Claude Code 또는 Codex 가 도는 환경)이다. 원격 MCP 서버 OS가 아니다.

- `uname -s` → `Darwin`이면 맥, `MINGW*`/`MSYS*`면 윈도(Git Bash), 그 외는 지원하지 않는다.
- 불확실하면 사용자에게 "지금 빌드를 돌릴 이 컴퓨터가 맥인가요, 윈도인가요?"로 확인한다.

## 대상 감지

저장소 구조로 판정(또는 사용자 확인):
- `ios/` 존재 + 웹/Capacitor/Flutter/RN → **iOS** 대상 가능.
- `android/` 또는 안드로이드 빌드 요청 → **Android** 대상.

## 2×2 분기

| 대상 \ 머신 | 맥 (로컬) | 윈도 (로컬) |
|---|---|---|
| **iOS** | ✅ 로컬 빌드 | 🚧 미지원(후속 — 맥 전용) |
| **Android** | ✅ 로컬(호스트 JDK·Node) | ⚠️ 로컬(Git Bash + JDK + Node, 맥과 같은 스크립트 — **미검증**: 윈도 실측 전) |

## 게이트 규칙

- **MCP 연결 게이트(최우선)**: 첫 upplz MCP 도구 호출이 연결 실패/인증 실패하면 **호스트별로** 확인하도록 안내하고 **중단**한다.
  - **Claude Code**: 플러그인은 MCP 를 번들하지 않는다(2026.10.5 부터) — 앱 저장소마다 local 범위로 등록한 `upplz` 서버가 연결 대상이다. `claude mcp list` 로 본다(키 값이 찍히지 않는다 — `claude mcp get upplz` 는 헤더의 키를 그대로 찍으므로 **쓰지 않는다**). ① `upplz:` 줄이 없으면 이 디렉토리에 등록이 없다 — 대시보드 프로젝트 페이지 「Claude Code」탭의 `claude mcp add` 줄을 **앱 저장소 디렉토리에서** 실행하고 Claude Code 를 재시작한다(`upplz-init` 스킬 §0 과 같다). ② `plugin:upplz:upplz` 줄만 있으면 옛 플러그인(2026.10.4 이하 — 설정값 1개가 전 프로젝트에 쓰인다)이다 — `/plugin` 에서 업데이트를 권하되, ① 로 등록하면 local 이 우선하므로 업데이트 없이도 진행된다. ③ `upplz:` 줄의 URL 이 자리표시자(`https://YOUR-UPPLZ-ENDPOINT`)거나 `/mcp` 로 끝나지 않으면 대시보드의 실제 주소로 ① 을 다시 한다(교체형이라 그대로 덮인다). ④ 줄이 있는데도 401 `invalid_or_inactive` 면 다른 범위(user)에 다른 프로젝트의 키로 등록된 `upplz` 일 수 있다 — `claude mcp list` 는 범위를 구분해 보여 주지 않으므로 ① 을 그대로 실행한다(local 이 우선한다).
  - **Codex**: 둘을 확인한다 — ① 키 줄: `grep -qs '^UPPLZ_API_KEY=.' ~/.codex/.env && echo set || echo missing` 이 `set` 인가(값을 찍지 않는 명령이다 — `missing` 이면 줄이 없거나 값이 비었거나 파일이 없다). `cat` 으로 파일을 열거나 `echo` 로 변수를 찍는 것처럼 **값이 출력되는 명령**, `env`·`printenv`·`set` 처럼 **환경 전체를 찍는 명령은 쓰지 않는다** — 키가 이 대화와 세션 로그에 남는다(Codex 는 `.env` 의 값을 기동 때 자기 환경에 올리고, 소스 근거로는 에이전트가 실행하는 셸 명령에도 그대로 넘긴다). ② 등록: `codex mcp get upplz` 출력의 `url` 줄이 자리표시자(`https://YOUR-UPPLZ-ENDPOINT`)가 아니고 `/mcp` 로 끝나며 `bearer_token_env_var` 줄에 `UPPLZ_API_KEY` 가 보이는가(이 출력에 키 값은 없다 — 변수 이름만 찍힌다). 실제 주소가 맞는지는 에이전트가 판정할 수 없다 — 의심되면 사용자에게 대시보드 「Codex」탭의 주소와 대조해 달라고 한다(엔드포인트를 되묻지 않는다). 없거나 틀리면 대시보드 프로젝트 페이지 「Codex」탭의 명령 두 줄(키 기록 → `codex mcp add`)을 다시 실행하고(첫 줄은 키가 셸 히스토리에 남는다 — 대시보드 안내 참조) **Codex 를 재시작**한다(MCP 서버는 기동 때 로드된다). 절차는 `upplz-init` 스킬 §0 과 같다.
  - 401 응답에 `reason`/`hint`가 있으면 그대로 안내한다: `missing_header`(`Authorization` 헤더가 안 실렸다 — Claude Code 는 `--header` 없이 등록된 것이라 위 Claude Code 갈래 ① 로 다시 등록하고, Codex 는 `codex mcp get upplz` 에 `bearer_token_env_var` 줄이 없는 등록이라 `.env` 를 고쳐도 풀리지 않는다: 위 Codex 갈래대로 「Codex」탭의 `codex mcp add` 줄을 다시 실행하고 재시작한다) / `malformed_header`(Bearer 형식 아님·공백 등) / `invalid_or_inactive`(키가 틀렸거나 **재발급·프로젝트 전환·멤버십 회수로 폐기됨** — 대시보드 **프로젝트 페이지**에서 키를 재발급했다면 이전 키는 즉시 비활성이므로 위 호스트별 자리에 새 키를 넣는다. 키 1개는 프로젝트 1개에 묶인다 — Claude Code 는 앱 저장소마다 그 프로젝트의 키를 등록하므로 다른 저장소의 등록을 가져오지 않는지 보고, Codex 는 다른 프로젝트를 배포하려면 `.env` 의 키를 교체한다). `reason`이 없는 구형 응답은 위 세 가지를 모두 점검하도록 안내한다.
  - Codex 에서 `UPPLZ_API_KEY` 가 `~/.codex/.env` 에 없거나 비어 있으면 요청이 서버까지 가지 않는다 — 401 이 아니라 Codex 가 기동 때 upplz MCP 서버를 올리지 않아 도구가 목록에 보이지 않는다. 이때 세션 시작 오류 문구가 `Environment variable UPPLZ_API_KEY for MCP server 'upplz' is not set` / `is empty` 라는 것은 Codex 0.162.1 소스 근거(`resolve_bearer_token`)이고 **실측 전**이다 — 문구가 달라도 아래 점검으로 판정한다. 실측된 점검은 `codex doctor --all --no-color` 다: 키 줄이 없으면 `mcp` 항목 아래 `upplz` 줄에 `bearer token env var UPPLZ_API_KEY is not set` 가 보이고, 키 줄이 읽히면 그 줄이 사라진다(다른 문제가 없으면 `✓ mcp` 에 등록된 서버 수가 찍히는데 그 수는 환경마다 다르니 판정에 쓰지 않는다. 키 값은 이 출력에 찍히지 않는다). 응답의 `reason` 이 없으니 위 Codex 갈래의 ① 키 줄 확인부터 한다. ① 키 줄이 `set` 이고 ② 등록도 정상이며 재시작까지 마쳤는데도 도구가 없으면 **같은 명령을 되풀이하지 않는다** — `codex doctor --all --no-color` 출력에 `upplz` 줄이 있으면 그 줄(오류 문구 그대로)을, 없으면 `mcp` 항목 전체(또는 '`upplz` 오류 줄 없음')를 그대로 사용자에게 전하고 **멈춘다**. 엔드포인트 불통·샌드박스의 네트워크 제한·관리형 프로필의 환경변수 필터(HTTP bearer 에는 미적용 가능성 — 미확인)는 이 스킬이 고칠 수 없는 범위이며, 사용자가 upplz 쪽에 그 출력을 그대로 전달하면 된다(키 값은 그 출력에 찍히지 않는다).
  - 500 응답이면 인증 처리 중 서버 내부 오류(예: DB 장애)다 — 사용자 설정 문제가 아니므로 잠시 후 재시도하도록 안내한다.
- **Codex 샌드박스 — upplz 명령은 샌드박스 밖에서 실행한다(Codex 만)**: Codex 는 에이전트 명령을 샌드박스 안에서 돌리고 기본으로 네트워크를 막는다(Codex 0.162.1 `codex debug prompt-input` 의 권한 안내로 확인 — 설정이 없으면 `sandbox_mode` 가 `read-only` 이고 네트워크가 제한되며, 샌드박스 밖 실행은 사용자 승인으로 한다). upplz 스크립트(인증서 저장소 `git ls-remote`·clone, brew·fastlane·jarsigner·node·키체인)와 `upplz-init` §0 의 `~/.codex` 쓰기는 샌드박스 안에서 되지 않으므로 처음부터 **승인 상승(샌드박스 밖 실행 요청)** 으로 실행한다. 저장소 접근 점검은 git 오류를 버리므로(`2>/dev/null`) 샌드박스의 네트워크 차단도 `[preflight]`·`[fetch]` 의 `…저장소에 접근할 수 없습니다` 로만 보인다 — 샌드박스 안에서 실행한 스크립트가 이 줄로 멈췄다면 collaborator 요청을 안내하기 전에 승인 상승으로 다시 실행한다. 실제 Codex 세션에서의 승인 흐름은 미검증이다.
- **도구 미설치는 게이트가 아니다(모든 스킬)**: 어느 스킬에서든 `command not found`·`미설치`·preflight 의 `✗` 로 멈추면 중단하지 않는다 — `first-time-setup.md` 「빌드 환경 설치」대로 **에이전트가 설치하고 같은 스크립트·명령을 다시 실행**한다(관리자 비밀번호·GUI 확인·로그인만 사용자에게 넘긴다). 설치 방법만 알려 주고 멈추지 않는다.
- **셋업 게이트(iOS·Android)**: MCP 연결이 확인되면 **첫 MCP 도구 호출**은 `get_setup_status({ platform })` 으로 한다(iOS 는 파라미터 없이 호출해도 같다 — 미지정 기본이 `ios`. Android 는 패키지명을 이미 알면 `packageName` 도 함께 넘겨 힌트의 요청문에 찍히게 한다 — 선택이며 판정은 바뀌지 않는다). `setupComplete`면 빌드 단계, 아니면 `nextStep`으로 분기(`first-time-setup.md` "셋업 완료 판정" — `nextStep` 이 `add_provisioning_profile` 이면 first-time-setup.md ⑨). 로컬 선행 단계는 이 판정 직후 먼저 수행한다 — iOS 는 ①`analyze_repository`~⑤ push(`analyze_repository` 자체는 MCP 도구이지만 `get_setup_status` 판정과는 별개로 그 다음에 순서대로 진행하는 로컬 체크리스트다), Android 는 `first-time-setup.md` "Android 최초 셋업" 1–4.
  - **최초 셋업의 명시적 진입점은 `upplz-init`**(Claude Code `/upplz:init` · Codex `$upplz:upplz-init`)이며, 동일 절차(`first-time-setup.md`)를 수행한다. 사용자가 아직 실행하지 않았다면 그것을 권하되, 스킬 안에서 이어서 진행해 달라고 하면 같은 절차를 여기서 그대로 수행한다.
- **대상=iOS ∧ 머신≠맥** → 다음을 안내하고 **중단**한다:
  > "현재 upplz는 iOS를 **맥 로컬 빌드**로만 지원합니다. 윈도에서의 iOS 빌드는 후속으로 제공될 예정입니다. 맥에서 다시 시도해 주세요(Android 는 이 윈도 머신에서 진행할 수 있습니다 — 윈도 Android 는 아직 실측 전입니다)."
  <!-- 후속 삽입 지점: 윈도 iOS(CI 러너) 경로를 여기에 추가한다. -->
- **대상=iOS ∧ 머신=맥** → 정상 경로. 각 스킬의 Process 진행.
- **대상=Android ∧ 머신=맥** → 정상 경로. **Android 빌드는 upplz 가 하지 않는다** — 에이전트가 저장소에 맞는 방법으로 서명 없는 AAB 를 직접 만들고, upplz 는 서명(`sign_android_bundle`)·Play 업로드·메타데이터·기록만 한다. 전부 사용자 머신의 **호스트 JDK(keytool·jarsigner)·Node** 로 돈다(Docker 없음). 흐름은 7단계다:
  1. `get_setup_status({ platform: "android" })` — `nextStep`·`android.nextVersionCode` 확인(패키지명을 알면 `packageName` 도 넘긴다).
  2. (필요 시) `setup_android_keystore`(조건부 — 저장소 URL·비밀번호가 저장돼 있으면 협업자도 실행) — 저장소 URL·프로젝트 비밀번호·keystore 준비.
  3. **에이전트가 직접** Android 프로젝트 준비(Capacitor 면 `npm i @capacitor/android@<core 와 같은 버전>` → `npx cap add android` → `npm run build` → `npx cap sync android`. Android SDK 는 `ANDROID_HOME` 또는 `android/local.properties`) + `versionCode`(= `nextVersionCode`)·`versionName` 설정 + 서명 없는 release AAB 빌드(`cd android && ./gradlew bundleRelease`). release 에 signingConfig 를 넣지 않는다. JDK 21·`JAVA_HOME` 이 필요하다(Capacitor 8·AGP 8.13 — JDK 17 이면 `invalid source release: 21`). 빌드 실패는 upplz 도구가 아니라 에이전트가 원인·해결을 보고한다.
  4. `sign_android_bundle({ packageName, repoUrl, aabPath })` → preflightScript → signScript → `.upplz/app-release.aab`.
  5. `report_build_result({ buildId, status: "completed", versionCode, marketingVersion })`.
  6. (최초 1회 수동) Play Console 앱 생성·내부 테스트 트랙에 `.upplz/app-release.aab` 수동 업로드 → (Play SA 가 없으면 owner 가 `register_android_credentials`) → `upload_to_store({ platform: "android", buildId })`.
  7. 메타데이터: `pull_android_metadata` / `apply_android_metadata`.
  각 스킬의 Process(Android) 진행. **협업자도 빌드·서명·업로드·스토어 메타데이터까지** 직접 수행한다 — 조건은 인증서 저장소(keystore·Play Service Account 가 사는 private GitHub 저장소)의 collaborator + 활성 멤버십이다. JDK·Node 가 없으면 에이전트가 설치한다(first-time-setup.md 「빌드 환경 설치」).
- **대상=Android ∧ 머신=윈도** → 맥과 **같은 경로·같은 스크립트**다(**미검증** — 윈도 실측 전이다. 스크립트가 경로 오류로 멈추면 출력을 그대로 사용자에게 보여 주고 upplz 에 알려 달라고 안내한다). 전제는 셋:
  1. **Git Bash**(Git for Windows) — 스크립트는 bash 이고, 인증서 저장소의 `.enc` 를 푸는 `openssl` 은 Git for Windows 에 들어 있다. `zip` 은 없을 수 있다(미검증 — 없으면 에이전트가 설치한다, first-time-setup.md 윈도 표). 스크립트는 반드시 Git Bash 에서 실행한다(PowerShell·cmd 아님).
  2. **JDK**(keytool·jarsigner — Android 를 빌드한 JDK 그대로, Temurin 21 권장) — `setup_android_keystore` 의 keystore 생성·검증과 `sign_android_bundle` 의 서명이 호스트 JDK 로 돈다. 없으면 에이전트가 설치한다(first-time-setup.md 「빌드 환경 설치」 윈도 표).
  3. **Node.js** — `upload_to_store`·`apply_android_metadata`·`pull_android_metadata` 가 호스트 Node 로 Play API 를 호출하고, owner 가 `register_android_credentials` 를 실행할 때 스크립트가 `node` 로 키 파일을 확인한다(`jq` 는 필요 없다).
  - 파일 경로 파라미터(`serviceAccountPath`)는 Git Bash 형식의 절대 경로로 넘긴다(예: `/c/Users/me/Downloads/my-project-1a2b3c.json`). 사용자 이름이 한글이면(`/c/Users/홍길동/…`) 도구가 거부하므로 `/c/upplz/` 같은 사용자 폴더 밖 영문 폴더로 옮긴다.
