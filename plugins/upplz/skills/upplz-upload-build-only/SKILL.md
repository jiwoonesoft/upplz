---
name: upplz-upload-build-only
description: 테스트 목적으로 빌드만 만들어 스토어에 업로드할 때 사용(iOS TestFlight / Android 내부 트랙). 메타데이터·아이콘·스크린샷은 다루지 않는다. 최초 배포면 셋업(Apple/keystore)부터 자동 진행. "테스트 빌드", "TestFlight에 올려줘", "빌드만 올려줘", "다시 빌드", "새 버전 올려줘", "업데이트 배포", "처음 배포", "앱 등록", "온보딩" 요청 시 활성화. 출시 목적(메타·에셋 점검 포함)이면 upplz-upload를 쓴다.
---

## Overview
테스트용 순수 빌드→업로드→보고 오케스트레이터(iOS/Android). 최초/반복은 셋업 상태로 자동 분기한다 — 셋업 미완료면 최초 셋업 레퍼런스를 먼저 수행한다. 실제 작업은 MCP 원자 도구가 수행한다.

## When to Use
- 코드 변경 후 새 빌드를 테스트 트랙(TestFlight/내부 테스트)에 올릴 때.
- 이 저장소/앱을 upplz로 **처음** 배포할 때(셋업부터 첫 테스트 업로드까지).
- 메타데이터·아이콘·스크린샷 점검까지 포함한 **출시**가 목적이면 `upplz-upload`를 쓴다.

## Routing (첫 단계, 필수)

> 플러그인 루트 = 이 SKILL.md 가 있는 디렉토리의 두 단계 위다(`skills/upplz-upload-build-only/SKILL.md`). 아래 `<플러그인 루트>/…` 는 그 경로다.

`<플러그인 루트>/references/routing.md`로 머신·대상 확정. iOS ∧ 비맥이면 iOS 맥 전용 게이트로 중단. Android 는 맥·윈도(Git Bash + JDK + Node.js — 윈도 미검증) 모두 같은 Process 다.

## 최초 셋업 게이트 (두 번째 단계, 필수)
`<플러그인 루트>/references/first-time-setup.md`의 "셋업 완료 판정"으로 확인한다 — **iOS·Android 모두 `get_setup_status` 로 판정**하고(iOS 는 파라미터 없이, Android 는 `platform: "android"` — 패키지명을 알면 `packageName` 도 함께 넘겨 힌트의 owner 요청문에 실제 패키지명이 찍히게 한다. 선택이며 판정은 바뀌지 않는다), `setupComplete`면 Process로, 아니면 `nextStep`으로 분기한다(두 플랫폼이 같은 모양의 루프다). 미완료면 해당 플랫폼의 최초 셋업 절차를 수행한 뒤 아래 Process로 복귀한다(사용자가 `upplz-init`(Claude Code `/upplz:init` · Codex `$upplz:upplz-init`)을 아직 실행하지 않았다면 그것을 권하되, 요청하면 같은 절차를 여기서 그대로 수행한다 — 둘은 같은 문서를 쓴다). 진행 중 PRECONDITION/VERSION 계열 에러 코드가 오면 도구가 방어하는 것이므로, 지시대로 같은 문서의 해당 단계로 돌아간다(`get_setup_status` 를 같은 `platform` 으로 재호출해 복귀 지점을 잡는다).

## Process
대상=iOS면 1–5(iOS)를, 대상=Android면 1'–5'(Android)를 따른다. **역할로 갈리지 않는다** — 협업자도 빌드부터 업로드·보고까지 전부 한다.

