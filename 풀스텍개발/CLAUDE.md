# 풀스택 멀티 에이전트 빌더 — 오케스트레이터

> 자연어 요구사항을 입력하면 3개 서브에이전트(architect / developer / reviewer)가 협업하여
> 풀스택 어플리케이션을 **신규 생성**하거나 **기존 서비스를 고도화**한다.
> 설계서: `design.md` (v0.3.0)

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
  "requirements_mode": "C",
  "deploy_target": "compose",
  "retry_limits": { "architect": 2, "developer": 3, "reviewer": 2 },
  "coverage_threshold": 80,
  "first_run": true
}
```

### 2.3 고도화 모드 — 소스 경로 수집
`workflow_type: "enhance"`인 경우:
1. 사용자에게 기존 코드 경로를 묻는다 (이미 메시지에 포함된 경우 파싱):
   > "기존 앱의 폴더 경로를 알려주세요. (예: /Users/me/my-app)"
2. 경로가 존재하는지 확인
3. `state.json`의 `source_path`에 저장

### 2.4 state.json 초기화
`output/{run_id}/state.json` 생성:
```json
{
  "run_id": "{run_id}",
  "workflow_type": "{create | enhance}",
  "source_path": "{경로 또는 null}",
  "current_step": "architect",
  "gates_passed": [],
  "retries": { "architect": 0, "developer": 0, "reviewer": 0 }
}
```

---

## 3. 오케스트레이션 정책

### 3.1 신규 개발 실행 순서 (workflow_type: "create")

```
단계 1: architect   (요구사항 수집 + 기술 설계)
  → G1 게이트 ("이 설계로 진행할까요?")
단계 2: developer   (BE + FE + 테스트 + Docker)
  → G2 게이트 ("구현 완료, 검증 진행할까요?")
단계 3: reviewer    (코드 리뷰 + 보안 + 성능 + 테스트 실행)
  → G3 게이트 ("최종 확정할까요?")
  → 최종 산출물 안내
```

### 3.2 고도화 실행 순서 (workflow_type: "enhance")

```
단계 1: architect   (고도화 요구사항 수집 + 기존 코드 분석 + 개선 계획)
  → G0 게이트 ("현재 이런 상태, 이렇게 고칠까요?")
  → G1 게이트 ("이 계획으로 진행할까요?")
단계 2: developer   (기존 코드 수정 + 신규 기능 + DB 마이그레이션 + Docker)
  → G2 게이트 ("변경 내용 확인, 검증 진행할까요?")
단계 3: reviewer    (회귀 테스트 + 신규 테스트 + 보안·성능 검토)
  → G3 게이트 ("최종 확정할까요?")
  → 최종 산출물 안내
```

### 3.3 서브에이전트 호출 방법

각 서브에이전트를 Agent 툴로 호출할 때 아래 프롬프트 구조를 사용한다:

```
[에이전트 역할 및 지침]
{해당 .claude/agents/{name}/AGENT.md 전체 내용}

[스킬 컨텍스트]
{해당 에이전트 스킬 그룹의 SKILL.md 파일들 전체 내용}

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

| 에이전트 | 로드할 스킬 그룹 | 공통 스킬 |
|---------|---------------|---------|
| architect | `.claude/skills/architect-skills/` 전체 | human-gate, state-manager |
| developer | `.claude/skills/developer-skills/` 전체 | human-gate, state-manager |
| reviewer  | `.claude/skills/reviewer-skills/` 전체  | human-gate, state-manager |

---

## 4. 게이트 처리 (G0 / G1 / G2 / G3)

### G0 — architect 직후 (enhance 전용)
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

### G1 — architect 직후 (공통)
```
"설계가 완료되었습니다.

 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 기술 스택: {선정 스택}
 핵심 기능: {기능 목록}
 API 엔드포인트: {N}개
 DB 테이블: {N}개
 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 📄 상세 설계서: output/{run_id}/design.md

 이 설계로 구현을 시작할까요?"
```

### G2 — developer 직후 (공통)
```
"구현이 완료되었습니다.

 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 백엔드: ✅ 빌드 성공 / 린트 통과
 프론트엔드: ✅ 빌드 성공 / 린트 통과
 Docker: ✅ 이미지 빌드 성공
 테스트: ✅ {N}개 작성 완료
 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 코드 리뷰·보안 스캔·테스트 실행을 진행할까요?"
```

### G3 — reviewer 직후 (공통)
```
"검증이 완료되었습니다.

 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 Critical 이슈: 0건 ✅
 테스트: ✅ 전체 통과 / 커버리지 {N}%
 보안: ✅ 취약점 없음
 Smoke 테스트: ✅ {N}/{N} 통과
 ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 실행 방법:
   cd output/{run_id}
   cp .env.example .env
   docker compose up -d

 이대로 최종 확정할까요?"
```

**G1 수정 요청 처리:**
- **소규모** (단어·문장 수정): 메인 에이전트가 `design.md` 직접 수정 → 재확인
- **대규모** (기능 추가·범위 변경): `architect` 재호출

---

## 5. state.json 계약

### 갱신 책임
- **메인 에이전트**: `current_step`, `gates_passed`, `workflow_type`, `source_path`
- **서브에이전트**: 각자의 완료/실패 상태를 `state.json`에 기록 후 메인에 보고

### current_step 값 (create)
`architect` → `coding` → `review` → `done`

### current_step 값 (enhance)
`architect` → `coding` → `review` → `done`

---

## 6. 실패·에스컬레이션 정책

### 재시도 로직
재시도 시 에러 로그를 컨텍스트에 주입하여 서브에이전트 재호출.
`state.json`의 `retries.{에이전트}` 값을 매 재시도마다 +1.

### 재시도 한도 초과 시 처리

**developer 빌드/Docker 실패**:
```
"[developer] 빌드가 [N]번 시도에도 실패했어요.

 더 안정적인 스택으로 변경을 제안합니다:
 - Next.js → vanilla HTML+JS
 - FastAPI → Flask

 변경할까요, 아니면 현재 스택으로 계속 시도할까요?"
```

**기타 실패**: `output/{run_id}/escalation.md`에 기록 후 사용자 호출

---

## 7. 설정 관리

### 자연어 명령 처리
- "요구사항 인터뷰로 바꿔줘" → `requirements_mode: "A"`
- "커버리지 기준 60%로 낮춰줘" → `coverage_threshold: 60`

변경 후 확인: "설정을 업데이트했습니다: [변경 내용]"

---

## 8. 재실행 동작

- 이전 타임스탬프 폴더는 보존 (덮어쓰지 않음)
- 중단된 실행 재개: 해당 폴더의 `state.json`의 `current_step`으로 이어달리기
- **고도화 모드 원본 보호**: `source_path`의 원본 코드는 절대 수정하지 않음
