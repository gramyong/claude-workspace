# test-runner 스킬

## 목적
단위·통합 테스트를 실행하고 커버리지를 측정한다.
결과를 `test-report.md`로 정리한다.

---

## 백엔드 테스트 (Python)

### pytest 설치 및 실행
```bash
cd output/{run_id}/src/backend

# 설치
pip install pytest pytest-cov pytest-asyncio httpx

# 실행
pytest tests/ -v --cov=. --cov-report=term-missing --cov-report=json:coverage.json
# 종료 코드 0 = 모두 통과
```

### 커버리지 확인
```bash
# 전체 커버리지
python -c "
import json
with open('coverage.json') as f:
    data = json.load(f)
total = data['totals']['percent_covered']
print(f'전체 커버리지: {total:.1f}%')
threshold = 60  # settings.json의 coverage_threshold
print('통과' if total >= threshold else '미달')
"
```

### FastAPI 통합 테스트 예시
```python
# tests/test_api.py
import pytest
from httpx import AsyncClient
from main import app

@pytest.mark.asyncio
async def test_health():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.get("/health")
    assert response.status_code == 200
    assert response.json()["status"] == "ok"
```

---

## 프론트엔드 테스트

### Vitest (React/Vue + Vite)
```bash
cd output/{run_id}/src/frontend
npm install -D vitest @testing-library/react @testing-library/user-event jsdom
# package.json에 test 스크립트 추가 후:
npm run test -- --coverage
```

---

## Smoke 테스트 스크립트 생성 위치

`output/{run_id}/tests/smoke/run_smoke.sh`

```bash
#!/bin/bash
# qa-engineer가 생성, devops-engineer가 compose up 중 실행
set -e
BASE_URL="${API_BASE_URL:-http://localhost:8000}"
PASS=0; FAIL=0

check() {
    local name="$1"; local expected_status="$2"
    local actual_status="$3"
    if [ "$actual_status" = "$expected_status" ]; then
        echo "✅ PASS: $name"
        PASS=$((PASS+1))
    else
        echo "❌ FAIL: $name (expected $expected_status, got $actual_status)"
        FAIL=$((FAIL+1))
    fi
}

echo "=== Smoke Test 시작: $BASE_URL ==="

# 헬스체크
STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$BASE_URL/health")
check "헬스체크" "200" "$STATUS"

# [각 핵심 엔드포인트별 추가]

echo ""
echo "=== 결과: $PASS 통과 / $FAIL 실패 ==="
[ $FAIL -eq 0 ] && echo "✅ 모든 Smoke 테스트 통과" && exit 0
echo "❌ 일부 테스트 실패" && exit 1
```

```bash
chmod +x output/{run_id}/tests/smoke/run_smoke.sh
```

---

## test-report.md 작성

```markdown
# 테스트 리포트

## 요약
| 항목 | 결과 |
|------|------|
| 전체 테스트 | {N}개 |
| 통과 | {N}개 |
| 실패 | {N}개 |
| 전체 커버리지 | {N}% |
| 커버리지 임계 | {N}% (통과/미달) |

## 실패 테스트
| 테스트명 | 실패 원인 |
|---------|---------|
| ... | ... |

## Smoke 테스트
위치: `tests/smoke/run_smoke.sh`
실행: devops-engineer가 compose up 이후 실행
```

---

## 성공 기준
1. 모든 테스트 종료 코드 0
2. 커버리지 ≥ `settings.json`의 `coverage_threshold` (기본 60%)
3. `tests/smoke/run_smoke.sh` 파일 존재
