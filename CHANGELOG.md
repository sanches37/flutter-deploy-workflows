# Changelog

본 reusable workflows의 변경사항을 기록합니다. semver 준수.

## [Unreleased]

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
