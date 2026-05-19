# integration-test

## 목적
Docker Compose 환경에서 실제 DB·서비스 간 통합 테스트를 실행하고 API 계약 준수를 검증한다.

## 실행 순서
```bash
# 1. Docker Compose 기동
docker compose up -d

# 2. 서비스 healthy 대기
docker compose ps  # 모든 서비스 healthy 상태 확인

# 3. 통합 테스트 실행
npm test -- --testPathPattern=integration

# 4. Smoke 테스트 실행
bash tests/smoke/run_smoke.sh

# 5. 정리
docker compose down -v
```

## API 계약 검증 (api-spec.yaml 기준)
실제 응답이 spec의 스키마와 일치하는지 자동 검증:
```javascript
// tests/integration/contract.test.js
const SwaggerParser = require('@apidevtools/swagger-parser');
const request = require('supertest');

describe('API 계약 검증', () => {
  let spec;
  beforeAll(async () => {
    spec = await SwaggerParser.dereference('api-spec.yaml');
  });

  test('GET /users 응답 스키마 일치', async () => {
    const res = await request(app).get('/users').set('Authorization', `Bearer ${token}`);
    const schema = spec.paths['/users'].get.responses['200'].content['application/json'].schema;
    // 응답 구조 검증
    expect(res.body).toMatchSchema(schema);
  });
});
```

## 테스트 DB 격리
```javascript
// 테스트 전 DB 초기화
beforeEach(async () => {
  await db.query('TRUNCATE users, posts CASCADE');
  await db.query(`INSERT INTO users VALUES ('test-id', 'test@example.com', ...)`);
});
```

## 커버리지 측정
```bash
# Jest 커버리지 리포트 생성
npm test -- --coverage --coverageReporters=text-summary

# 핵심 모듈 80% 미달 시 실패 처리
jest --coverageThreshold='{"global":{"lines":80}}'
```

## 회귀 테스트 (enhance 모드)
기존 기능 보호를 위한 회귀 테스트 먼저 실행:
```bash
# 1. 회귀 테스트 먼저
npm test -- --testPathPattern=regression
echo "회귀 테스트 결과: $?"

# 2. 신규 기능 테스트
npm test -- --testPathPattern=integration
```
