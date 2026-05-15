# 신규 프로젝트 iOS 자동배포 설정 가이드

`sanches37/flutter-deploy-workflows` + `fastlane-plugin-sanches37_deploy`를 새 프로젝트에 연결하는 체크리스트.  
각 항목은 순서대로 완료한 뒤 다음으로 넘어간다.

---

## 사전 준비 (계정당 1회)

아래 항목은 Apple 계정·GitHub 조직 단위로 1회만 수행한다.  
이미 완료돼 있으면 건너뛴다.

- [ ] Apple Developer Program 가입 + Apple Team ID 확인  
      (Apple Developer → Membership → Team ID, 예: `626VJ36K6Q`)
- [ ] App Store Connect API Key 발급  
      (Users and Access → Integrations → App Store Connect API → + 버튼)  
      → Key ID, Issuer ID, `.p8` 파일 세 가지를 안전한 곳에 보관
- [ ] `sanches37/match-certificates` private GitHub 레포 생성  
      (인증서·프로파일 암호화 저장소. 이 레포 자체는 비어있어도 됨)
- [ ] match-certificates 레포 접근용 PAT 발급  
      (GitHub → Settings → Developer settings → Personal access tokens → `repo` 스코프)  
      → 이 토큰이 `MATCH_GIT_TOKEN` / `PLUGIN_GIT_TOKEN` secret에 사용됨

---

## 1단계 — App Store Connect 앱 등록

- [ ] App Store Connect → My Apps → `+` → Bundle ID 등록 (`com.yourapp`)
- [ ] Xcode 또는 Apple Developer 포털에서 App ID 등록 (Bundle ID 일치 확인)

> fastlane match 실행은 **3단계(fastlane 파일 설정) 완료 후** 진행한다.

---

## 2단계 — Flutter 프로젝트 구조

### 진입점 파일

**패턴 A — 표준 (`lib/main.dart`에 `main()` 있음)**  
대부분의 Flutter 프로젝트. Fastfile에서 `--target` 없이 빌드한다. 추가 작업 불필요.

**패턴 B — 분리 진입점 (`lib/main.dart`에 `main()` 없음)**  
dev/prod 환경 설정을 진입점에서 분기하는 경우. 이 구조라면 아래처럼 파일을 만들고,  
Fastfile에서 `build_flutter_args`로 `--target`을 명시해야 한다 (3단계 Fastfile 참고).

```dart
// lib/main.dart — main() 없음. 공통 초기화 헬퍼만.
Future<void> runMyApp(FlavorConfig config) async { ... }
```

```dart
// lib/main_dev.dart
Future<void> main() async {
  await runMyApp(FlavorConfig(type: FlavorType.dev, ...));
}
```

```dart
// lib/main_prod.dart
Future<void> main() async {
  await runMyApp(FlavorConfig(type: FlavorType.prod, ...));
}
```

### Firebase 프로젝트 분리 (필수)

**dev와 prod는 반드시 별도의 Firebase 프로젝트를 사용해야 한다.**

하나의 Firebase 프로젝트를 공유하면 FCM 토큰 풀이 합쳐지므로,  
dev에서 알림을 테스트할 때 prod 실사용자에게 전달될 수 있다.

- [ ] Firebase Console에서 프로젝트 2개 생성: `your-app-dev`, `your-app`
- [ ] 각 프로젝트에서 Android 앱 + iOS 앱 등록 (Bundle ID / Package Name 동일해도 무방)
- [ ] `google-services.json` (Android) / `GoogleService-Info.plist` (iOS) 각각 발급
- [ ] dev용 config → `.env.dev`, prod용 config → `.env.prod`에 분리 저장

> Firebase 프로젝트가 분리되면 FCM 토큰, Analytics, Crashlytics 모두 자동으로 격리된다.

### 환경변수 파일

```
.env.example    ← 커밋 O (CI 복사 기준, 실제 값 없음)
.env.dev        ← 커밋 X (gitignore) — dev Firebase 프로젝트 config
.env.prod       ← 커밋 X (gitignore) — prod Firebase 프로젝트 config
```

