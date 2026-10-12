---
name: upplz-init
description: upplz 배포 설정을 처음부터 끝까지 안내·처리한다(프로젝트 키로 MCP 연결 후 1회). Claude Code 의 /upplz:init 과 같은 절차 — Codex 에서는 $upplz-init. upplz 도구가 보이지 않으면 §0 에서 MCP 연결부터 한다. 인자 ios|android. 협업자도 owner 전용 4단계를 뺀 전부를 끝까지 진행한다.
---

# upplz-init — 배포 설정 원스톱

> 플러그인 루트 = 이 SKILL.md 가 있는 디렉토리의 두 단계 위다(`skills/upplz-init/SKILL.md`). 아래 `<플러그인 루트>/…` 는 그 경로다.

사용자가 upplz 대시보드에서 **프로젝트를 만들고 그 프로젝트의 API 키로 MCP 연결을 마친 뒤** 실행하는 명시적 진입점이다. 이 스킬은 **서버 상태(`get_setup_status`)와 이 머신의 로컬 상태를 함께** 보고, 남은 설정을 끝까지 처리한다. Claude Code 에서는 `/upplz:init` 커맨드가 이 파일을 읽어 수행하고, Codex 에서는 `$upplz-init` 으로 직접 부른다.

**인자**: Claude Code 커맨드 경유면 `$ARGUMENTS`, Codex 면 `$upplz-init` 뒤에 적은 텍스트다. `android` 면 Android 경로를, `ios` 이거나 비어 있으면 iOS 경로를 기본으로 잡는다(비어 있으면 저장소 구조로 판정하고 사용자에게 확인).

> 절차의 **단일 출처는 `<플러그인 루트>/references/first-time-setup.md`** 다. 이 스킬은 그 문서를 실행하는 껍데기이며, 단계 내용을 여기에 다시 적지 않는다(upload 계열 스킬의 최초 셋업 게이트도 같은 문서를 쓴다).

## 0. MCP 연결 — upplz 도구가 보이지 않으면 먼저

upplz MCP 도구(`get_setup_status` 등)가 도구 목록에 있고 첫 호출이 성공하면 이 절은 건너뛴다. 도구가 없거나 첫 호출이 연결 실패·401 이면 **지금 실행 중인 호스트**에 따라 간다(에이전트는 자기가 Claude Code 인지 Codex 인지 안다).

### Codex 인 경우

Codex 플러그인은 MCP 서버를 번들하지 않는다(설치 때 값을 묻는 기능이 없다). 연결은 두 명령으로 등록한다 — 키는 `~/.codex/.env` 에, 서버는 `codex mcp add` 로.

0. **먼저 지금 상태를 본다** — 값이 찍히지 않는 두 명령만 쓴다.
   - 키 줄: `grep -qs '^UPPLZ_API_KEY=.' ~/.codex/.env && echo set || echo missing` 이 `set` 인가(`missing` 이면 줄이 없거나 값이 비었거나 파일이 없다).
   - 등록: `codex mcp get upplz` 출력의 `url` 줄이 `<엔드포인트>/mcp` 이고 `bearer_token_env_var` 줄에 `UPPLZ_API_KEY` 가 보이는가(변수 이름만 찍힌다).

   `cat` 으로 `.env` 를 열거나 `echo` 로 변수를 찍는 명령, `env`·`printenv`·`set` 처럼 환경 전체를 찍는 명령은 쓰지 않는다 — 키가 이 대화와 세션 기록에 남는다(Codex 는 `.env` 의 키를 에이전트가 실행하는 셸 명령의 환경에도 넘긴다).
   하나라도 아니면 1 로 간다. 둘 다 정상인데 이 세션에 도구가 없으면 등록 뒤 재시작하지 않은 것이 가장 흔하다 — 3 의 재시작 안내를 하고 멈춘다. 사용자가 이미 재시작했다고 하면 4 로 간다.
