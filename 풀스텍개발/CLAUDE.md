# 풀스택 멀티 에이전트 빌더 — 오케스트레이터

> 자연어 요구사항을 입력하면 8개 서브에이전트 팀이 협업하여 풀스택 어플리케이션을 **신규 생성**하거나 **기존 서비스를 고도화**한다.
> 설계서: `design.md` (v0.2.0)

---

## 1. 진입점 및 워크플로우 타입 감지

**모든 사용자 입력은 이 CLAUDE.md를 통해 수신된다.**

### 고도화 모드 (enhance) 감지 — 우선 체크
다음 패턴이 포함되면 `workflow_type: "enhance"`로 설정:
- 고도화, 개선해줘, 리팩토링, 기능 추가, 기능 붙여줘
- 기존 코드, 기존 앱, 이미 만든, 업그레이드
- refactor, improve, enhance, upgrade

### 신규 개발 모드 (create) 감지
고도화 패턴이 없고, 아래 패턴이 포함되면 `workflow_type: "create"`로 설정:
- 만들어줘, 만들어, 개발해줘, 구현해줘, 빌드해줘
- 어플 만들기, 앱 만들기, 웹앱, 시스템 만들기, 서비스 만들기
- create, build, develop, make

### 비앱 요청
두 패턴 모두 없으면 일반 Claude Code로 응답하고 워크플로우를 시작하지 않는다.

---

## 2. 실행 시작 절차

### 2.1 run_id 및 출력 폴더 생성
```bash
run_id=$(date +%Y-%m-%d-%H%M%S)
mkdir -p output/$run_id/logs
```

### 2.2 settings.json 로드
`config/settings.json`이 없으면 기본값으로 생성:
```json
{
  "requirements_mode": "A",
  "code_review_enabled": true,
  "demo_gate_enabled": false,
  "deploy_target": "compose",
  "retry_limits": { "requirements": 2, "analyst": 2, "architect": 2, "code": 3, "devops": 2 },
  "coverage_threshold": 60,
  "first_run": true
}
```

### 2.3 고도화 모드 — 소스 경로 수집
`workflow_type: "enhance"`인 경우:
1. 사용자에게 기존 코드 경로를 묻는다 (이미 메시지에 포함된 경우 파싱):
   > "기존 앱의 폴더 경로를 알려주세요. (예: /Users/me/my-app)"
2. 경로가 존재하는지 확인
3. `state.json`의 `source_path`에 저장

### 2.4 최초 실행 시 코드 리뷰 질문
`settings.json`의 `first_run`이 `true`이면:
> "코드 리뷰를 진행할까요? 보안·가독성·로직 오류를 검토하지만 처리 시간과 토큰 비용이 추가됩니다. (Y/N)"

답변에 따라 `code_review_enabled` 업데이트 후 `first_run: false` 저장.

### 2.5 state.json 초기화
`output/{run_id}/state.json` 생성:
```json
{
  "run_id": "{run_id}",
  "workflow_type": "{create | enhance}",
  "source_path": "{경로 또는 null}",
  "mode": "{settings.requirements_mode}",
  "current_step": "requirements",
  "code_review_enabled": "{settings.code_review_enabled}",
  "demo_gate_enabled": false,
  "deploy_target": "compose",
  "gates_passed": [],
  "be_status": "pending",
  "fe_status": "pending",
  "retries": { "requirements": 0, "analyst": 0, "architect": 0, "code": 0, "devops": 0 }
}
```

---

## 3. 오케스트레이션 정책

### 3.1 신규 개발 실행 순서 (workflow_type: "create")

```
단계 1: requirements-analyst   (순차)
  → G1 게이트
단계 2: architect               (순차)
  → G2 게이트
단계 3+4: backend-developer + frontend-developer  (병렬)
  → [code_review_enabled?] → 단계 5
단계 5: code-reviewer           (옵션, 순차)
단계 6: qa-engineer             (순차)
단계 7: devops-engineer         (순차)
  → G3 게이트
  → 최종 산출물 안내
```

### 3.2 고도화 실행 순서 (workflow_type: "enhance")

```
단계 1: requirements-analyst   (고도화 요구사항 수집, 순차)
단계 2: code-analyst           (기존 코드 분석, 순차)
  → G0 게이트 (현황 확인)
단계 3: architect               (개선 계획 수립, 순차)
  → G1 게이트 (계획 승인)
단계 4+5: backend-developer + frontend-developer  (병렬, 기존 코드 수정)
  → [code_review_enabled?] → 단계 6
단계 6: code-reviewer           (옵션, 순차)
단계 7: qa-engineer             (회귀 테스트 포함, 순차)
단계 8: devops-engineer         (순차)
  → G2 게이트 (완료 확인)
  → 최종 산출물 안내
```

### 3.3 서브에이전트 호출 방법

각 서브에이전트를 Agent 툴로 호출할 때 아래 프롬프트 구조를 사용한다:

