# code-style-lint

## 목적
코드 컨벤션 일관성을 검사하고 가독성·유지보수성을 저해하는 패턴을 식별한다.

## 자동 검사 명령어

### JavaScript/TypeScript
```bash
# ESLint 실행
npx eslint src/ --ext .js,.ts,.tsx

# Prettier 포맷 확인
npx prettier --check src/

# TypeScript 타입 검사
npx tsc --noEmit
```

### Python
```bash
# Flake8 린트
flake8 src/ --max-line-length=100

# mypy 타입 검사
mypy src/

# Black 포맷 확인
black --check src/
```

## 주요 검사 항목

### Critical 제외 (Minor 이하)
| 항목 | 설명 |
|------|------|
| 미사용 변수·임포트 | `import X from 'y'` 사용 안 함 |
| 과도한 함수 길이 | 함수 50줄 초과 |
| 중첩 depth 과다 | if/for 3단계 이상 중첩 |
| console.log 잔존 | 운영 코드에 디버그 출력 |
| TODO 주석 | 해결되지 않은 TODO |

### Conventional Commits 검증
developer가 작성한 커밋 메시지가 형식을 따르는지 확인:
```
✅ feat: 사용자 인증 API 추가
✅ fix: JWT 만료 처리 버그 수정
✅ test: 로그인 통합 테스트 추가
❌ 인증 추가함 (타입 없음)
❌ fixed bug (영문 단수형 아님)
```

## ESLint 권장 설정 (.eslintrc.js)
```javascript
module.exports = {
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
  ],
  rules: {
    'no-console': 'warn',
    'no-unused-vars': 'error',
    'prefer-const': 'error',
    'no-var': 'error',
  }
};
```