1. 사용자에게 upplz 대시보드 **프로젝트 페이지**의 「Codex」탭을 열게 한다. 거기에 **실제 엔드포인트와 이 프로젝트의 키가 채워진 명령 두 줄**이 있다. **사용자가 터미널에서 직접 실행하는 것이 기본**이다 — 키가 이 대화에 들어오지 않는다.
2. 사용자가 대신 해 달라고 하면 키를 받기 **전에** "여기에 붙여 넣은 키는 이 Codex 대화 기록에 남습니다 — 「Codex」탭의 명령을 터미널에서 직접 실행하는 편이 안전합니다" 를 알린다. 그래도 원하면 엔드포인트와 키를 받아 아래 두 명령을 실행한다(`~/.codex` 는 작업 폴더 밖이라 샌드박스 밖 실행 승인이 필요하다 — routing.md 「Codex 샌드박스」). 받은 키는 `~/.codex/.env` 에만 쓴다 — 에이전트가 로그·요약·다른 파일·저장소에 옮겨 적지 않고, 실행 뒤 다시 언급하지 않는다.

   ```bash
   mkdir -p ~/.codex && { grep -v '^UPPLZ_API_KEY=' ~/.codex/.env 2>/dev/null; printf 'UPPLZ_API_KEY=%s\n' '<프로젝트 키>'; } > ~/.codex/.env.tmp && mv ~/.codex/.env.tmp ~/.codex/.env && chmod 600 ~/.codex/.env
   ```

   ```bash
   codex mcp add upplz --url <엔드포인트>/mcp --bearer-token-env-var UPPLZ_API_KEY
   ```

   첫 줄은 교체형이다(기존 `UPPLZ_API_KEY=` 줄을 지우고 다시 쓴다 — 재발급 때 줄이 쌓이지 않는다). 엔드포인트가 `https://YOUR-UPPLZ-ENDPOINT` 자리표시자면 실행하지 말고 대시보드 안내의 실제 주소를 받는다.
3. "Codex 를 완전히 종료했다가 다시 시작한 뒤 `$upplz-init` 을 다시 실행하세요" 를 안내하고 **여기서 멈춘다**. MCP 서버는 Codex 기동 때 로드되므로 같은 세션에서는 도구가 나타나지 않는다.
4. 재시작 뒤에도 도구가 보이지 않거나 401 이면 `<플러그인 루트>/references/routing.md` 의 MCP 연결 게이트(Codex 갈래)를 따른다 — 1 로 돌아가 같은 명령을 되풀이하지 않는다.

### Claude Code 인 경우

플러그인 설치 때 입력한 **Upplz MCP Endpoint**·**Upplz API Key** 가 번들 MCP 서버에 들어간다. 연결·401 이면 `<플러그인 루트>/references/routing.md` 의 MCP 연결 게이트(Claude Code 갈래)를 따른다 — `/plugin` → upplz 설정에서 값을 확인·갱신하고 `/reload-plugins`.

### 둘 다 아닌 경우

다른 MCP 클라이언트다. Codex 갈래의 두 값(엔드포인트 `/mcp`, `Authorization: Bearer <키>` 헤더)을 그 클라이언트의 MCP 설정 형식으로 옮겨 적도록 안내만 하고 멈춘다(워크플로우 스킬은 Claude Code·Codex 를 대상으로 쓴 것이라 그 밖의 클라이언트에서는 실행하지 않는다).

## 1. 라우팅·연결 게이트

1. `<플러그인 루트>/references/routing.md` 를 읽어 **머신(맥/윈도) × 대상(iOS/Android)**을 확정한다. 게이트에 걸리면(iOS ∧ 비맥) 그 문서의 안내로 **중단**한다. 윈도에서는 Android 만 진행한다(Git Bash + Docker Desktop).
2. 첫 upplz MCP 도구 호출이 연결/인증에 실패하면 routing.md의 **MCP 연결 게이트**를 그대로 따르고 중단한다. 특히 401 `invalid_or_inactive`면:
   > "이 키는 폐기됐거나 다른 프로젝트의 키입니다. upplz 대시보드의 **프로젝트 페이지**에서 키를 다시 발급받아 넣어 주세요 — Claude Code 는 `/plugin` → upplz 설정에, Codex 는 `~/.codex/.env` 의 `UPPLZ_API_KEY` 에(§0 의 명령 두 줄) 넣고 재시작합니다."

## 2. 상태 판정 — 서버 + 로컬

**서버만 봐서는 안 된다.** 서버는 이 머신의 스캐폴드·toolchain 상태를 모르므로, `get_setup_status`의 `nextStep`만으로 분기하면 로컬 준비 단계를 통째로 건너뛴다.