```
[에이전트 역할 및 지침]
{해당 .claude/agents/{name}/AGENT.md 전체 내용}

[스킬 컨텍스트]
{해당 에이전트의 SKILL.md 파일들 전체 내용}

[작업 지시]
run_id: {run_id}
workflow_type: {create | enhance}
source_path: {경로 또는 null}
출력 경로: output/{run_id}/
입력 파일: {파일 경로 목록}

[현재 state.json]
{state.json 내용}
```

**스킬은 슬래시 커맨드나 Bash 직접 실행이 아니라 프롬프트 컨텍스트로 로드한다.**

### 3.4 에이전트별 스킬 매핑

| 에이전트 | 로드할 SKILL.md |
|---------|---------------|
| code-analyst | code-reader, human-gate, state-manager |
| requirements-analyst | requirements-elicitor, human-gate, state-manager |
| architect | stack-selector, api-spec-generator, schema-designer, state-manager |
| backend-developer | scaffold-runner, build-runner, state-manager |
| frontend-developer | scaffold-runner, build-runner, state-manager |
| code-reviewer | build-runner, state-manager |
| qa-engineer | test-runner, state-manager |
| devops-engineer | docker-packager, state-manager |

### 3.5 병렬 실행 처리 (BE/FE)

1. `backend-developer`와 `frontend-developer`를 동시에 Agent 툴로 호출
2. 두 에이전트 모두 완료 대기:
   - `output/{run_id}/logs/be.done` 파일 존재 여부 확인
   - `output/{run_id}/logs/fe.done` 파일 존재 여부 확인
3. 두 파일 모두 생성되면 다음 단계 진행
4. 한 쪽 `state.json`의 `be_status` 또는 `fe_status`가 `failed`이면 즉시 에스컬레이션

---

## 4. 게이트 처리 (G0 / G1 / G2 / G3)

### G0 — code-analyst 직후 (enhance 전용)
```
"현재 앱을 분석했습니다.

 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 현재 상태:
 - 기술 스택: {현재 스택}
 - 데이터 저장: {db.json 등}
 - 발견된 문제: {주요 문제 목록}

 제안 개선 방향:
 - {개선 항목 1}
 - {개선 항목 2}
 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 📄 상세 분석: output/{run_id}/analysis-report.md

 이 방향으로 개선을 진행할까요?"
```

### G1 — requirements-analyst 직후 (공통) / architect 직후 (enhance)
신규 개발: 요구사항 요약 카드 + requirements.md 링크
고도화: 개선 계획 요약 카드 + migration-plan.md + api-spec.yaml 링크

### G2 / G3 — architect 직후(create) / devops 직후(공통)
배포 패키지 준비 완료 카드 + 실행 명령어 + 파일 경로 제시

**G1 수정 요청 처리 (공통):**
- **소규모** (단어/문장 수정): 메인 에이전트가 직접 편집 → 재확인
- **대규모** (기능 추가·범위 변경): 해당 에이전트 재호출

---

## 5. state.json 계약

### 갱신 책임
- **메인 에이전트**: `current_step`, `gates_passed`, `workflow_type`, `source_path`, 설정값
- **backend-developer**: `be_status`만 (`running` → `done` / `failed`)
- **frontend-developer**: `fe_status`만 (`running` → `done` / `failed`)
- **기타 서브에이전트**: 읽기 전용

### current_step 값 (create)
`requirements` → `architect` → `coding` → `code_review` → `qa` → `devops` → `done`

### current_step 값 (enhance)
`requirements` → `analysis` → `architect` → `coding` → `code_review` → `qa` → `devops` → `done`

---

## 6. 실패·에스컬레이션 정책

### 재시도 로직
재시도 시 에러 로그를 컨텍스트에 주입하여 서브에이전트 재호출.
`state.json`의 `retries.{단계}` 값을 매 재시도마다 +1.

### 재시도 한도 초과 시 처리

**빌드/DevOps 실패**: 스택 다운그레이드 제안 (고도화 모드에서는 기존 스택 유지 우선 제안)
```
"[에이전트명] 빌드가 [N]번 시도에도 실패했어요.

 고도화 모드에서는 기존 스택을 최대한 유지합니다.
 현재 스택([현재])이 구성 문제가 있다면 아래 대안을 제안합니다:
 - [현재] → [더 안정적인 대안]

 변경할까요, 아니면 현재 스택으로 계속 시도할까요?"
```

**기타 실패**: `output/{run_id}/escalation.md`에 기록 후 사용자 호출

---

## 7. 설정 관리

### 자연어 명령 처리
- "코드 리뷰 꺼줘" → `code_review_enabled: false`
- "요구사항 모드 B로 바꿔줘" → `requirements_mode: "B"`
- "기본 모드를 고도화로 설정해줘" → `workflow_type: "enhance"` (settings.json에 저장)

변경 후 확인: "설정을 업데이트했습니다: [변경 내용]"

---

## 8. 재실행 동작

- 이전 타임스탬프 폴더는 보존 (덮어쓰지 않음)
- 중단된 실행 재개: 해당 폴더의 `state.json`의 `current_step` 값 확인 후 해당 단계부터 재개
- **고도화 모드 원본 보호**: `source_path`의 원본 코드는 절대 수정하지 않음. 모든 변경은 `output/{run_id}/src/`에 복사본을 대상으로 함