**스크립트 실행 규칙(공통)**: 모든 스크립트 도구의 응답 `env` 는 프로젝트 비밀번호 `MATCH_PASSWORD` 하나다. 그 값을 export 한 뒤 응답의 스크립트를 **파일로 저장해 통째로** 실행하고(`bash <파일>` — 인라인 `bash -c` 는 Claude Code 안전 검사가 `rm -rf "$WORKDIR"` 때문에 차단한다), 끝나면 unset 하고 파일을 지운다. 키 파일을 직접 준비하지 않고, **파일 본문을 어떤 도구로도 대화에 읽어 들이지 않는다** — 스크립트가 저장소에서 필요한 파일(`.p8`·SA·keystore)을 풀어 쓰고 종료 시 지운다. `[fetch] … 가 없습니다` 면 owner 가 지목된 등록 도구를 실행해야 한다. `[decrypt]` 면 먼저 응답 `env.MATCH_PASSWORD` 를 그대로 export 했는지 확인하고, 맞으면 응답 `troubleshooting.passwordMismatch` 를 그대로 전달한다(owner 가 그 파일의 등록 도구를 다시 실행 — 같은 저장소를 다른 upplz 프로젝트도 쓰면 대시보드 '다른 프로젝트에서 복사' 로 비밀번호를 맞춘다).

**하네스 우회 금지(iOS 전용)**: iOS 빌드는 반드시 `start_build` 가 반환한 preflightScript/buildScript 로만 수행한다. 레포 구조 불일치·preflight 실패·서명·아카이브 오류 등으로 스크립트가 실패하면, **하네스를 우회한 임의 빌드(예: xcodebuild 직접 실행)로 대체하지 않는다.** 중단하고 실패 원인과 해결책(구조 조정 방법 또는 미지원 사실)을 사용자에게 보고하며, 하네스 밖에서 만든 산출물을 upload_to_store/report_build_result로 이어가지 않는다. **Android 는 다르다** — 빌드는 에이전트가 저장소에 맞는 방법으로 직접 한다(upplz 도구가 아니다). 빌드가 실패하면 원인과 해결을 보고하고, **서명·업로드는 반드시 upplz 스크립트(`sign_android_bundle`·`upload_to_store`)로** 한다(직접 서명한 AAB 를 올리지 않는다). **도구가 없어서 멈춘 것**(preflight 의 `✗`·`미설치` 줄, `command not found`, JDK·Node 부재)은 중단 사유가 아니다 — `<플러그인 루트>/references/first-time-setup.md` 「빌드 환경 설치」대로 **에이전트가 설치·실행하고 같은 스크립트를 다시 실행**한다(설치 방법만 알려 주고 멈추지 않는다).

### iOS
1. (수동) `git status` / `git log origin/main..HEAD`로 변경·미push 확인. **push 안 하면 이전 코드로 빌드된다.** 미push 커밋이 있으면 push.
2. 메타데이터(`ios/fastlane/metadata|screenshots/**`)만 바뀌었다면 → `apply_ios_metadata`만 하고 종료(빌드 불필요). upplz-upload에서 위임된 경우에는 종료하지 않고 upplz-upload의 다음 단계로 복귀한다.
3. `start_build` — `repoUrl` 전달(필수). projectType은 직전 빌드 값이 재사용된다(**최초 빌드이거나 native-ios/flutter/react-native인데 불확실하면 `projectType` 명시** — 미지정 시 web-capacitor로 잘못 빌드됨. 빌드 대상이 모호하면 `scheme`/`xcodeProjectPath` 지정. 스테이징 백엔드용 Release 형 구성이 따로 있으면 `configuration` 지정 — 기본은 Release 라 스킴의 아카이브 구성을 덮는다). 반환된 로컬 빌드 스크립트를 응답 `env.MATCH_PASSWORD` 와 함께 실행 → `.upplz/App.ipa` 생성. **응답에 Apple API 키(.p8)는 들어 있지 않다** — buildScript 의 match 가 `--readonly` 라 저장소의 인증서·프로파일만 받는다. 프로젝트 비밀번호가 없으면 `PRECONDITION_FAILED` 로 멈춘다(owner 가 `register_ios_credentials` 의 `matchPassword` 로 저장해야 한다).
   - marketingVersion 미입력 시 MARKETING_VERSION_REQUIRED로 되물음. App Store에 이미 출시/심사된 버전(예: 1.0)이면 **더 높은 버전**(1.0.1)을 지정(train closed 예방). 빌드 번호는 자동 +1.
   - match 가 프로파일을 찾지 못해 실패하면 이 Bundle ID 의 프로파일이 저장소에 없다는 뜻이다 → `add_provisioning_profile` 로 추가한 뒤 다시 빌드한다(활성 멤버 누구나).
