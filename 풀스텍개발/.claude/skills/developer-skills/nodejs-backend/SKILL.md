# nodejs-backend

## 목적
Node.js 기반 백엔드 구현 패턴과 베스트 프랙티스를 제공한다.

## 프레임워크 선택
| 조건 | 추천 |
|------|------|
| REST API, 빠른 구현 | Express.js |
| TypeScript 강타입 | NestJS |
| 경량 서버 | Fastify |

## 프로젝트 구조 (Express 기준)
```
src/
├── app.js              # Express 앱 설정
├── server.js           # 서버 시작점
├── routes/             # 라우터 (api-spec.yaml 기준)
├── controllers/        # 요청 처리 로직
├── services/           # 비즈니스 로직
├── models/             # DB 모델 (Sequelize/Prisma)
├── middleware/         # 인증, 에러 핸들러
└── config/             # 환경변수, DB 설정
```

## 필수 패키지
```json
{
  "express": "^4.x",
  "jsonwebtoken": "^9.x",
  "bcryptjs": "^2.x",
  "sequelize": "^6.x",    // ORM (PostgreSQL)
  "pg": "^8.x",
  "dotenv": "^16.x",
  "express-validator": "^7.x"
}
```

## 에러 핸들링 표준
```javascript
// middleware/errorHandler.js
const errorHandler = (err, req, res, next) => {
  const status = err.status || 500;
  res.status(status).json({
    error: {
      message: err.message,
      code: err.code || 'INTERNAL_ERROR'
    }
  });
};
```

## 헬스체크 엔드포인트 (필수)
```javascript
app.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});
```

## 환경변수 (.env.example)
```env
NODE_ENV=production
PORT=8000
DATABASE_URL=postgresql://user:pass@db:5432/appdb
JWT_SECRET=change-this-secret
JWT_EXPIRES_IN=24h
```
