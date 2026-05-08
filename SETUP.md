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

### 환경변수 파일

```
.env.example    ← 커밋 O (CI 복사 기준, 실제 값 없음)
.env.dev        ← 커밋 X (gitignore)
.env.prod       ← 커밋 X (gitignore)
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

# envied 생성 파일 (실제 키 포함)
lib/core/config/env_*.g.dart

# Gemfile.lock — BUNDLED WITH 버전이 Ruby 3.2+ 환경에서 충돌 유발
ios/Gemfile.lock
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

```
v{semver}-dev-ios    # TestFlight Internal Testing
v{semver}-prod-ios   # App Store 심사 제출
```

- `pubspec.yaml`의 `version: 1.0.0+3`에서 semver 부분(`1.0.0`)만 사용한다.  
  빌드번호(`+3`)는 태그에 포함하지 않는다.
- **prod 태그는 반드시 `main` 브랜치에서 생성해야 한다.**  
  다른 브랜치에서 prod 태그를 push하면 `verify-version` 단계에서 차단.

### 배포 순서 예시 (v1.0.0 첫 배포)

```bash
# 1. pubspec.yaml version 확인: 1.0.0+1
# 2. main 브랜치에 최신 코드 push 완료 확인
git tag v1.0.0-dev-ios
git push origin v1.0.0-dev-ios
# → TestFlight Internal Testing 업로드 → 내부 테스트 후

git tag v1.0.0-prod-ios
git push origin v1.0.0-prod-ios
# → App Store 심사 제출
```

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

---

## 버전 호환성

| flutter-deploy-workflows | fastlane-plugin-sanches37_deploy | Flutter |
|---|---|---|
| v1.7.0 | v0.2.1 | >= 3.32.0 |
| v1.6.2 ~ v1.6.3 | v0.2.1 | >= 3.32.0 |
| v1.6.0 ~ v1.6.1 | v0.2.0 | >= 3.32.0 |
| v1.0.0 ~ v1.1.0 | v0.1.x | >= 3.32.0 |