4. `upload_to_store` — 로컬 pilot 업로드. 스크립트의 프롤로그가 인증서 저장소에서 팀 키(`.p8`)를 풀어 쓰므로 응답 `env.MATCH_PASSWORD` 만 export 하면 된다. VERSION_ALREADY_USED면 3으로 돌아가 marketingVersion 상향. 실패하면 응답의 `appleErrorGuidance` 로 원인을 안내한다(예: ASC 앱 레코드가 없음 → ASC 웹 My Apps 에서 앱을 만든 뒤 재시도).
5. `report_build_result` — 결과 보고(iOS이고 실패가 아니면 스토어 공개 시 서버가 iTunes Lookup으로 쇼케이스 자동 보강 — showcase 생략 가능. 플랫폼 판정은 항상 빌드 값 기준).

### Android (맥 · 윈도)
1'. (수동) `git status` / `git log origin/main..HEAD`로 변경·미push 확인. 미push 커밋이 있으면 push(`sign_android_bundle` 의 `commitSha` 기록과 맞춘다).
2'. 메타데이터(`android/fastlane/metadata/android/**`)만 바뀌었다면 → `apply_android_metadata`만 하고 종료(빌드 불필요). upplz-upload에서 위임된 경우에는 종료하지 않고 upplz-upload의 다음 단계로 복귀한다.
3'. **에이전트가 직접 빌드한다** — upplz 는 Android 를 빌드하지 않는다(`start_build` 에 `platform: "android"` 를 넘기면 `ANDROID_USE_SIGN_TOOL` 안내만 돌아온다).
   - `get_setup_status({ platform: "android", packageName })` 의 `android.nextVersionCode`(기록된 최댓값+1, 없으면 1)를 versionCode 로, 사용자와 정한 x.y.z 를 versionName 으로 `android/app/build.gradle(.kts)` 에 설정한다. Play Console 에 이미 출시/심사된 버전이면 더 높은 버전을 쓴다.
   - Capacitor 면 `npx cap add android`(최초 1회) → `npx cap sync android`. Kotlin·Flutter·RN 은 저장소의 방식대로 준비한다.
   - **release 에 `signingConfig` 를 넣지 않는다** — 서명 없는 AAB 를 만든다: `cd android && ./gradlew bundleRelease` → `android/app/build/outputs/bundle/release/app-release.aab`. JDK 21 + `JAVA_HOME`(Capacitor 8·AGP 8.13 — JDK 17 이면 `invalid source release: 21`).
   - 빌드 실패는 upplz 도구 문제가 아니다 — 원인과 해결을 사용자에게 보고한다(위 「하네스 우회 금지」의 Android 규칙).
3''. `sign_android_bundle({ packageName, repoUrl, aabPath })` — `aabPath` 기본값은 위 경로. 반환된 preflightScript(JDK jarsigner·zip·인증서 저장소 접근·비밀번호·AAB 존재 점검) → signScript 를 응답 `env.MATCH_PASSWORD` 와 함께 실행 → `.upplz/app-release.aab` 생성. **keystore 비밀번호는 프로젝트 비밀번호와 같은 값**이라 따로 받거나 정하지 않는다. 응답 `buildId` 를 4'·5' 에 쓴다. **서명이 끝나면 바로** `report_build_result({ buildId, status: "completed", versionCode, marketingVersion })` 로 보고한다 — 최초 배포처럼 4' 가 수동 업로드로 넘어가도 versionCode 가 기록돼 다음 `nextVersionCode` 가 맞는다.
   - **keystore 는 인증서 저장소(match 저장소)의 `android/<packageName>.keystore` 에서 받는다** — signScript 가 사용자 본인 git 자격으로 shallow clone 해 서명하고 끝나면 임시 디렉토리를 지운다. 협업자도 같은 스크립트를 받으며, 그 저장소의 **GitHub collaborator** 여야 한다(접근 거부 시 owner 에게 추가 요청 — upplz 가 부여할 수 없다). 저장소에 keystore 가 없으면 스크립트가 "owner 가 `setup_android_keystore` 를 실행해야 한다"며 멈춘다 — 그 문구를 그대로 전달한다. 인증서 저장소 URL 이나 프로젝트 비밀번호 자체가 없으면 도구가 `PRECONDITION_FAILED` 로 막는다(owner 가 `setup_android_keystore({ packageName, matchRepoUrl, matchPassword })`).
   - `[sign] 서명 실패` 면 응답 `troubleshooting.signFailed` 를 그대로 전달한다(다른 비밀번호로 만든 옛 keystore — `setup_android_keystore` 가 `keytool -storepasswd` 절차를 안내한다).
