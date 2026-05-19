# build-runner 스킬

## 목적
백엔드·프론트엔드 코드의 빌드·타입체크·린트를 실행하고 결과를 보고한다.
종료 코드 0이 모든 검사의 성공 기준이다.

---

## 백엔드 (Python)

### 의존성 설치
```bash
cd output/{run_id}/src/backend

# uv 사용 시
uv sync

# pip 사용 시
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 타입체크 (mypy)
```bash
pip install mypy
mypy . --ignore-missing-imports --exclude '.venv'
# 종료 코드 0 = 통과
```

### 린트 (flake8 또는 ruff)
```bash
# flake8
pip install flake8
flake8 . --exclude=.venv --max-line-length=120

# 또는 ruff (더 빠름)
pip install ruff
ruff check .
```

---

## 프론트엔드 (React/Vue + TypeScript)

### 의존성 설치
```bash
cd output/{run_id}/src/frontend
npm install
```

### 타입체크
```bash
npx tsc --noEmit
# 종료 코드 0 = 통과
```

### 린트 (ESLint)
```bash
npx eslint src --ext .ts,.tsx,.vue
# 종료 코드 0 = 에러 없음
```

### 빌드
```bash
npm run build
# 종료 코드 0 = 빌드 성공
```

---

## 실패 시 처리

1. 에러 출력을 `logs/build_error.log`에 저장
2. 에러 메시지를 분석하여 수정 시도
3. 수정 후 재실행
4. 최대 재시도 횟수(기본 3회) 초과 시 메인 에이전트에 보고

```bash
# 에러 로그 저장 예시
npm run build 2>&1 | tee output/$RUN_ID/logs/fe_build.log
echo "Exit code: $?"
```

---

## 정적 분석 (code-reviewer용)

```bash
# Python 보안 취약점
pip install bandit
bandit -r . --exclude .venv -f json -o security_report.json

# JavaScript 취약점
npm audit --json > npm_audit.json
```

---

## 성공 판단 기준

| 검사 | 성공 조건 |
|------|---------|
| 의존성 설치 | 종료 코드 0 |
| 타입체크 | 종료 코드 0 |
| 린트 | 종료 코드 0 (에러 0건) |
| 빌드 | 종료 코드 0 |
