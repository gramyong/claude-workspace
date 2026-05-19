# unit-testing

## 목적
테스트 커버리지 80% 달성을 위한 단위·통합 테스트 작성 가이드를 제공한다.

## 백엔드 테스트 (Node.js/Jest)

### 설정
```json
// package.json
{
  "scripts": {
    "test": "jest --coverage",
    "test:watch": "jest --watch"
  },
  "jest": {
    "coverageThreshold": {
      "global": { "lines": 80, "functions": 80 }
    }
  }
}
```

### 단위 테스트 패턴
```javascript
// tests/unit/authService.test.js
const { hashPassword, verifyPassword } = require('../../services/authService');

describe('authService', () => {
  test('비밀번호 해싱 후 검증 성공', async () => {
    const hash = await hashPassword('mypassword');
    expect(await verifyPassword('mypassword', hash)).toBe(true);
  });

  test('잘못된 비밀번호 검증 실패', async () => {
    const hash = await hashPassword('mypassword');
    expect(await verifyPassword('wrongpassword', hash)).toBe(false);
  });
});
```

### 통합 테스트 패턴 (supertest)
```javascript
// tests/integration/auth.test.js
const request = require('supertest');
const app = require('../../app');

describe('POST /auth/login', () => {
  test('올바른 자격증명으로 JWT 반환', async () => {
    const res = await request(app)
      .post('/auth/login')
      .send({ email: 'test@example.com', password: 'password123' });
    expect(res.status).toBe(200);
    expect(res.body).toHaveProperty('token');
  });

  test('잘못된 비밀번호 401 반환', async () => {
    const res = await request(app)
      .post('/auth/login')
      .send({ email: 'test@example.com', password: 'wrong' });
    expect(res.status).toBe(401);
  });
});
```

## 프론트엔드 테스트 (Vitest + Testing Library)
```typescript
// components/__tests__/LoginForm.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import LoginForm from '../LoginForm';

test('로그인 폼 렌더링', () => {
  render(<LoginForm onSubmit={jest.fn()} />);
  expect(screen.getByPlaceholderText('이메일')).toBeInTheDocument();
  expect(screen.getByText('로그인')).toBeInTheDocument();
});
```

## 커버리지 목표
- 비즈니스 로직 (services/): 90%+
- 라우터/컨트롤러: 80%+
- 유틸리티: 70%+
