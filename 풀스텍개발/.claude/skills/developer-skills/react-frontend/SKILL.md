# react-frontend

## 목적
React 기반 프론트엔드 구현 패턴과 베스트 프랙티스를 제공한다.

## 초기화 (Vite + TypeScript 기준)
```bash
npm create vite@latest frontend -- --template react-ts
cd frontend && npm install
```

## 프로젝트 구조
```
src/
├── components/         # 재사용 컴포넌트
├── pages/              # 페이지 컴포넌트
├── services/           # API 호출 모듈
│   └── api.ts          # axios 인스턴스 + 엔드포인트
├── hooks/              # 커스텀 훅
├── store/              # 상태 관리 (Context 또는 Zustand)
├── types/              # TypeScript 타입 정의
└── utils/              # 유틸리티 함수
```

## API 서비스 레이어 패턴
```typescript
// services/api.ts
import axios from 'axios';

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  timeout: 10000,
});

// 요청 인터셉터: JWT 자동 주입
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// 응답 인터셉터: 401 자동 처리
api.interceptors.response.use(
  (res) => res,
  (err) => {
    if (err.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(err);
  }
);

export default api;
```

## 필수 패키지
```json
{
  "axios": "^1.x",
  "react-router-dom": "^6.x",
  "zustand": "^4.x"     // 경량 상태 관리 (필요 시)
}
```

## 환경변수
```env
VITE_API_URL=http://localhost:8000
```
