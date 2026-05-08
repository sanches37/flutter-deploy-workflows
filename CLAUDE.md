# flutter-deploy-workflows

iOS/Android Flutter 자동배포용 reusable GitHub Actions workflows + composite actions.
페어 plugin: `sanches37/fastlane-plugin-sanches37_deploy`.

## 문서 동기화 규칙

**CI 버그픽스를 커밋할 때 반드시 같은 커밋에서 아래를 함께 처리한다:**

1. `CHANGELOG.md` — 버전 항목 추가 (Fixed 섹션)
2. `SETUP.md` Known Issues 테이블 — 해당 오류 증상·원인·해결 행 추가 또는 갱신

새 증상이 이미 테이블에 있는 행의 변형이면 기존 행을 수정한다. 완전히 새로운 유형이면 행을 추가한다.

커밋 메시지 예시:
```
fix(ios): CocoaPods broken 수정 + SETUP.md Known Issues 갱신
```

## 버전 규칙

- semver 준수. 버그픽스 → patch, 하위 호환 기능 → minor, 파괴적 변경 → major.
- `DEPLOY_WF_REF`를 하드코딩하는 구조상, 워크플로우 파일 변경 시 반드시 새 태그를 발행해야 호출 측에 반영된다.
- 태그 발행 전 `CHANGELOG.md`의 `[Unreleased]` 항목을 버전으로 확정한다.

## 호환성 유지

`README.md` 하단 호환성 테이블을 새 버전 발행 시 업데이트한다.
