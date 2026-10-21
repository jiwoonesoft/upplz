---
name: upplz-metadata
description: 앱 스토어 메타데이터(설명·키워드·릴리스 노트·스크린샷 등)를 조회(pull)하거나 반영(apply)할 때 사용. "메타데이터 수정", "설명 바꿔줘", "릴리스 노트 업데이트" 요청 시 활성화.
---

## Overview
스토어 메타데이터를 pull(현재 값 가져오기)하고 apply(변경 반영)하는 스킬. 빌드와 독립 — 코드 변경 없이 메타만 갱신 가능.

## When to Use
- 앱 설명/키워드/릴리스 노트/스크린샷 등 스토어 리스팅만 바꿀 때.

## Routing (첫 단계, 필수)

> 플러그인 루트 = 이 SKILL.md 가 있는 디렉토리의 두 단계 위다(`skills/upplz-metadata/SKILL.md`). 아래 `<플러그인 루트>/…` 는 그 경로다.

`<플러그인 루트>/references/routing.md` 로드해 대상(iOS/Android)을 확정한다. iOS는 로컬 fastlane deliver(맥 필요, 비맥이면 iOS 맥 전용 게이트 — fastlane·ImageMagick(frameit 배경)이 없으면 에이전트가 설치하고 다시 실행한다, routing.md 「도구 미설치」). Android는 `pull_android_metadata`/`apply_android_metadata` 가 돌려주는 Node Play API 스크립트(`command` 하나)로 처리한다(맥·윈도 공통 — Node.js 필요, Docker 없음. iOS와 동일한 pull→편집→apply 흐름).

## Process

> **메타데이터 도구 4종(iOS·Android)은 활성 멤버 누구나 호출한다**(협업자 포함). 응답 `env` 는 프로젝트 비밀번호 `MATCH_PASSWORD` 하나이고, 자격(Apple 팀 키·Play Service Account)은 스크립트가 인증서 저장소에서 풀어 쓴다 — 그 값을 export 한 뒤 스크립트를 **통째로** 실행한다. iOS 메타데이터 수정은 팀 키의 ASC 역할이 **App Manager 이상**이어야 한다(Developer 로는 403).

### iOS

> Key ID·Issuer ID·Team ID 는 `register_ios_credentials` 로 등록한 값을 서버가 스크립트에 넣는다(파라미터로 받지 않는다). 키 파일을 직접 준비하지 않고, **파일 본문을 어떤 도구로도 대화에 읽어 들이지 않는다** — 스크립트가 저장소의 `upplz/ios/AuthKey.p8.enc` 를 풀어 쓰고 종료 시 지운다. 응답의 `securityNote`·`troubleshooting.keyFileNotFound` 문구를 **그대로 전달**한다(`[fetch]` 면 owner 가 `register_ios_credentials` 를 실행해야 한다).

1. `pull_ios_metadata` — 현재 스토어 메타데이터를 로컬로 가져온다(`ios/fastlane/metadata/**`).
2. 사용자가 로컬 파일 편집(설명/키워드/릴리스 노트/스크린샷 배치).
3. `apply_ios_metadata` — 변경 반영. scope 선택: 텍스트만=listing / 스크린샷만=screenshots / 둘 다=all.

### Android
> **Play Service Account 는 프롤로그가 인증서 저장소에서 풀어 쓴다** — 응답 `env.MATCH_PASSWORD` 를 export 한 뒤 응답의 command 를 **프롤로그부터 통째로** 파일로 저장해 실행한다(인라인 `bash -c` 는 Claude Code 안전 검사가 차단한다. `upplz/android/service-account.json.enc` 를 임시 파일로 풀어 `GOOGLE_APPLICATION_CREDENTIALS` 로 node 에 경로만 넘겼다가 종료 시 지운다. 로컬 키 파일은 쓰지 않는다. 패키지는 `$HOME/.upplz/play-api` 에 1회 설치한다). 미등록이면 두 도구 모두 `PRECONDITION_FAILED` — owner 에게 `register_android_credentials` 실행을 요청한다(owner 전용).

1. `pull_android_metadata({ packageName })` — 현재 Play Store 리스팅을 로컬로 가져온다(`android/fastlane/metadata/android/**`). 응답의 `command`(텍스트·이미지를 한 스크립트가 받는다)를 위 env 와 함께 사용자 머신에서 실행. 릴리스 노트는 트랙별 값이라 받지 않는다 — `changelogs/default.txt` 는 직접 작성한다.
2. 사용자가 로컬 파일 편집(제목/설명/릴리스 노트/이미지·스크린샷 배치).
3. `apply_android_metadata({ packageName, scope })` — 변경 반영. scope 선택: 텍스트만=listing / 릴리스노트만=changelog / 이미지만=images / 전체=all. 응답의 `command` 를 위 env 와 함께 사용자 머신에서 실행(prerequisites: SA 등록·Node.js(`node -v`)·메타데이터 디렉토리).
> Play 정책상 최초 업로드(앱 생성)가 아직이면 트랙에 릴리스가 없다는 에러가 날 수 있다 — `<플러그인 루트>/references/first-time-setup.md`의 Android 절차(최초 수동 업로드)가 먼저 완료돼야 한다.

## Verification
- iOS: apply 성공, App Store Connect에 변경 반영.
- Android: apply 성공, Play Console 리스팅에 변경 반영.

## Next
`<플러그인 루트>/references/chaining.md`: 메타데이터는 빌드와 독립이므로 보통 여기서 종료. 코드 변경도 있으면 upplz-upload-build-only 제안.