`.env.example` 내용 (프로젝트에 필요한 모든 키를 빈 값으로):

```
APP_NAME=
ENVIRONMENT=
SUPABASE_URL=
SUPABASE_ANON_KEY=
```

> `ci.yml`의 Analyze & Test 잡은 `cp .env.example .env.dev`로 플레이스홀더를 만든 뒤  
> `build_runner`를 실행한다. `.env.example`이 없으면 CI analyze/test 실패.

---

## 3단계 — iOS fastlane 파일

### `ios/Gemfile`

`cocoapods`를 반드시 포함해야 한다.  
`ruby/setup-ruby`가 `GEM_HOME`을 bundle 경로로 제한하므로, Gemfile에 없으면  
`bundle exec fastlane` 내부에서 pod를 찾지 못해 "CocoaPods is installed but broken" 오류.

```ruby
source "https://rubygems.org"

gem "fastlane"
gem "cocoapods"

plugins_path = File.join(File.dirname(__FILE__), 'fastlane', 'Pluginfile')
eval_gemfile(plugins_path) if File.exist?(plugins_path)
```

> `ios/Gemfile.lock`은 커밋하지 않는다. `BUNDLED WITH` 버전이 Ruby 3.2+ 환경에서  
> 충돌을 일으켜 CI 실패. `.gitignore`에 추가하거나 파일 자체를 삭제.

### `ios/fastlane/Pluginfile`

```ruby
gem 'fastlane-plugin-sanches37_deploy',
    git: 'https://github.com/sanches37/fastlane-plugin-sanches37_deploy',
    tag: 'v0.2.1'   # 최신 태그로 교체
```

### `ios/fastlane/Appfile`

```ruby
app_identifier "com.yourapp"
apple_id "your@email.com"
```

### `ios/fastlane/Matchfile`

```ruby
git_url("https://github.com/sanches37/match-certificates")
storage_mode("git")
type("appstore")
app_identifier(["com.yourapp"])

# 로컬 실행 시에만 .api_key.json 사용. CI는 환경변수로 처리.
_api_key_path = File.expand_path("fastlane/.api_key.json", __dir__)
api_key_path(_api_key_path) if File.exist?(_api_key_path)

# match 인증서 암호화 호환성 — 생략 시 CI에서 복호화 실패 가능
force_legacy_encryption(true)
```

### `ios/fastlane/Fastfile`

**패턴 A** — `lib/main.dart`에 `main()`이 있는 표준 구조:

```ruby
default_platform(:ios)

BUNDLE_ID = "com.yourapp"
IPA_PATH = File.expand_path("../../build/ios/ipa/your_app.ipa", __dir__)

SIGN_BUILD_COMMON = {
  bundle_id:            BUNDLE_ID,
  apple_team_id:        ENV['APPLE_TEAM_ID'],
  match_type:           'appstore',
  project_file:         'ios/Runner.xcodeproj/project.pbxproj',
  flutter_project_root: File.expand_path('../..', __dir__),
}.freeze

def app_version
  line = File.read(File.expand_path("../../pubspec.yaml", __dir__)).match(/^version:\s*(.+)/)
  line[1].strip.split("+").first
end

platform :ios do
  desc "Dev: 빌드 + TestFlight Internal Testing 업로드 (CI용)"
  lane :dev_build_and_release do
    ipa = ios_sign_and_build(**SIGN_BUILD_COMMON)
    ios_dev_release(bundle_id: BUNDLE_ID, ipa_path: ipa)
  end

  desc "Prod: 빌드 + App Store 메타데이터 + 심사 제출 (CI용)"
  lane :prod_build_and_submit do
    ipa = ios_sign_and_build(**SIGN_BUILD_COMMON)
    ios_prod_release_and_submit(bundle_id: BUNDLE_ID, ipa_path: ipa, app_version: app_version)
  end

  desc "TestFlight Internal Testing 업로드 (심사 제출 없음)"
  lane :dev_release do
    ios_dev_release(bundle_id: BUNDLE_ID, ipa_path: IPA_PATH)
  end

  desc "App Store 메타데이터 + TestFlight + 심사 제출"
  lane :prod_release_and_submit do
    ios_prod_release_and_submit(bundle_id: BUNDLE_ID, ipa_path: IPA_PATH, app_version: app_version)
  end

  desc "심사 거절 후 빌드 재업로드 + 재제출"
  lane :resubmit do
    ios_resubmit(bundle_id: BUNDLE_ID, ipa_path: IPA_PATH, app_version: app_version)
  end
end
```

