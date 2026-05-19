# stack-selector 스킬

## 목적
요구사항을 분석하여 최적의 기술 스택을 선정하고, 선정 근거를 문서화한다.

---

## 스택 선정 매트릭스

### 백엔드

| 요구사항 특성 | 추천 스택 | 이유 |
|-------------|----------|------|
| 기본 CRUD, 단순 API | FastAPI (Python) | 빠른 개발, OpenAPI 자동 생성 |
| 기능 매우 단순 (3개 이하) | Flask (Python) | 최소 설정, 가벼움 |
| 실시간 기능 (채팅, 알림) | FastAPI + WebSocket | FastAPI 기본 지원 |
| JavaScript 선호 / Node 생태계 | Express.js (Node.js) | 프론트와 언어 통일 |
| 완전한 ORM, 관리자 패널 필요 | Django (Python) | 배터리 포함 |
| 고성능, 대용량 트래픽 | FastAPI | async 기본 지원 |

### 프론트엔드

| 요구사항 특성 | 추천 스택 | 이유 |
|-------------|----------|------|
| 기본 SPA, 컴포넌트 기반 | React + TypeScript | 생태계 최대, 타입 안정성 |
| 단순 페이지 (3개 이하) | React + TypeScript | 동일, 오버킬 아님 |
| 빠른 개발, 설정 적게 | Vue.js 3 + TypeScript | 학습 곡선 낮음 |
| 성능 극대화, 번들 크기 | Svelte + TypeScript | 컴파일 타임 반응성 |
| 기존 레거시 HTML 확장 | vanilla HTML + Alpine.js | 최소 변경 |

### 데이터베이스

| 요구사항 특성 | 추천 DB | 이유 |
|-------------|--------|------|
| 기본 (대부분의 경우) | PostgreSQL | 관계형, JSON 지원, 안정성 |
| 단순 / 로컬 전용 | SQLite | 설치 불필요, 파일 기반 |
| 유연한 스키마 필요 | PostgreSQL (JSONB) | 관계형 + 문서형 혼용 |
| 고속 캐시 / 세션 | Redis (보조) | 메인 DB 보조로 추가 |

### 인증

| 요구사항 특성 | 추천 방식 |
|-------------|---------|
| 기본 (대부분) | JWT (Access + Refresh Token) |
| 단순 내부 도구 | Session-based |
| SSO 연동 필요 | OAuth 2.0 + 기업 IdP |

---

## 복잡도별 기본 스택 조합

### 낮음 (기능 3개 이하, 사용자 2종 이하)
```
백엔드: Flask + SQLite
프론트엔드: React + TypeScript
인증: JWT (선택적)
컨테이너: Docker Compose (서비스 2개)
```

### 중간 (기능 4~7개, 역할 기반 접근 제어)
```
백엔드: FastAPI + PostgreSQL
프론트엔드: React + TypeScript
인증: JWT (Access + Refresh)
컨테이너: Docker Compose (서비스 3개)
```

### 높음 (기능 8개 이상, 실시간, 대규모)
```
백엔드: FastAPI + PostgreSQL + Redis
프론트엔드: React + TypeScript
인증: JWT + OAuth 2.0
컨테이너: Docker Compose (서비스 4개 이상)
```

---

## 선정 근거 문서화 템플릿

stack-decision.md 작성 시 반드시 포함:

```markdown
## 백엔드: [선정 스택]
- **선택 이유**: 요구사항의 [특성]이 이 스택에 적합하기 때문
- **대안 검토**: [대안 A]는 [이유]로 제외, [대안 B]는 [이유]로 제외
- **리스크**: [알려진 단점 또는 주의사항]
```
