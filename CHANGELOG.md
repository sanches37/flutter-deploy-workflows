# Changelog

본 reusable workflows의 변경사항을 기록합니다. semver 준수.

## [Unreleased]

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