**패턴 B** — 분리 진입점 구조 (`lib/main_dev.dart` / `lib/main_prod.dart`):  
`SIGN_BUILD_COMMON`은 동일, 레인에서 `build_flutter_args`로 `--target`을 전달한다.

```ruby
# build_flutter_args 메서드 추가 (SIGN_BUILD_COMMON 아래)
def build_flutter_args(target)
  base = ENV.fetch('FLUTTER_ARGS', '')
  [base, "--target lib/#{target}"].reject(&:empty?).join(' ')
end

# 레인만 아래처럼 변경
lane :dev_build_and_release do
  ipa = ios_sign_and_build(**SIGN_BUILD_COMMON, flutter_args: build_flutter_args('main_dev.dart'))
  ios_dev_release(bundle_id: BUNDLE_ID, ipa_path: ipa)
end

lane :prod_build_and_submit do
  ipa = ios_sign_and_build(**SIGN_BUILD_COMMON, flutter_args: build_flutter_args('main_prod.dart'))
  ios_prod_release_and_submit(bundle_id: BUNDLE_ID, ipa_path: ipa, app_version: app_version)
end
```

### `ios/fastlane/.api_key.json` (로컬 전용, gitignore)

```json
{
  "key_id": "YOUR_KEY_ID",
  "issuer_id": "YOUR_ISSUER_ID",
  "key": "-----BEGIN PRIVATE KEY-----\nMIGH...\n-----END PRIVATE KEY-----",
  "in_house": false
}
```

> CI는 이 파일 없이 `ASC_KEY_ID` / `ASC_ISSUER_ID` / `ASC_PRIVATE_KEY` 환경변수를 직접 읽는다.

### 최초 match 실행 (인증서·프로파일 생성)

fastlane 파일 설정이 완료된 후 로컬에서 1회 실행한다.  
인증서와 프로비저닝 프로파일을 생성해 `match-certificates` 레포에 push한다.

```bash
cd ios
bundle exec fastlane match appstore --readonly false
```

> 이후 CI는 항상 `--readonly true`로만 읽는다. 인증서 갱신 필요 시에만 로컬에서 재실행.

---

## 3-B. Android fastlane 파일

iOS와 동일하게 `android/` 디렉토리에 fastlane 설정을 둔다. Android는 Firebase App Distribution 플러그인 하나만 사용하므로 iOS보다 단순하다.

### `android/Gemfile`

```ruby
source "https://rubygems.org"

gem "fastlane"

plugins_path = File.join(File.dirname(__FILE__), "fastlane", "Pluginfile")
eval_gemfile(plugins_path) if File.exist?(plugins_path)
```

> Android는 iOS와 달리 `cocoapods` 의존성 없음. 기본 `fastlane`만 명시.

### `android/fastlane/Pluginfile`

```ruby
gem "fastlane-plugin-firebase_app_distribution"
```

> 이 플러그인은 Firebase 인증을 service account JSON으로 처리하므로, iOS의 `fastlane-plugin-sanches37_deploy`와 달리 별도 PAT가 필요 없다.

### `android/fastlane/Fastfile`

```ruby
default_platform(:android)

platform :android do
  desc "Firebase App Distribution dev 배포"
  lane :dev_release do
    firebase_app_distribution(
      app: ENV.fetch("FIREBASE_APP_ID"),
      android_artifact_type: "APK",
      android_artifact_path: ENV.fetch("APK_PATH"),
      service_credentials_file: ENV.fetch("FIREBASE_SERVICE_ACCOUNT_PATH"),
      release_notes: ENV.fetch("RELEASE_NOTES", "dev build")
    )
  end
end
```

