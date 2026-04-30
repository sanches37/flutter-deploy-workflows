# flutter-deploy-workflows

Flutter 자동배포용 GitHub Actions reusable workflows. iOS/Android × dev/prod 4개 워크플로우 + composite actions.

## 상태

🚧 **빈 repo. 다음 세션에서 구현 예정.**

## 다음 작업

전체 계획서 참조:
`/Users/taehoonpark/Desktop/Project/flutter_deploy_setup/PLAN.md`

특히:
- **Section 7**: 본 repo의 폴더 구조 + 4개 reusable workflow inputs/secrets 명세
- **Section 9 - Day 2**: 구현 작업 순서

## 제공 예정 워크플로우

| 워크플로우 | 용도 | 트리거 (호출 측) |
|---|---|---|
| `ios-dev.yml` | TestFlight Internal Testing | `v*-dev-ios` |
| `ios-prod.yml` | TestFlight + App Store 심사 제출 | `v*-prod-ios` |
| `android-dev.yml` | Firebase App Distribution | `v*-dev-android` |
| `android-prod.yml` | Play Store production track | `v*-prod-android` |

## 사용 예시 (구현 후 모습)

```yaml
# 프로젝트의 .github/workflows/deploy-ios-prod.yml
name: Deploy iOS Prod
on:
  push:
    tags: ['v*-prod-ios']

jobs:
  deploy:
    uses: sanches37/flutter-deploy-workflows/.github/workflows/ios-prod.yml@v1
    with:
      bundle_id: com.lifeadmin.app
      apple_team_id: <Apple Team ID>
      flutter_version: '3.32.2'
    secrets: inherit
```

## 디자인 원칙

- 베달판(`/Users/taehoonpark/Desktop/Project/baedalpan/.github/workflows/`)의 검증된 workflow를 source of truth로 사용
- 솔로 Flutter 개발자 시나리오에 맞춰 단순화 (runner 라우팅, Slack 통합, 다국어 등 제거)
- fastlane match 도입으로 인증서 step ~150줄 → ~5줄 축소
- 페어 plugin (`fastlane-plugin-sanches37_deploy`)을 통해 lane 로직 호출

## 호환성

- 페어 plugin: `fastlane-plugin-sanches37_deploy@~> 0.1`
- Flutter: `>= 3.32.0`
- Java (Android): `17`
