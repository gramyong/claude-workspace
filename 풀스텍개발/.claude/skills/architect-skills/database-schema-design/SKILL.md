# schema-designer 스킬

## 목적
요구사항과 API 스펙을 기반으로 DB 스키마를 설계하고 SQL DDL을 생성한다.

---

## 설계 원칙

### 필수 컬럼 (모든 테이블 공통)
```sql
id          SERIAL PRIMARY KEY,           -- 또는 UUID
created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
updated_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
```

### 소프트 삭제 (삭제 가능한 리소스)
```sql
deleted_at  TIMESTAMP WITH TIME ZONE DEFAULT NULL
```
삭제 시 `deleted_at`에 타임스탬프 기록. 조회 시 `WHERE deleted_at IS NULL` 조건 추가.

### 사용자 테이블 (인증 필요 시)
```sql
CREATE TABLE users (
    id          SERIAL PRIMARY KEY,
    email       VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role        VARCHAR(50) NOT NULL DEFAULT 'user',
    is_active   BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
```

---

## 관계 패턴

### 1:N 관계
```sql
-- posts 테이블 (N 쪽)
user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
```

### N:M 관계 (중간 테이블)
```sql
CREATE TABLE user_roles (
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id INTEGER NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);
```

---

## schema.sql 작성 형식

```sql
-- =============================================================
-- {테이블 설명}
-- =============================================================
CREATE TABLE {table_name} (
    id          SERIAL PRIMARY KEY,
    -- 비즈니스 필드
    {field_name} {TYPE} {CONSTRAINTS},
    -- 공통 필드
    created_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 인덱스
CREATE INDEX idx_{table}_{field} ON {table}({field});

-- updated_at 자동 갱신 트리거 (선택적)
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_{table}_updated_at
BEFORE UPDATE ON {table}
FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

---

## 검증 방법

```bash
# PostgreSQL 문법 검증 (psql 설치 시)
psql -c "\i schema.sql" -d postgres --dry-run

# 또는 Python으로 파싱 검증
python3 -c "
import re
with open('schema.sql') as f:
    content = f.read()
# 기본 문법 체크: CREATE TABLE 블록 존재 여부
tables = re.findall(r'CREATE TABLE (\w+)', content, re.IGNORECASE)
print(f'테이블 {len(tables)}개 발견: {tables}')
"
```

---

## SQLite 대안 (단순 프로젝트)

PostgreSQL 대신 SQLite 사용 시 차이점:
- `SERIAL PRIMARY KEY` → `INTEGER PRIMARY KEY AUTOINCREMENT`
- `TIMESTAMP WITH TIME ZONE` → `DATETIME`
- `BOOLEAN` → `INTEGER` (0/1)
- 트리거 문법 약간 다름