4'. `upload_to_store({ platform: "android", buildId, packageName?, aabPath?, track? })` — 호스트 Node 로 Google Play Developer API 를 부르는 uploadScript 를 돌려준다. 응답 `env.MATCH_PASSWORD` 를 export 한 뒤 스크립트를 **통째로** 실행하면, 프롤로그가 인증서 저장소에서 Play Service Account 를 임시 파일로 풀어 `GOOGLE_APPLICATION_CREDENTIALS` 로 node 에 **경로만** 넘기고 종료 시 지운다(패키지는 `$HOME/.upplz/play-api` 에 1회 설치 — 고객 저장소에는 설치하지 않는다). AAB 를 올리고 트랙 릴리스를 completed 로 만든다. SA 미등록이면 `PRECONDITION_FAILED` — owner 에게 `register_android_credentials` 실행을 요청한다(owner 전용). **최초 배포는 이 단계 대신 Play Console 웹에서 앱 생성 + 내부 테스트 트랙에 `.upplz/app-release.aab` 수동 업로드가 필요** — `<플러그인 루트>/references/first-time-setup.md`의 Android 절차 참고. 실패 시 응답의 `playErrorGuidance`(예: PLAY_APP_NOT_FOUND → 최초 수동 업로드가 아직 안 됐다는 뜻)로 원인을 안내한다. PLATFORM_MISMATCH/PACKAGE_NAME_REQUIRED는 도구가 방어.
5'. `report_build_result({ buildId, status: "uploaded", showcase: { platform: "android", ... } })` — 업로드 결과 보고(3'' 에서 completed 로 보고한 뒤 두 번째 보고). versionCode·marketingVersion 은 3'' 에서 이미 기록했으므로 다시 넘기지 않아도 된다(넘기면 같은 값으로 덮어쓴다). 쇼케이스는 아이콘이 있을 때만 수집, 대표 스크린샷은 선택(android는 자동 보강 미지원이므로 직접 제공).

## Verification
- **iOS**: `.upplz/App.ipa` 생성, 업로드 성공(App Store Connect 처리 중/완료), 보고 완료. Apple 처리(5–30분) 후 TestFlight에서 새 빌드 확인 가능함을 안내.
- **Android**: `.upplz/app-release.aab` 생성, 업로드 성공(최초 1회는 Play Console 웹 수동 업로드 완료), 보고 완료.

## Next
`<플러그인 루트>/references/chaining.md`에 따라 물어본다.
- 출시까지 진행? → `upplz-upload`(메타데이터·아이콘·스크린샷 점검 포함).
- iOS: 테스터 추가?(add_testers — report_build_result로 uploaded 보고가 끝난 빌드만. 테스터 관리는 팀 키의 ASC 역할이 App Manager 이상이어야 한다. 이메일은 ASC Users and Access 에 있는 **팀 사용자**만 가능하다(내부 그룹) — 아니면 ASC 에서 먼저 초대하고, 이메일을 받기 전에 이 조건을 알린다. Android 에는 테스터 도구가 없다) / 아이콘·스크린샷·메타데이터 개별 갱신?(upplz-icon/upplz-screenshot/upplz-metadata).
- Android: 메타데이터 갱신?(upplz-metadata → apply_android_metadata). 아이콘/스크린샷 자동 생성·테스터 도구는 iOS 전용.
- 스토어 심사 제출은 수동(iOS: App Store Connect 웹 Add for Review → Submit for Review, Android: Play Console 웹 트랙별 검토·출시)임을 안내한다.
