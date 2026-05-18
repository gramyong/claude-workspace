# scaffold-runner 스킬

## 목적
선정된 기술 스택에 맞는 프로젝트 초기 구조를 생성한다.

---

## 스택별 초기화 명령

### 백엔드

**FastAPI (Python)**
```bash
# 프로젝트 폴더 생성
mkdir -p src/backend && cd src/backend

# uv 사용 (권장)
pip install uv
uv init .
uv add fastapi uvicorn sqlalchemy alembic psycopg2-binary python-jose[cryptography] passlib[bcrypt] python-dotenv pydantic-settings

# 또는 pip
python -m venv .venv
source .venv/bin/activate
pip install fastapi uvicorn sqlalchemy alembic psycopg2-binary python-jose passlib python-dotenv
pip freeze > requirements.txt
```

**Flask (Python)**
```bash
mkdir -p src/backend && cd src/backend
python -m venv .venv && source .venv/bin/activate
pip install flask flask-sqlalchemy flask-migrate flask-jwt-extended flask-cors python-dotenv
pip freeze > requirements.txt
```

**Express (Node.js)**
```bash
mkdir -p src/backend && cd src/backend
npm init -y
npm install express cors dotenv jsonwebtoken bcryptjs sequelize pg
npm install -D typescript @types/node @types/express ts-node nodemon
npx tsc --init
```

### 프론트엔드

**React + TypeScript (Vite)**
```bash
cd output/{run_id}/src
npm create vite@latest frontend -- --template react-ts
cd frontend && npm install
# 필수 라이브러리
npm install axios react-router-dom
npm install -D @types/react @types/react-dom
```

**Vue 3 + TypeScript (Vite)**
```bash
cd output/{run_id}/src
npm create vite@latest frontend -- --template vue-ts
cd frontend && npm install
npm install axios vue-router pinia
```

---

## 폴더 구조 생성 (FastAPI 예시)

```bash
mkdir -p src/backend/{routers,models,schemas,services,tests/unit}
touch src/backend/{main.py,database.py,config.py,.env.example}
```

**main.py 기본 구조:**
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(title="{앱 이름} API", version="1.0.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/health")
def health():
    return {"status": "ok"}
```

---

## .env.example 생성

백엔드:
```env
DATABASE_URL=postgresql://user:password@localhost:5432/dbname
JWT_SECRET=your-secret-key-change-this
JWT_ALGORITHM=HS256
JWT_EXPIRE_MINUTES=30
```

프론트엔드:
```env
VITE_API_URL=http://localhost:8000
```

---

## 주의사항
- 실제 `.env` 파일은 생성하지 않음 (`.env.example`만 생성)
- `node_modules/`, `__pycache__/`, `.venv/`는 `.gitignore`에 추가
- 초기화 후 반드시 build-runner 스킬로 빌드 성공 확인