> `FIREBASE_APP_ID`, `APK_PATH`, `FIREBASE_SERVICE_ACCOUNT_PATH`, `RELEASE_NOTES` 환경변수는 `android-dev.yml`이 자동으로 주입한다. Fastfile에 패키지명 / 앱 ID를 하드코딩하지 않아도 됨.

### Firebase App ID 확인

Firebase Console → 프로젝트 설정 → 일반 → Android 앱 → "앱 ID"  
형식: `1:000000000000:android:abcd...` — 비민감 식별자이므로 GitHub Actions caller에 평문으로 박는다 (secret 아님).

### Firebase service account JSON 발급

Firebase Console → 프로젝트 설정 → 서비스 계정 → "새 비공개 키 생성" → 다운로드한 JSON 파일 전체를 `FIREBASE_SERVICE_ACCOUNT_JSON_DEV` secret에 그대로 붙여넣는다 (개행 포함, base64 인코딩 X).

---

## 4단계 — GitHub Actions 워크플로우

### `.github/workflows/deploy-ios-dev.yml`

```yaml
name: iOS Dev Deploy

on:
  push:
    tags:
      - 'v*-dev-ios'

jobs:
  deploy:
    uses: sanches37/flutter-deploy-workflows/.github/workflows/ios-dev.yml@v1.7.0
    with:
      bundle_id_dev: com.yourapp
      apple_team_id: ${{ vars.APPLE_TEAM_ID }}
      flutter_version: '3.41.7'
    secrets: inherit
```

### `.github/workflows/deploy-ios-prod.yml`

```yaml
name: iOS Prod Deploy

on:
  push:
    tags:
      - 'v*-prod-ios'

jobs:
  deploy:
    uses: sanches37/flutter-deploy-workflows/.github/workflows/ios-prod.yml@v1.7.0
    with:
      bundle_id: com.yourapp
      apple_team_id: ${{ vars.APPLE_TEAM_ID }}
      flutter_version: '3.41.7'
    secrets: inherit
```

### `.github/workflows/deploy-android-dev.yml`

```yaml
name: Android Dev Deploy (Firebase)

on:
  push:
    tags:
      - 'v*-dev-android'

jobs:
  deploy:
    uses: sanches37/flutter-deploy-workflows/.github/workflows/android-dev.yml@v1.8.0
    with:
      package_name_dev: com.yourapp.dev
      firebase_app_id: '1:000000000000:android:placeholder'  # Firebase Console에서 확인
      flutter_version: '3.41.7'
      target: lib/main_dev.dart   # 패턴 A(표준 main.dart)면 생략
    secrets:
      FIREBASE_SERVICE_ACCOUNT_JSON: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_JSON_DEV }}
      ENV_DEV: ${{ secrets.ENV_DEV }}
```

> Android는 iOS와 달리 `secrets: inherit`을 쓰지 않고 명시적으로 매핑한다. dev/prod Firebase 프로젝트 secret 이름이 분리되기 때문(`*_DEV` / `*_PROD`).

> v1.8.0 미만 버전은 `.gha` checkout 누락 + 키스토어 단계 `if:` 시크릿 평가 버그로 Android 배포 실패. v1.8.0 이상 필수.

---

## 5단계 — GitHub Secrets / Variables 등록

GitHub 레포 → Settings → Secrets and variables → Actions

### Secrets (필수)

| Secret | 설명 |
|--------|------|
| `ASC_KEY_ID` | App Store Connect API Key ID |
| `ASC_ISSUER_ID` | App Store Connect Issuer ID |
| `ASC_PRIVATE_KEY` | `.p8` 파일 내용 전체 (개행 포함) |
| `MATCH_PASSWORD` | match 인증서 암호화 비밀번호 (아무 문자열) |
| `MATCH_GIT_TOKEN` | match-certificates 레포 접근용 PAT (`repo` 스코프) |
| `ENV_DEV` | base64 인코딩된 `.env.dev` 파일 내용 |