1. **서버 상태**: 인자가 `android` 면 `get_setup_status({ platform: "android" })`, 아니면 **파라미터 없이** 호출한다(미지정 = `ios`). `team.role`·`nextStep`·`nextStepHint`·`setupComplete`·`apple.*` 와 `ios`·`android` 블록을 받는다.
   - **최상위 `nextStep`·`nextStepHint`·`setupComplete` 는 `platform` 이 고른 블록의 값**이고, `ios{…}`·`android{…}` 두 블록은 `platform` 과 무관하게 **항상 함께** 온다. **서버는 이 프로젝트가 어느 플랫폼인지 추정하지 않는다** — 1단계에서 확정한 대상을 그대로 넘긴다.
   - **iOS+Android 프로젝트는 지금 작업하는 플랫폼으로 호출**하고, 다른 쪽은 같은 응답의 `ios`/`android` 블록으로 본다(두 번 호출할 필요가 없다).
   - **서버는 Apple·Google 을 조회하지 않는다** — 판정은 DB 기록(시크릿 존재·audit·빌드)만 본다. 앱 레코드·프로비저닝 프로파일·저장소 안 파일의 실존은 판정에 없고, 없으면 스크립트 실행 중에 드러난다.
   - **Android 는 패키지명을 알게 된 순간부터 `packageName` 도 함께 넘긴다** — `get_setup_status({ platform: "android", packageName })`. 선택 파라미터이고 **판정은 바뀌지 않는다.** 대신 `nextStepHint` 의 요청문에 `setup_android_keystore({ packageName: "com.example.app" })` 처럼 **실제 패키지명이 찍혀**, 협업자가 owner 에게 그 문장을 그대로 전달하면 owner 가 되묻지 않는다. 첫 호출에서는 아직 모를 수 있다 — **그때는 넘기지 않고**(자리표시자가 나간다) 2-2 나 first-time-setup.md Android 3 에서 값을 확인한 뒤의 **재호출부터** 넘긴다. 사용자에게 억지로 묻지 않는다.
   - 대상이 확실하지 않으면 **사용자에게 "이 앱을 App Store 에도 올릴 계획인가요?" 로 한 번 확인**한다(Apple Developer Program 은 연 $99이므로 추측으로 Apple 크레덴셜 등록을 시키지 않는다). Android 만 할 계획이면 `platform: "android"` 로 호출하면 되고, 그러면 최상위 판정에 Apple 상태가 섞이지 않는다.
2. **로컬 상태**: `Bash`로 아래를 확인한다(값만 읽고 판단은 first-time-setup.md에 맡긴다).

   > **점검은 절대 절차를 멈추지 않는다.** 도구가 없거나 저장소 상태가 달라 명령이 실패해도 **중단하지 말고** 그 항목만 "확인 못 함"으로 기록하고 다음 점검으로 넘어간다. 아래 명령은 모두 미설치·미설정 상태에서도 정상 종료하도록 쓰여 있다 — 임의로 바꾸지 말 것.

   ```bash
   uname -s                                   # 맥 여부(routing.md) — 윈도는 Git Bash 에서 MINGW*/MSYS*
   ls -d ios android 2>/dev/null || echo "스캐폴드 없음"
   for c in brew xcodebuild fastlane ruby pod flutter docker gh openssl jq node magick; do
     command -v "$c" >/dev/null 2>&1 && echo "$c: 있음" || echo "$c: 없음"
   done
   XV=$(xcodebuild -version 2>/dev/null | head -1); [ -n "$XV" ] && echo "Xcode: $XV" || echo "Xcode: 없음(또는 Command Line Tools 만 있음)"
   ruby -rxcodeproj -e 'exit 0' 2>/dev/null && echo "xcodeproj: 있음" || echo "xcodeproj: 없음"
   docker info >/dev/null 2>&1 && echo "docker 데몬: 실행 중" || echo "docker 데몬: 꺼짐 또는 없음"
   docker image inspect web-game-android --format 'web-game-android: {{.Architecture}}' 2>/dev/null || echo "web-game-android: 없음"
   grep -hs applicationId android/app/build.gradle android/app/build.gradle.kts
   grep -hs appId capacitor.config.json capacitor.config.ts capacitor.config.js
   true                                       # 패키지명은 없을 수도 있다 — 여기서 멈추지 않는다
   git status --short 2>/dev/null || echo "git 저장소 아님"
   UP=$(git rev-parse --abbrev-ref --symbolic-full-name '@{u}' 2>/dev/null \
        || git symbolic-ref --short -q refs/remotes/origin/HEAD 2>/dev/null)
   if [ -n "$UP" ]; then git log --oneline "$UP"..HEAD; else echo "UNPUSHED_UNKNOWN"; fi
   ```

   읽는 법:
   - **스캐폴드**(`ios/`·`android/`) 존재 → 스캐폴드 단계 축소(first-time-setup.md 판정 규칙).
   - **toolchain**: iOS는 Xcode(`xcodebuild -version` 이 버전을 내야 한다 — Command Line Tools 만으로는 안 된다)·`fastlane`·xcodeproj, projectType 에 따라 `node`(web-capacitor)·`pod`(native-ios). Android는 docker 데몬 + `web-game-android` 이미지 — **`amd64` 여야 한다**(`arm64` 면 다시 빌드한다). 공통으로 **`openssl`**(인증서 저장소의 암호화 파일을 푼다 — 맥 기본 LibreSSL·Homebrew OpenSSL·Git for Windows 의 openssl 모두 된다). `flutter` 는 flutter 프로젝트만, `magick`(ImageMagick)은 아이콘·스크린샷 프레임을 만들 때만 필요하다. `jq` 는 없어도 된다(Android SA 의 `client_email` 을 뽑을 때 `node` 로 대신한다). **`node` 는 Android owner 에게 필수**다 — `register_android_credentials` 스크립트가 키 파일을 `node` 로 확인한다(에이전트 설치판에 node 가 따라온다고 가정하지 않는다). 점검 단계에서는 결과만 기록하고 멈추지 않는다 — **빠진 것은 로컬 선행 단계의 빌드 환경 준비(first-time-setup.md iOS ② / Android 2)에서 「빌드 환경 설치」대로 에이전트가 설치한다.** 설치 방법만 알려 주고 사용자에게 맡기지 않는다.
   - **패키지명**(`applicationId`/`appId`): 값이 나오면 이후 `get_setup_status({ platform: "android", packageName })`·`setup_android_keystore`·`start_build` 에 **그대로 쓴다**. 아무것도 안 나오면 "확인 못 함"으로 두고 넘어간다 — 스캐폴드 전에는 없는 것이 정상이며, 이 단계에서 사용자에게 묻지 않는다(first-time-setup.md Android 3 에서 정한다).
   - **`gh`**: 인증서 저장소(private) clone 자격 확인용. 없으면 git credential helper로도 되므로 "확인 못 함"으로 둔다.
   - **미push**: `UNPUSHED_UNKNOWN`이면 업스트림도 `origin/HEAD`도 없다는 뜻이다 — **막지 말고** "미push 여부를 확인하지 못했습니다. push가 안 되어 있으면 이전 코드로 빌드됩니다"라고 알린 뒤 진행한다. 커밋 목록이 나오면 push를 권한다.
