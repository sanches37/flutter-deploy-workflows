# Changelog

본 reusable workflows의 변경사항을 기록합니다. semver 준수.

## [Unreleased]

## [1.5.7] - 2026-05-07

### Fixed
- `setup-ios-signing`: `fastlane match --readonly` 완료 후 설치된 `.mobileprovision` 파일에서 실제 프로필 이름 추출 → `MATCH_PROFILE_NAME`으로 `$GITHUB_ENV` export
- `ios-dev.yml`, `ios-prod.yml`: "Set Manual Signing" 단계에서 하드코딩된 `"match AppStore <bundle_id>"` 대신 `$MATCH_PROFILE_NAME` 사용 — fastlane match가 timestamp를 붙인 이름(예: `match AppStore com.lifeadmin.mobile 1778134700`)과 Xcode 프로필 검색이 일치하지 않아 발생하는 "No profile found" 오류 수정
- `setup-ios-signing` ExportOptions.plist도 동일하게 실제 이름 사용 (`MATCH_PROFILE_NAME` 우선, 없으면 기존 컨벤션 폴백)

## [1.5.6] - 2026-05-08

### Fixed
- `ios-dev.yml`, `ios-prod.yml`: `FLUTTER_XCODE_*` 제거 — Pod 타겟 전체에 `PROVISIONING_PROFILE_SPECIFIER` 전파 문제 수정
- `ios-dev.yml`, `ios-prod.yml`: Setup iOS signing 후 `perl`+`Ruby`로 Runner.xcodeproj/project.pbxproj 직접 수정 — Runner 타겟에만 수동 서명 주입 (baedalpan 패턴)

## [1.5.5] - 2026-05-07

### Fixed
- `ios-dev.yml`, `ios-prod.yml`: `flutter build ipa` 시 `FLUTTER_XCODE_CODE_SIGN_STYLE=Manual` + `PROVISIONING_PROFILE_SPECIFIER` + `CODE_SIGN_IDENTITY` 주입 — Xcode 자동 서명 시도로 인한 "No Accounts" 오류 수정

## [1.5.4] - 2026-05-07

### Fixed
- `setup-ios-signing`: `MATCH_GIT_BASIC_AUTHORIZATION`에 raw PAT 대신 `base64("x-access-token:<PAT>")` 사용 — HTTP 400 git clone 오류 수정

## [1.5.3] - 2026-05-07

### Fixed
- `ios-dev.yml`, `ios-prod.yml`: env 파일 디코딩을 composite action input에서 workflow 직접 step으로 이동 — composite action `if:` 조건에서 secret 값이 비어있는 것으로 평가되는 GitHub Actions 동작 우회

## [1.5.2] - 2026-05-07

### Fixed
- `setup-ruby`: `bundler: latest` 추가 — Ruby 3.2+에서 `untaint` 제거로 Bundler 1.17.2 실패하는 문제 수정 (Gemfile.lock `BUNDLED WITH` 버전 무시하고 최신 Bundler 사용)

## [1.5.1] - 2026-05-07

### Fixed
- `verify-version`: pubspec 버전의 `+N` 빌드번호를 태그 비교 전에 제거 — 태그 `v1.0.3-dev-ios`와 pubspec `1.0.3+1`이 정상 매치됨

## [1.5.0] - 2026-05-07

### Fixed
- `ios-dev.yml`, `ios-prod.yml`: `github.workflow_ref`도 호출 repo ref를 반환하는 문제 최종 수정 — `DEPLOY_WF_REF: refs/tags/v1.5.0` 하드코딩으로 대체. 신규 릴리즈 시 이 값만 갱신.

## [1.4.0] - 2026-05-07

### Fixed
- `ios-dev.yml`, `ios-prod.yml`: `github.workflow_sha`가 호출 repo SHA로 해석되는 문제 수정 — `github.workflow_ref`에서 `@` 뒤를 잘라 정확한 ref(예: `refs/tags/v1.4.0`) 추출
- `setup-flutter` composite action: `env-stub-file` 입력 추가 — dev 빌드에서 `.env.prod` 미존재로 build_runner 실패하는 문제 해결 (파일 없으면 primary env 파일 복사)
- `ios-dev.yml`: setup-flutter에 `env-stub-file: '.env.prod'` 전달
- `ios-prod.yml`: `ENV_RELEASE`→`ENV_PROD`, `.env.release`→`.env.prod`로 통일; `env-stub-file: '.env.dev'` 전달

## [1.3.0] - 2026-05-07

### Fixed
- `ios-dev.yml`, `ios-prod.yml`: reusable workflow에서 composite action 상대 경로(`./.github/actions/xxx`)가 호출 repo 워크스페이스를 참조하는 문제 수정 — `github.workflow_sha`로 flutter-deploy-workflows 자신을 `.gha/`에 체크아웃 후 절대 경로로 참조

## [1.2.0] - 2026-05-07

### Changed
- `ios-dev.yml`, `ios-prod.yml`: `flavor` input 기본값 `''`으로 변경 — 값이 없으면 `--flavor` 인수 없이 빌드 (Flutter flavor 미사용 앱 호환)

## [1.1.0] - 2026-05-01

### Added
- Composite action `setup-flutter`: Flutter 설치 + pub get + .env 디코딩 + build_runner
- Composite action `setup-ruby`: Ruby 셋업 + private gem git auth + bundler-cache (auth → install 순서 교정으로 첫 실행 lock 없을 때 실패 방지)
- Composite action `setup-ios-signing`: fastlane match readonly + ExportOptions.plist 생성 (match-type 분기)
- Composite action `verify-version`: tag ↔ pubspec.yaml 버전 일치 검증 + main 브랜치 강제 옵션 (`enforce-main`), `outputs.version` 제공

### Changed
- `ios-prod.yml`, `ios-dev.yml`, `android-prod.yml`, `android-dev.yml`: 반복 step 묶음을 4개 composite으로 치환

### 호환성
- 페어 plugin: `fastlane-plugin-sanches37_deploy@~> 0.1`
- Flutter: `>= 3.32.0`
- Java (Android): `17`

## [1.0.0] - 2026-04-30

### Added
- `ios-dev.yml`: TestFlight Internal Testing 업로드 (workflow_call)
- `ios-prod.yml`: TestFlight 업로드 + App Store 메타데이터 + 심사 제출 (workflow_call)
- `android-dev.yml`: Firebase App Distribution 배포 (workflow_call)
- `android-prod.yml`: Play Store production track AAB 업로드 (workflow_call, changelog sync 포함)

### 호환성
- 페어 plugin: `fastlane-plugin-sanches37_deploy@~> 0.1`
- Flutter: `>= 3.32.0`
- Java (Android): `17`
