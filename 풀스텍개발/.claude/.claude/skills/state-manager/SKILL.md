# state-manager 스킬

## 목적
`output/{run_id}/state.json` 파일을 읽고 쓰는 일관된 방법을 제공한다.
모든 에이전트가 이 스킬을 사용하여 상태를 관리한다.

---

## 읽기

```python
import json

def read_state(run_id: str) -> dict:
    path = f"output/{run_id}/state.json"
    with open(path, "r") as f:
        return json.load(f)
```

```bash
# Bash에서 읽기
cat output/$RUN_ID/state.json
```

---

## 쓰기 (특정 필드만 업데이트)

```python
import json

def update_state(run_id: str, updates: dict):
    path = f"output/{run_id}/state.json"
    with open(path, "r") as f:
        state = json.load(f)
    state.update(updates)
    with open(path, "w") as f:
        json.dump(state, f, indent=2, ensure_ascii=False)
```

```bash
# Bash에서 jq로 업데이트 (jq 설치 필요)
jq '.be_status = "done"' output/$RUN_ID/state.json > /tmp/state_tmp.json
mv /tmp/state_tmp.json output/$RUN_ID/state.json
```

---

## 에이전트별 갱신 책임

| 에이전트 | 갱신 가능한 필드 |
|---------|---------------|
| 메인 에이전트 | `current_step`, `gates_passed`, `mode`, `code_review_enabled`, `retries` |
| backend-developer | `be_status`만 |
| frontend-developer | `fe_status`만 |
| 기타 서브에이전트 | 읽기 전용 (갱신 필요 시 메인에 요청) |

---

## 완료 마커 파일 생성

```bash
# BE 완료 마커
touch output/$RUN_ID/logs/be.done

# FE 완료 마커
touch output/$RUN_ID/logs/fe.done
```

## 완료 마커 확인 (메인 에이전트용)

```bash
# 두 파일이 모두 존재하면 병렬 작업 완료
if [ -f "output/$RUN_ID/logs/be.done" ] && [ -f "output/$RUN_ID/logs/fe.done" ]; then
  echo "병렬 작업 완료"
fi
```

---

## state.json 스키마 참조

```json
{
  "run_id": "YYYY-MM-DD-HHMMSS",
  "mode": "A",
  "current_step": "requirements | architect | coding | code_review | qa | devops | done",
  "code_review_enabled": true,
  "demo_gate_enabled": false,
  "deploy_target": "compose",
  "gates_passed": ["G1", "G2"],
  "be_status": "pending | running | done | failed",
  "fe_status": "pending | running | done | failed",
  "retries": {
    "requirements": 0,
    "architect": 0,
    "code": 0,
    "devops": 0
  }
}
```