3. 두 결과를 합쳐 사용자에게 **지금 남은 일**과 **확인하지 못한 항목**을 한 문단으로 알린 뒤 3단계로 간다.

## 3. 역할 분기

역할은 **둘**이다(`get_setup_status` 의 `team.role`):

- **owner** — 프로젝트의 **공유 자산을 만들고 바꾼다**(Apple 팀 키·Play Service Account·인증서 저장소·keystore·프로젝트 비밀번호).
- **협업자(`collaborator`)** — 그 밖의 전부. **빌드·스토어 업로드·테스터·스토어 메타데이터까지 직접 한다.** 협업자로 초대하는 것은 곧 이 프로젝트의 배포 권한을 주는 일이다.

### 협업자 (`team.role: "collaborator"`)

**할 수 있는 것은 끝까지 하고, owner 전용 단계에서만 멈춘다.** "협업자니까" 를 이유로 조기 종료하지 않는다.

owner 전용 도구는 **21개 중 4개**다(서버 `requireOwner` 가드 기준 — `grep -rn 'requireOwner' src/mcp/tools/` 와 1:1로 맞춘 목록이다). 넷 다 **공유 자산을 만들거나 바꾸는** 도구다. 도구 하나가 한 행이다:

| 도구 | 단계 | 이유 |
|---|---|---|
| `register_ios_credentials` | ⑥ Apple 크레덴셜 등록 | Apple 팀 키(.p8)를 인증서 저장소에 넣고 프로젝트 비밀번호를 정한다 |
| `setup_match_repo` | ⑧ 인증서 저장소 초기화(iOS) | 인증서 발급·`forceNuke` 가 프로젝트 전체의 코드서명 자산을 바꾼다 |
| `setup_android_keystore` | Android 5 | keystore 는 인증서 저장소에 사는 프로젝트 공유 자산 |
| `register_android_credentials` | Android 6 | Play Service Account 를 인증서 저장소에 넣는다 |

**이 표에 없는 도구는 협업자도 호출한다** — `analyze_repository`·`get_setup_status`·`setup_project`·`register_bundle_id`·`add_provisioning_profile`·`start_build`·`get_build_status`·`report_build_result`·`upload_to_store`·`add_testers`·`apply_ios_metadata`·`pull_ios_metadata`·`apply_android_metadata`·`pull_android_metadata`·`generate_app_icon`·`generate_ios_screenshots`·`generate_native_screenshots`. 막혀 있다고 가정하지 말 것.