### Secrets (선택)

| Secret | 설명 |
|--------|------|
| `PLUGIN_GIT_TOKEN` | 플러그인 레포가 별도 계정이라면 별도 PAT. 미설정 시 `MATCH_GIT_TOKEN` 재사용 |
| `ENV_PROD` | base64 인코딩된 `.env.prod` 파일 내용 (prod 배포 시 필요) |

### Secrets (Android, 필수)

| Secret | 설명 |
|--------|------|
| `FIREBASE_SERVICE_ACCOUNT_JSON_DEV` | Firebase 서비스 계정 JSON 파일 전체 (dev 프로젝트). caller에서 `FIREBASE_SERVICE_ACCOUNT_JSON` 시크릿에 매핑 |

### Secrets (Android, 선택 — 릴리즈 키스토어 사용 시)

| Secret | 설명 |
|--------|------|
| `RELEASE_KEYSTORE_BASE64` | `release.jks`를 base64 인코딩한 문자열. 미설정 시 디버그 서명으로 빌드 |
| `RELEASE_KEYSTORE_PASSWORD` | 키스토어 비밀번호 |
| `RELEASE_KEY_ALIAS` | 키 alias |
| `RELEASE_KEY_PASSWORD` | 키 비밀번호 |

> Android 4개 키스토어 secret은 모두 같이 설정하거나 모두 미설정. 일부만 설정하면 빌드는 되지만 서명이 의도와 다를 수 있음.

### Variables (필수)

| Variable | 예시 |
|----------|------|
| `APPLE_TEAM_ID` | `626VJ36K6Q` |

> Variables는 Settings → Secrets and variables → Actions → **Variables** 탭.  
> Secret이 아니므로 `${{ vars.APPLE_TEAM_ID }}`로 참조.

### ENV_DEV / ENV_PROD 인코딩 방법

```bash
# macOS
base64 -i .env.dev | tr -d '\n'
# 출력된 문자열을 GitHub Secret에 붙여넣기
```

---

## 6단계 — .gitignore 확인

아래 항목이 `.gitignore`에 포함돼 있는지 확인:

```gitignore
# 환경변수 파일 (실제 값)
.env.dev
.env.prod

# fastlane 로컬 파일
ios/fastlane/.api_key.json

# Bundler local install (CI에서 ruby/setup-ruby가 여기에 설치)
ios/vendor/
android/vendor/

# envied 생성 파일 (실제 키 포함)
lib/core/config/env_*.g.dart

# Gemfile.lock — BUNDLED WITH 버전이 Ruby 3.2+ 환경에서 충돌 유발
ios/Gemfile.lock
android/Gemfile.lock

# Android 릴리즈 키스토어 (CI에서 RELEASE_KEYSTORE_BASE64로부터 decode)
android/app/keystore/
android/key.properties
```

---

## 7단계 — prod 첫 배포 전 메타데이터 초기화

`ios_prod_release_and_submit`은 내부적으로 `fastlane deliver`를 호출한다.  
`ios/fastlane/metadata/` 폴더가 없으면 실패하므로, prod 배포 전 1회 실행:

```bash
cd ios
bundle exec fastlane deliver init
```

→ `ios/fastlane/metadata/ko/` 폴더가 생성된다. 앱 설명·키워드 등은 이 폴더에서 편집.

---

## 8단계 — 배포 트리거

### 태그 형식

| 태그 | 워크플로우 | 대상 |
|------|-----------|------|
| `v{semver}-dev-ios` | `ios-dev.yml` | TestFlight Internal Testing |
| `v{semver}-prod-ios` | `ios-prod.yml` | App Store 심사 제출 |
| `v{semver}-dev-android` | `android-dev.yml` | Firebase App Distribution |
| `v{semver}-prod-android` | `android-prod.yml` | Play Store production track |

