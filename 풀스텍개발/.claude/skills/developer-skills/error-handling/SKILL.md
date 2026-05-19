# error-handling

## 목적
일관된 에러 처리와 로깅 패턴을 적용하여 디버깅 용이성과 사용자 경험을 동시에 보장한다.

## 에러 응답 표준 형식
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "이메일 형식이 올바르지 않습니다.",
    "details": [{ "field": "email", "issue": "invalid format" }]
  }
}
```

## 에러 코드 체계
| 코드 | HTTP | 설명 |
|------|------|------|
| `VALIDATION_ERROR` | 400 | 입력 유효성 실패 |
| `UNAUTHORIZED` | 401 | 인증 필요 |
| `FORBIDDEN` | 403 | 권한 없음 |
| `NOT_FOUND` | 404 | 리소스 없음 |
| `CONFLICT` | 409 | 중복 데이터 |
| `INTERNAL_ERROR` | 500 | 서버 내부 오류 |

## 백엔드 에러 처리 (Express)
```javascript
// middleware/errorHandler.js
class AppError extends Error {
  constructor(message, status, code) {
    super(message);
    this.status = status;
    this.code = code;
  }
}

const errorHandler = (err, req, res, next) => {
  const status = err.status || 500;
  const code = err.code || 'INTERNAL_ERROR';

  // 운영 환경에서 내부 스택 숨김
  const isDev = process.env.NODE_ENV === 'development';
  console.error(`[ERROR] ${req.method} ${req.path}:`, err.message);
  if (isDev) console.error(err.stack);

  res.status(status).json({
    error: { code, message: err.message }
  });
};
```

## 로깅 패턴
```javascript
// 요청/응답 로그 미들웨어
const requestLogger = (req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(`${req.method} ${req.path} ${res.statusCode} - ${duration}ms`);
  });
  next();
};
```

## 프론트엔드 에러 처리
```typescript
// API 에러 처리 훅
const useApiError = () => {
  const handleError = (err: unknown) => {
    if (axios.isAxiosError(err)) {
      const message = err.response?.data?.error?.message || '요청 처리 중 오류가 발생했습니다.';
      toast.error(message);  // 사용자 안내
    }
  };
  return { handleError };
};
```

## DB 에러 처리
```javascript
// DB 작업 래퍼
const safeQuery = async (queryFn) => {
  try {
    return await queryFn();
  } catch (err) {
    if (err.code === '23505') {  // PostgreSQL unique violation
      throw new AppError('이미 존재하는 데이터입니다.', 409, 'CONFLICT');
    }
    throw new AppError('데이터베이스 오류', 500, 'INTERNAL_ERROR');
  }
};
```