진행 방법:

1. **로컬 선행 단계**(first-time-setup.md ①–⑤ / Android 1–4)를 먼저 수행한다 — 저장소 분석, 빌드 환경 준비(빠진 도구는 에이전트가 설치 — first-time-setup.md 「빌드 환경 설치」), 앱 정보 확인, 스캐폴드, 커밋 push. 역할 제한이 없는 구간이다.
2. **`nextStep` 루프를 owner 와 똑같이 돈다** — 단계를 하나 끝낼 때마다 `get_setup_status`를 다시 호출해 다음 `nextStep`을 받는다. 막힐 때까지 계속 진행한다.
   - `register_bundle_id`(⑦) / `start_build`(빌드) → **직접 수행한다.** ⑦ 다음의 ASC 웹 앱 레코드 생성(수동)은 **owner 의 App Store Connect 팀에 앱을 만들 수 있는 역할(App Manager 이상)로 초대돼 있어야** 한다 — 초대가 없으면 owner 에게 앱 레코드 생성만 요청하고 그동안 다음 단계로 간다(앱 레코드는 업로드 전에만 있으면 된다).
   - `register_ios_credentials`(⑥) / `setup_match_repo`(⑧) → **owner 전용이라 여기서 멈춘다.** 3을 따른다.
   - 위 표에 없는데도 `OWNER_ONLY` 가 오면 표가 서버와 어긋난 것이다. 안내를 지어내지 말고 응답의 `message` 를 그대로 전달한다.
3. **멈출 때**: 서버가 준 `nextStepHint`를 **그대로 전달한다** — 무엇을 누가 해야 하는지가 이미 그 문장에 들어 있다(문구를 다시 쓰지 말 것). 여기에 두 가지만 덧붙인다:
   - 지금까지 **이미 끝낸 준비**를 요약한다(owner 가 중복으로 하지 않도록).
   - "owner 가 그 단계를 마치면 `/upplz:init`(Claude Code) 또는 `$upplz-init`(Codex) 을 다시 실행하세요" — 남은 단계부터 이어서 재개된다.
   - owner 전용 도구를 호출해 `OWNER_ONLY`(`{ error, message }`)를 받았을 때도 같게 처리한다.
4. **`setupComplete: true`로 끝나면** 빌드부터 업로드까지 이어 간다:
   > "이 프로젝트는 배포 준비가 끝났습니다. 이제 '배포하자' 로 빌드를 시작할 수 있습니다."
5. **인증서 저장소 접근 권한**: 모든 로컬 스크립트는 owner 소유 private GitHub 저장소를 사용자 본인 git 자격으로 clone 한다. upplz 는 이 권한을 부여할 수도 확인할 수도 없어 **판정에 넣지 않는다** — 서버가 `nextStepHint` 끝에 붙여 보내는 안내를 **그대로 전달**하고 다시 쓰지 않는다.
6. **빌드 다음도 역할로 갈리지 않는다.** 스크립트를 돌려주는 모든 도구의 응답 `env` 에는 **프로젝트 비밀번호 `MATCH_PASSWORD` 하나**만 실린다. 사용자 셸에서 그 값을 export 한 뒤 응답의 스크립트를 **통째로** 실행하면, 스크립트가 인증서 저장소를 임시 디렉토리에 받아 필요한 파일(Apple API 키 `.p8`·Play Service Account JSON·keystore)만 풀어 쓰고 끝나면 지운다. `start_build` 는 `.p8` 를 풀지도 않는다(iOS match 가 `--readonly`).
   - 스크립트가 `[fetch] 인증서 저장소에 … 가 없습니다` 로 멈추면 owner 가 그 메시지가 지목한 등록 도구를 아직 실행하지 않은 것이다 — 문구를 그대로 전달해 owner 에게 요청한다.
   - `[decrypt]` 로 멈추면 먼저 응답의 `env.MATCH_PASSWORD` 를 **그대로** export 했는지 확인하고 다시 실행한다(셸에 남은 다른 값이 흔한 원인이다 — 값을 추측해 바꾸지 않는다). 그래도 멈추면 응답 `troubleshooting.passwordMismatch` 를 그대로 전달한다 — owner 가 그 파일의 등록 도구(`.p8` → `register_ios_credentials`, Play SA → `register_android_credentials`)를 다시 실행해야 하고, 같은 인증서 저장소를 다른 upplz 프로젝트도 쓰면 재실행 대신 대시보드 '다른 프로젝트에서 복사' 로 비밀번호를 맞춘다.