- `pubspec.yaml`의 `version: 1.0.0+3`에서 semver 부분(`1.0.0`)만 사용한다.  
  빌드번호(`+3`)는 태그에 포함하지 않는다.
- **prod 태그는 반드시 `main` 브랜치에서 생성해야 한다.**  
  다른 브랜치에서 prod 태그를 push하면 `verify-version` 단계에서 차단.

### 배포 순서 예시 — iOS (v1.0.0 첫 배포)

```bash
# 1. pubspec.yaml version 확인: 1.0.0+1
# 2. main 브랜치에 최신 코드 push 완료 확인
git tag v1.0.0-dev-ios
git push origin v1.0.0-dev-ios
# → TestFlight Internal Testing 업로드 → 내부 테스트 후

# 3. prod 태그 전 빌드번호만 증가 (버전은 동일하게 유지)
#    pubspec.yaml: version: 1.0.0+1 → 1.0.0+2
#    이유: Apple은 같은 version+build 조합 중복 허용 안 함.
#    dev(+1)가 이미 업로드된 상태에서 prod(+1)를 올리면 거부됨.
git add pubspec.yaml
git commit -m "chore: 빌드번호 증가 (1.0.0+1 → 1.0.0+2)"
git push origin main
git tag v1.0.0-prod-ios
git push origin v1.0.0-prod-ios
# → App Store 심사 제출
```

### 배포 순서 예시 — Android (v1.0.0 dev 배포)

```bash
# 1. pubspec.yaml version 확인: 1.0.0+1
# 2. main 브랜치 또는 feature 브랜치에서 태그 생성 (dev는 main 강제 X)
git tag v1.0.0-dev-android
git push origin v1.0.0-dev-android
# → Firebase App Distribution 테스터 그룹에 APK 배포
```

> Android dev는 Firebase App Distribution이라 Apple과 달리 빌드번호 중복 제약이 없다. dev → prod 사이 빌드번호 증가는 불필요.

---

## 9단계 — CI (선택)

PR / `main` push 시 `flutter analyze` + `very_good test`를 돌려 회귀를 막는다. 자동배포 워크플로우(1~8단계)는 태그 push 시점에만 도므로, CI 없이는 분석·테스트 회귀를 PR 단계에서 잡을 수 없다.

### `.github/workflows/ci.yml`

```yaml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  analyze-and-test:
    name: Analyze & Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: 3.41.7
          channel: stable
          cache: true

      - name: Cache pub packages
        uses: actions/cache@v4
        with:
          path: |
            ~/.pub-cache
            .dart_tool
          key: ${{ runner.os }}-pub-${{ hashFiles('**/pubspec.lock') }}
          restore-keys: |
            ${{ runner.os }}-pub-

      # ↓ envied 사용 시에만 필요. 미사용 프로젝트는 이 step 통째로 삭제.
      - name: Prepare env files (envied placeholder)
        run: |
          cat > .env.dev <<EOF
          APP_NAME=your_app DEV
          ENVIRONMENT=dev
          SUPABASE_URL=https://placeholder.supabase.co
          SUPABASE_ANON_KEY=placeholder_anon_key
          EOF
          cat > .env.prod <<EOF
          APP_NAME=your_app
          ENVIRONMENT=prod
          SUPABASE_URL=https://placeholder.supabase.co
          SUPABASE_ANON_KEY=placeholder_anon_key
          EOF

      - name: Install dependencies
        run: flutter pub get

      # ↓ envied 사용 시에만 필요. 미사용 프로젝트는 이 step 통째로 삭제.
      - name: Run build_runner
        run: dart run build_runner build --delete-conflicting-outputs

      - name: Analyze
        run: flutter analyze

      - name: Install very_good_cli
        run: dart pub global activate very_good_cli

      - name: Test with coverage (min 30%)
        run: very_good test --min-coverage 30

      - name: Upload coverage artifact
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/lcov.info
          retention-days: 7
```

### envied 사용 여부에 따른 분기

