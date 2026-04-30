# flutter-deploy-workflows

Flutter 자동배포용 GitHub Actions reusable workflows. iOS/Android × dev/prod 4개 워크플로우 + 4개 composite actions.

## 제공 워크플로우

| 워크플로우 | 용도 | 트리거 (호출 측) |
|---|---|---|
| `ios-dev.yml` | TestFlight Internal Testing | `v*-dev-ios` |
| `ios-prod.yml` | TestFlight + App Store 심사 제출 | `v*-prod-ios` |
| `android-dev.yml` | Firebase App Distribution | `v*-dev-android` |
| `android-prod.yml` | Play Store production track | `v*-prod-android` |

## Composite Actions

| Action | 역할 |
|---|---|
| `setup-flutter` | Flutter 설치 + pub get + .env 디코딩 + build_runner |
| `setup-ruby` | Ruby 셋업 + private gem git auth + bundler-cache |
| `setup-ios-signing` | fastlane match readonly + ExportOptions.plist 생성 |
| `verify-version` | tag ↔ pubspec.yaml 버전 일치 검증 + main 강제 옵션 |

## 사용 예시

```yaml
# .github/workflows/deploy-ios-prod.yml
name: Deploy iOS Prod
on:
  push:
    tags: ['v*-prod-ios']

jobs:
  deploy:
    uses: sanches37/flutter-deploy-workflows/.github/workflows/ios-prod.yml@v1.1.0
    with:
      bundle_id: com.yourapp
      apple_team_id: YOUR_TEAM_ID
      flutter_version: '3.32.2'
    secrets: inherit
```

```yaml
# .github/workflows/deploy-android-dev.yml
name: Deploy Android Dev
on:
  push:
    tags: ['v*-dev-android']

jobs:
  deploy:
    uses: sanches37/flutter-deploy-workflows/.github/workflows/android-dev.yml@v1.1.0
    with:
      package_name_dev: com.yourapp.dev
      firebase_app_id: "1:xxx:android:xxx"
      tester_groups: test
      flutter_version: '3.32.2'
    secrets: inherit
```

## 디자인 원칙

- 솔로 Flutter 개발자 시나리오에 맞춰 단순화 (runner 라우팅, Slack 통합, 다국어 등 제거)
- fastlane match 도입으로 인증서 step ~150줄 → ~5줄 축소
- 페어 plugin (`fastlane-plugin-sanches37_deploy`)을 통해 lane 로직 호출
- **버전 핀 디시플린**: `@main` 사용 금지, 항상 `@v1.x.x` 명시 핀

## 호환성

| workflow 버전 | plugin 버전 | Flutter |
|---|---|---|
| v1.0.0 ~ v1.1.0 | v0.1.x | >= 3.32.0 |
