# Changelog

본 reusable workflows의 변경사항을 기록합니다. semver 준수.

## [Unreleased]

## [1.8.1] - 2026-05-15

### Fixed
- `android-prod.yml`: `.gha` checkout 추가 + composite action 경로 `./.github/...` → `./.gha/.github/...` 수정 — `android-dev.yml` v1.8.0과 동일한 "Can't find action" 버그가 prod에 잔존하던 것을 평행 적용. 키스토어 `if:` 시크릿 평가 버그는 prod에 해당 없음(키스토어 secret이 `required: true`로 의무화).

### Changed
- 모든 워크플로우(`ios-dev.yml`, `ios-prod.yml`, `android-dev.yml`, `android-prod.yml`)의 `DEPLOY_WF_REF`를 `refs/tags/v1.8.1`로 동기화 — v1.5.0 정책("신규 릴리즈 시 이 값만 갱신") 일관성 회복.

### Docs
- `README.md`: 호출 예시 ref `@v1.6.2` → `@v1.8.1` 갱신, 호환성 테이블에 v1.7.0 / v1.8.0 / v1.8.1 행 추가 (v1.7.0~v1.8.0 발행 시 누락된 룰 보정)
- `SETUP.md`: Android 자동배포 가이드 추가 — `3-B. Android fastlane 파일` 섹션(Gemfile / Pluginfile / Fastfile), 4단계 `deploy-android-dev.yml` thin caller 예시(`@v1.8.1`), 5단계 Secrets Android 섹션, 6단계 `.gitignore` Android 항목, 8단계 Android 태그 형식·배포 순서
- `SETUP.md`: 9단계 — CI (선택) 신규 — `flutter analyze` + `very_good test` 템플릿, envied 사용/미사용 분기 명시
- `SETUP.md`: Known Issues 테이블에 v1.8.0/v1.8.1 Android 버그픽스 행 추가 (`.gha` checkout 누락 / 릴리즈 키스토어 단계 항상 스킵)
- `SETUP.md`: 버전 호환성 표에 v1.8.0 / v1.8.1 행 추가

## [1.8.0] - 2026-05-13

### Fixed
- `android-dev.yml`: `.gha` checkout 추가 + composite action 경로 `./.github/...` → `./.gha/.github/...` 수정 — reusable workflow에서 caller 레포 경로로 composite action을 찾아 "Can't find action" 오류 수정
- `android-dev.yml`: 릴리즈 키스토어 조건부 단계를 `if:` 대신 run 내부 조건으로 변경 — 시크릿은 `env:` 블록 평가 전 `if:` 조건에서 접근 불가해 항상 스킵되던 버그 수정

### Added
- `android-dev.yml`: `target` input 추가 — 분리 진입점(`lib/main_dev.dart` 등) Flutter 프로젝트 지원
- `android-dev.yml`: `--split-per-abi` 기본 적용 — arm64-v8a APK 우선 선택으로 APK 크기 최적화
- `android-dev.yml`: Fastlane 단계에 `FIREBASE_APP_ID`, `TESTER_GROUPS` env 추가 — `dev_release` 레인에서 동적으로 참조 가능

## [1.7.0] - 2026-05-08

### Added
- `ios-dev.yml`, `ios-prod.yml`: `Select Xcode` 단계 추가 — `maxim-lobanov/setup-xcode@v1`로 `latest-stable`(Xcode 26.x) 선택. Apple App Store Connect가 iOS 26 SDK(Xcode 26+)로 빌드된 앱만 허용함에 따라 기본값인 Xcode 16.4로는 업로드 시 "SDK version issue" 오류 발생

## [1.6.3] - 2026-05-08

### Fixed
- `ios-dev.yml`, `ios-prod.yml`: `Install FlutterFire CLI` 단계 추가 — `firebase_crashlytics`가 `pubspec.yaml`에 선언된 경우 `dart pub global activate flutterfire_cli` 실행. Xcode 빌드 단계(`flutterfire upload-crashlytics-symbols`)가 CI 러너에서 `flutterfire: command not found`로 `ARCHIVE FAILED` 오류 수정

## [1.6.2] - 2026-05-08

### Fixed
- `ios-dev.yml`, `ios-prod.yml`: `Setup CocoaPods` 단계를 `bundle exec pod install`로 변경 — `ruby/setup-ruby`가 `bundle config set --local path ios/vendor/bundle`을 설정해 `bundle exec fastlane` 실행 시 `GEM_HOME`이 bundle 경로로 제한됨. `cocoapods`가 `ios/Gemfile`에 없으면 `pod`가 gem을 찾지 못해 "CocoaPods is installed but broken" 오류 발생. 호출 앱 `ios/Gemfile`에 `gem "cocoapods"` 추가 필요 (sanches37_deploy v0.2.0+ 레인 사용 시)

## [1.6.1] - 2026-05-08

### Fixed
- `ios-dev.yml`, `ios-prod.yml`: `Setup CocoaPods` 단계를 `Setup Ruby` **이후**로 이동 — setup-ruby가 PATH를 바꾼 후 기존 Ruby로 설치된 pod가 버전 불일치로 깨지는 문제 수정
- `ios-dev.yml`, `ios-prod.yml`: `pod --version` 실패 시 homebrew 대신 `gem install cocoapods` 사용 — 현재 활성 Ruby(3.3.x)에 cocoapods 재설치

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
