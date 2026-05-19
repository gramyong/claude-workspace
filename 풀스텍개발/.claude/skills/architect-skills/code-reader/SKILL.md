# code-reader 스킬

## 목적
기존 코드베이스의 구조를 파악하고, 의존성과 데이터 레이어를 분석한다.
code-analyst 에이전트가 사용한다.

---

## 파일 트리 생성

```bash
# node_modules, .git, dist, build 제외
find {source_path} \
  -not -path "*/node_modules/*" \
  -not -path "*/.git/*" \
  -not -path "*/dist/*" \
  -not -path "*/build/*" \
  -not -path "*/.venv/*" \
  -not -path "*/__pycache__/*" \
  | sort

# 또는 tree 명령 (설치된 경우)
tree {source_path} -I "node_modules|.git|dist|build|.venv|__pycache__" --filelimit 50
```

---

## 의존성 분석

### Node.js (package.json)
```bash
cat {source_path}/package.json

# 주요 의존성만 추출
node -e "
const pkg = require('{source_path}/package.json');
console.log('이름:', pkg.name);
console.log('버전:', pkg.version);
console.log('진입점:', pkg.main || 'index.js');
console.log('의존성:', Object.keys(pkg.dependencies || {}).join(', '));
console.log('devDependencies:', Object.keys(pkg.devDependencies || {}).join(', '));
"
```

### Python (requirements.txt)
```bash
cat {source_path}/requirements.txt
# 또는
cat {source_path}/pyproject.toml
```

---

## db.json 패턴 감지

```bash
# db.json 파일 존재 여부
find {source_path} -name "*.json" -not -path "*/node_modules/*" | grep -E "(db|data|store|database)"

# lowdb 사용 패턴 감지
grep -r "lowdb\|FileSync\|JSONFile\|JSONSyncAdapter" {source_path} \
  --include="*.js" --include="*.ts" \
  --exclude-dir=node_modules -l

# fs.readFile/writeFile로 JSON을 DB처럼 사용하는 패턴
grep -r "readFileSync\|writeFileSync\|readFile\|writeFile" {source_path} \
  --include="*.js" --include="*.ts" \
  --exclude-dir=node_modules -l

# db.json 현재 구조 (있는 경우)
# 주의: 민감한 데이터가 있을 수 있으므로 구조만 파악
```

### db.json 스키마 역설계
```javascript
// db.json이 있는 경우 구조 분석
const fs = require('fs');
const db = JSON.parse(fs.readFileSync('{source_path}/db.json', 'utf8'));

// 각 키의 타입과 첫 번째 레코드만 출력 (실제 데이터 제외)
Object.keys(db).forEach(key => {
  const value = db[key];
  if (Array.isArray(value) && value.length > 0) {
    console.log(`${key} (배열, ${value.length}건): 필드 = ${Object.keys(value[0]).join(', ')}`);
  } else {
    console.log(`${key}: ${typeof value}`);
  }
});
```

---

## API 엔드포인트 추출

### Express.js 패턴
```bash
# 라우터 파일 찾기
find {source_path} -name "*.js" -o -name "*.ts" | \
  xargs grep -l "router\.\|app\.get\|app\.post\|app\.put\|app\.delete\|app\.patch" \
  --exclude-dir=node_modules 2>/dev/null

# 엔드포인트 패턴 추출
grep -rn "router\.\(get\|post\|put\|delete\|patch\)\|app\.\(get\|post\|put\|delete\|patch\)" \
  {source_path} \
  --include="*.js" --include="*.ts" \
  --exclude-dir=node_modules
```

---

## Docker 구성 파악

```bash
# Docker 파일 확인
ls -la {source_path}/Dockerfile* {source_path}/docker-compose* 2>/dev/null

# 포트 및 서비스 구조 파악
cat {source_path}/docker-compose.yml 2>/dev/null || echo "docker-compose 없음"
cat {source_path}/Dockerfile 2>/dev/null || echo "Dockerfile 없음"
```

---

## 환경변수 파악

```bash
# .env.example 또는 .env 파일 확인 (실제 값은 로깅하지 않음)
cat {source_path}/.env.example 2>/dev/null || \
cat {source_path}/.env.sample 2>/dev/null || \
echo ".env.example 없음"

# 코드에서 process.env 사용 목록 추출
grep -rn "process\.env\." {source_path} \
  --include="*.js" --include="*.ts" \
  --exclude-dir=node_modules | \
  grep -oP "process\.env\.\w+" | sort | uniq
```

---

## 보안 취약점 빠른 스캔

```bash
# 하드코딩된 시크릿 패턴 (경고용)
grep -rn \
  -e "password\s*=\s*['\"]" \
  -e "secret\s*=\s*['\"]" \
  -e "api_key\s*=\s*['\"]" \
  {source_path} \
  --include="*.js" --include="*.ts" \
  --exclude-dir=node_modules \
  --exclude="*.example" 2>/dev/null | head -20

# SQL 인젝션 취약 패턴
grep -rn "query\s*=.*+\s*req\|query\s*=.*\`.*\$" \
  {source_path} \
  --include="*.js" --include="*.ts" \
  --exclude-dir=node_modules 2>/dev/null | head -10
```

---

## 마이그레이션 대상 DB 판단 기준

| 조건 | 권장 마이그레이션 대상 |
|------|-------------------|
| db.json 데이터 1,000건 미만, 단일 서버 | SQLite (파일 기반, 설치 불필요) |
| db.json 데이터 1,000건 이상, 또는 동시 접속 필요 | PostgreSQL |
| 기존에 관계형 쿼리 많음 | PostgreSQL |
| 추후 확장 가능성 높음 | PostgreSQL (Docker Compose에 포함) |
