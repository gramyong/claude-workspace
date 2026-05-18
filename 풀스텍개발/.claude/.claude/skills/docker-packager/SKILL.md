# docker-packager 스킬

## 목적
Dockerfile과 docker-compose.yml을 생성하고, 빌드·헬스체크·smoke test까지 실행하여 검증한다.

---

## Dockerfile 패턴

### FastAPI (멀티스테이지)
```dockerfile
# Build stage
FROM python:3.12-slim AS builder
WORKDIR /app
RUN pip install uv
COPY requirements.txt .
RUN uv pip install --system -r requirements.txt

# Runtime stage
FROM python:3.12-slim
RUN adduser --disabled-password --gecos "" appuser
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.12 /usr/local/lib/python3.12
COPY --from=builder /usr/local/bin /usr/local/bin
COPY . .
RUN chown -R appuser:appuser /app
USER appuser
EXPOSE 8000
HEALTHCHECK --interval=10s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### React (빌드 + Nginx)
```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json .
RUN npm ci
COPY . .
RUN npm run build

# Runtime stage
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
HEALTHCHECK --interval=10s --timeout=3s \
  CMD wget -qO- http://localhost/health || exit 1
```

### nginx.conf (React SPA용)
```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /health {
        return 200 'ok';
        add_header Content-Type text/plain;
    }
}
```

---

## docker-compose.yml 기본 구조

```yaml
version: "3.9"

services:
  backend:
    build:
      context: ./src/backend
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://appuser:${DB_PASSWORD:-changeme}@db:5432/appdb
      JWT_SECRET: ${JWT_SECRET:-dev-secret-change-in-production}
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 10s

  frontend:
    build:
      context: ./src/frontend
      dockerfile: Dockerfile
    ports:
      - "3000:80"
    depends_on:
      - backend
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: ${DB_PASSWORD:-changeme}
    volumes:
      - db_data:/var/lib/postgresql/data
      - ./schema.sql:/docker-entrypoint-initdb.d/01_schema.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: unless-stopped

volumes:
  db_data:
```

---

## 검증 명령어

```bash
cd output/$RUN_ID

# 1. docker build
docker compose build --no-cache

# 2. compose config 검증
docker compose config --quiet && echo "Config valid"

# 3. compose up + 헬스체크 대기
docker compose up -d
sleep 15  # 서비스 시작 대기

# 서비스 상태 확인
docker compose ps
docker compose ps | grep -E "unhealthy|Exit" && echo "UNHEALTHY" && exit 1

# 4. smoke test 실행
API_BASE_URL=http://localhost:8000 bash tests/smoke/run_smoke.sh

# 5. 정리 (검증 후)
docker compose down
```

---

## .dockerignore 기본 내용

```
.git
.env
node_modules
__pycache__
.venv
*.pyc
*.pyo
dist
build
.pytest_cache
```