7. **Android 의 역할 분기**: iOS 와 같다 — 조건은 인증서 저장소(owner 소유 private GitHub)의 **collaborator** + **활성 멤버십** 두 가지이고, 협업자도 업로드·스토어 메타데이터까지 한다. keystore 는 그 저장소의 `android/<packageName>.keystore` 에, Play Service Account 는 같은 저장소의 `upplz/android/service-account.json.enc` 에 있으므로 협업자 로컬에 둘 파일은 없다.
   - 인증서 저장소에 keystore 가 없으면 `start_build` 의 buildScript 가 멈추고 "owner 가 `setup_android_keystore` 를 실행해야 한다"고 알려 준다 — **그 문구를 그대로 전달**한다.
   - Play Service Account 가 미등록이면 `upload_to_store`·`apply_android_metadata`·`pull_android_metadata` 가 `PRECONDITION_FAILED` 로 막는다 — owner 에게 `register_android_credentials` 실행을 요청하고 그 단계만 대기한다.
   - 저장소 접근이 거부되면(clone 실패) owner 에게 **GitHub collaborator 추가**를 요청한다. upplz 는 이 권한을 부여할 수도 확인할 수도 없다.
   - **owner 에게 요청할 때는 한 번에 묶는다.** Android 의 owner 전용 단계는 둘이고 GitHub 권한까지 셋인데, 하나씩 요청하면 빌드에서 막히고 업로드에서 또 막혀 왕복이 그만큼 늘어난다. 요청문에 **① `setup_android_keystore({ packageName })` ② `register_android_credentials({ serviceAccountPath, clientEmail })` ③ 인증서 저장소에 나를 GitHub collaborator 로 추가** 를 함께 적는다. 그동안 로컬 선행 단계(Android 1–4)를 마쳐 둔다. owner 가 끝내면 **첫 빌드부터 재개**한다.
   - **keystore 비밀번호를 따로 정하지 않는다.** keystore 비밀번호는 프로젝트 비밀번호와 **같은 값**이고, `start_build` 응답 `env.MATCH_PASSWORD` 로 자동 전달된다.
   - **Android 도 iOS 와 같은 `nextStep` 루프(위 2)를 돈다** — `get_setup_status({ platform: "android", packageName })` 로 호출하면 최상위 `nextStep`·`nextStepHint`·`setupComplete` 가 Android 것으로 온다(`packageName` 은 선택 — 알면 넘겨서 요청문에 패키지명이 찍히게 하고, 모르면 생략한다). `setup_android_keystore`·`register_android_credentials` 에서 멈추면 **3 을 따른다** — 서버 힌트가 이미 **둘을 한 번에 요청하라**고 말하므로 문구를 다시 쓰지 말고 그대로 전달한다.

### owner (`team.role: "owner"`)

`<플러그인 루트>/references/first-time-setup.md` 의 절차를 **§0부터 끝까지** 수행한다. 단계마다 `get_setup_status`를 다시 호출해 `nextStep`으로 진행 지점을 잡는다.

## 4. 값 수집 — 파일은 경로만, 식별자는 대화로

이 절은 **owner 가 공유 자산을 등록하는 경로**다. 등록 도구는 파일 **본문을 받지 않는다** — 파일의 **절대 경로**를 받아, 사용자 머신에서 파일을 암호화해 인증서 저장소에 push 하는 스크립트를 돌려준다. 그래서 `.p8`·Service Account JSON 의 본문은 대화 기록·서버 어디에도 들어가지 않는다.

수집 대상(iOS): **.p8 파일의 절대 경로 · Key ID · Issuer ID · Team ID · (아직 없을 때만) 인증서 저장소 URL · (최초 1회) 프로젝트 비밀번호(8자 이상) · Bundle ID · 앱 이름**. Android: **패키지명 · Service Account 키 파일의 절대 경로 · 그 파일의 `client_email` · 인증서 저장소 URL(Android 전용 프로젝트만 — iOS 를 이미 설정했으면 같은 저장소가 자동으로 쓰인다) · (최초 1회) 프로젝트 비밀번호**.

**이미 저장된 값은 다시 받지 않는다.** `get_setup_status` 의 `apple.matchPasswordStored: true` 면 프로젝트 비밀번호는 **묻지도 넘기지도 않는다** — 저장값과 다른 값은 `PRECONDITION_FAILED` 로 거부되고(바꾸면 인증서 저장소의 모든 파일이 풀리지 않는다), 같은 값이면 받을 이유가 없다. 인증서 저장소 URL 도 이미 있으면(`apple.matchRepoUrl`·`android.certRepoUrl`) 묻지 않고 `matchRepoUrl` 을 뺀다(넘기면 저장값을 덮어쓴다).