- **envied 사용**: 위 yml 그대로 사용. `.env.dev` / `.env.prod` placeholder 작성 + `build_runner` step 필수.
- **envied 미사용**: `Prepare env files (envied placeholder)` + `Run build_runner` 두 step을 통째로 삭제. `flutter pub get` 직후 바로 `Analyze`로 진행.

> placeholder 값은 envied가 `@EnviedField`의 타입 제약(URL 형식 등)을 컴파일 타임에 검사하므로, 위처럼 형식만 맞춰주면 충분. 실제 값을 채울 필요는 없다.

---

## Known Issues & 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| `CocoaPods is installed but broken` | `ios/Gemfile`에 `gem "cocoapods"` 없음 | Gemfile에 추가 |
| `No 'main' method found` | `flutter build ipa` 타겟이 `lib/main.dart`인데 `main()` 없음 | Fastfile `build_flutter_args`로 `--target lib/main_dev.dart` 명시 |
| `No profile for team matching 'match AppStore com.x' found` | match가 timestamp suffix 붙인 프로파일 이름과 ExportOptions.plist 불일치 | plugin v0.2.1+ 사용 (`MATCH_PROVISIONING_PROFILE_MAPPING` 우선 읽기) |
| `flutterfire: command not found` → `ARCHIVE FAILED` | `firebase_crashlytics` 사용 시 Xcode 빌드 단계가 FlutterFire CLI 호출. CI에 미설치 | `flutter-deploy-workflows v1.6.3+` 사용 시 자동 처리 |
| `SDK version issue. built with iOS 18.5 SDK` → 업로드 거부 | Apple App Store Connect가 iOS 26 SDK(Xcode 26+) 빌드만 허용. `macos-latest` 러너 기본 Xcode가 16.4로 고정됨 | `flutter-deploy-workflows v1.7.0+` 사용 시 자동 처리 (`latest-stable` Xcode 선택) |
| `BUNDLED WITH` 버전 오류 | `ios/Gemfile.lock` 커밋 시 Ruby 3.2+ 충돌 | `Gemfile.lock` 삭제 + `.gitignore` 추가 |
| `api_key_path` 파일 없음 오류 | CI에 `.api_key.json` 없는데 Matchfile에 조건 없음 | `api_key_path(_api_key_path) if File.exist?(_api_key_path)` 로 조건부 처리 |
| match 복호화 실패 | `force_legacy_encryption` 설정 불일치 | Matchfile에 `force_legacy_encryption(true)` 추가 |
| prod 태그 push 후 `main 브랜치에서만` 오류 | feature 브랜치에서 prod 태그 push | main 병합 후 태그 |
| `Environment variable not found for field` (build_runner) | `.env.example`이 없거나 필드 누락 | `.env.example`에 필요한 모든 키 추가 |
| `Can't find action './.github/actions/...'` (Android dev 배포) | reusable workflow의 composite action 상대 경로가 caller 레포 워크스페이스로 해석. `.gha` checkout 누락 + 경로 미수정 | `flutter-deploy-workflows v1.8.0+` 사용 시 자동 처리 |
| Android dev 배포 시 릴리즈 키스토어 단계가 항상 스킵됨 (`RELEASE_KEYSTORE_BASE64` 설정해도) | `if:` 조건에서 secret 값이 빈 문자열로 평가됨 (GitHub Actions가 `if:` 평가 시점에 secret 미접근) | `flutter-deploy-workflows v1.8.0+` 사용 시 자동 처리 (run 내부에서 조건 분기) |

---

## 버전 호환성

| flutter-deploy-workflows | fastlane-plugin-sanches37_deploy | Flutter |
|---|---|---|
| v1.8.0 | v0.2.1 | >= 3.32.0 |
| v1.7.0 | v0.2.1 | >= 3.32.0 |
| v1.6.2 ~ v1.6.3 | v0.2.1 | >= 3.32.0 |
| v1.6.0 ~ v1.6.1 | v0.2.0 | >= 3.32.0 |
| v1.0.0 ~ v1.1.0 | v0.1.x | >= 3.32.0 |