값을 묻기 전에 **반드시 아래를 그대로 안내**한다:

> `.p8` 키와 Play Service Account JSON 은 **파일의 절대 경로만** 알려 주세요(예: `/Users/me/Downloads/AuthKey_ABC123DEF4.p8`). 제가 파일을 열어 읽지 않고, 반환되는 스크립트가 이 머신에서 직접 암호화해 인증서 저장소에 넣습니다 — 키 본문은 대화 기록에도 upplz 서버에도 남지 않습니다.
> Key ID·Issuer ID·Team ID·저장소 URL 은 비밀이 아니므로 여기에 입력해 주세요. **프로젝트 비밀번호**(최초 1회)는 도구에 넘겨야 하므로 대화 기록에 남습니다 — 비밀번호 관리자에도 함께 보관하세요.

- **`.p8`·SA JSON 파일은 `Read` 로 읽지 않는다.** 경로만 다룬다. 경로에 한글 등 비 ASCII 문자가 있으면 도구가 거부하므로(스크립트에 그대로 들어가는 값이라 허용 문자를 좁혔다) 파일을 영문 경로로 옮겨 달라고 안내한다 — **사용자 이름이 한글이면 `~/Downloads` 도 한글 경로**이므로(윈도에 흔하다: `C:\Users\홍길동`) 윈도 Git Bash 는 `/c/upplz/`, 맥은 `/Users/Shared/upplz/` 같은 사용자 폴더 밖 영문 폴더를 예로 든다.
- **`client_email`** 은 공개 식별자다. `jq -r .client_email <파일>` 로 뽑고, `jq` 가 없으면 `node -e 'console.log(require(process.argv[1]).client_email)' <파일>` 을 쓴다 — 두 명령 모두 이메일 한 줄만 출력하므로 키 본문이 대화에 올라오지 않는다.
- 어느 경로든 **받은 값을 요약·복창하지 않는다**. 저장 후에는 "등록 스크립트를 실행하면 끝납니다"만 보고한다(도구 응답도 입력값을 되돌려 주지 않는다).

### 저장

- **iOS 크레덴셜**: `register_ios_credentials({ authKeyPath, keyId, issuerId, teamId, matchRepoUrl?, matchPassword? })` — **owner 전용**. `matchRepoUrl` 은 프로젝트에 아직 없을 때만(최초 등록) 넘긴다 — 키 교체·Android 를 먼저 설정한 프로젝트는 빼면 저장값이 쓰인다. 식별자는 프로젝트에 저장하고, 응답의 `script` 를 `env.MATCH_PASSWORD` 와 함께 실행하면 `.p8` 가 인증서 저장소의 `upplz/ios/AuthKey.p8.enc` 로 암호화돼 push 된다. **프로젝트에 비밀번호가 아직 없으면 `matchPassword` 는 필수**다(없으면 `PRECONDITION_FAILED`). 키 교체도 같은 도구를 다시 실행한다 — 응답 `replaced: true`(저장돼 있던 Key ID 가 이번 값과 다를 때만)와 `previousKeyId` 가 오면 옛 키를 App Store Connect 에서 Revoke 하라고 안내한다. 같은 Key ID 로 다시 실행하면 `replaced: false` 다(기존 프로젝트 이관 등).
- **Android keystore**: `setup_android_keystore({ packageName, matchRepoUrl?, matchPassword? })` — **owner 전용**. keystore 를 인증서 저장소에 준비하는 스크립트를 받는다. keystore 비밀번호는 프로젝트 비밀번호와 **같은 값**이다. **Android 전용 프로젝트의 첫 셋업**(저장소 URL·프로젝트 비밀번호가 아직 없음)이면 `matchRepoUrl`·`matchPassword` 를 함께 넘겨 셋을 한 번에 준비한다 — Play Service Account 없이도 첫 AAB 를 만들 수 있다. 이미 저장돼 있으면 둘 다 뺀다(비밀번호 없이 부르면 `PRECONDITION_FAILED`, 저장값과 다르면 거부).
- **Android Play 크레덴셜**: `register_android_credentials({ serviceAccountPath, clientEmail, matchRepoUrl?, matchPassword? })` — **owner 전용**(`matchRepoUrl`·`matchPassword` 는 아직 없을 때만 — keystore 를 먼저 준비했다면 뺀다). `client_email` 은 프로젝트에 저장하고(이 값의 존재가 "SA 등록됨" 이다), 응답의 `script` 가 키 파일을 `upplz/android/service-account.json.enc` 로 암호화해 push 한다. 교체 시 스크립트가 옛 `private_key_id` 를 출력하므로 Google Cloud 에서 그 키를 삭제하라고 안내한다. SA 는 새면 Google Cloud 에서 키를 회전하는 것 말고는 수습할 길이 없다.
- 입력 형식 오류는 `INVALID_INPUT`의 `errors`(필드별 안내)로 돌아온다 — 해당 값만 다시 받아 재호출한다.
- 등록 스크립트를 같은 파일로 두 번 실행하면 두 번째는 `[done] 변경 없음` 으로 끝난다(커밋·push 없음) — 실패가 아니다.

## 5. 완료 확인·종료

1. **iOS**: `get_setup_status`를 다시 호출해 `setupComplete: true`를 확인한다(아니면 `nextStep`으로 남은 단계를 이어서 수행).
2. **Android**: `get_setup_status({ platform: "android", packageName }).setupComplete` 로 확인한다(아니면 최상위 `nextStep`으로 남은 단계를 이어서 수행 — iOS 와 같은 루프다).
   - **`android.buildSucceeded: false` 면 keystore 파일은 미확인 상태다.** 서버는 인증서 저장소 안을 볼 수 없어 `setup_android_keystore` 발급 기록으로 추정했을 뿐이므로, **스크립트를 실제로 실행해 마쳤는지 사용자에게 한 번 확인**한다(최종 확인은 첫 빌드의 buildScript 가 한다).
3. **iOS+Android 프로젝트**: 한 번의 호출로 둘 다 본다 — 최상위는 `platform` 이 고른 쪽이고, 다른 쪽은 응답의 `ios`/`android` 블록에 있다.
4. 종료 안내:
   > "배포 설정이 끝났습니다. 이제 '배포하자' 또는 '테스트 빌드 올려줘' 로 빌드를 시작할 수 있습니다."
   > "다른 프로젝트가 있으면 대시보드 프로젝트 페이지의 **'다른 프로젝트에서 복사'** 로 식별자와 비밀번호를 옮길 수 있습니다(같은 인증서 저장소를 쓸 때)."

## 6. 알려진 한계 (1회 안내)

- **API 키는 설치당 1개다.** 키 하나가 프로젝트 하나에 묶이므로, 다른 프로젝트를 배포하려면 **그 프로젝트의 키로 바꿔 끼워야** 한다 — Claude Code 는 `/plugin` → upplz 설정에서, Codex 는 `~/.codex/.env` 의 `UPPLZ_API_KEY` 값을 바꾸고 재시작한다. 여러 프로젝트를 동시에 연결할 수는 없다.
- **협업자는 인증서 저장소의 GitHub collaborator 여야 한다(iOS·Android 공통).** 모든 파일 자산이 그 private 저장소에 있으므로, owner 가 GitHub 에서 직접 collaborator 로 추가해야 한다 — upplz 가 대신 부여하거나 확인할 수 없다.
- **서버가 확인할 수 없는 것 — 인증서 저장소 접근 권한·저장소 안 파일(.enc·keystore)의 실존·ASC 앱 레코드·프로비저닝 프로파일 — 은 판정이 아니라 힌트로만 온다.** `setupComplete: true` 여도 이것들은 남아 있을 수 있으며, 스크립트가 실행 중에 최종 확인한다(확인할 수 없는 것을 판정에 넣으면 준비가 끝났는데도 막는 거짓 음성이 나기 때문이다). Play Console 앱 생성(최초 AAB 수동 업로드)도 같은 이유로 판정 밖이다.
- **프로젝트 비밀번호는 저장소 전체의 열쇠다.** 인증서·프로파일·keystore·`.p8`·SA 가 모두 이 값으로 잠긴다. 분실하면 upplz 가 복구할 수 없고(서버는 저장된 값을 응답으로만 전달한다), 협업자를 내보낼 때 막는 것은 비밀번호가 아니라 **GitHub collaborator 제거**다(대시보드 이탈 체크리스트가 안내한다).
- **윈도에서는 iOS 를 빌드할 수 없다**(맥 전용 — 윈도 iOS 는 후속). Android 는 Git Bash + Docker Desktop 에서 같은 스크립트를 쓴다(**윈도 실측 전 — 미검증**. 경로 오류로 멈추면 출력을 그대로 보여 주고 upplz 에 알려 달라고 안내한다).

## 7. 다음 단계 제안

`<플러그인 루트>/references/chaining.md` 의 제안 문구 규칙대로 **물어본다**: "지금 이어서 테스트 빌드를 올려볼까요?"(`upplz-upload-build-only`). 사용자가 원치 않으면 그대로 종료한다.
