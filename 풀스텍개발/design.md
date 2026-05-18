# 풀스택 멀티 에이전트 빌더 설계서

> **자연어 요구사항을 입력하면, 8명의 에이전트 팀이 협업하여 컨테이너화된 풀스택 어플리케이션을 신규 생성하거나, 기존 서비스를 고도화하는 시스템.**

| 항목 | 값 |
|------|---|
| 버전 | v0.2.0 (고도화 모드 추가 — 2026-05-16) |
| 사용자 | 비개발자 PM(본인) + 타팀 비개발자 |
| 범위 | **신규 개발** (0→1) + **기존 서비스 고도화** (기능 추가·리팩토링·DB 마이그레이션) |
| 산출물 형태 | Docker Compose 기반 풀스택 어플리케이션 |
| 구현 플랫폼 | Claude Code (서브에이전트 + 스킬) |

---

## 1. 작업 컨텍스트

### 1.1 배경 및 목적
비개발자 사용자가 코드 작성 없이 풀스택 어플리케이션을 만들 수 있도록, **역할이 분리된 에이전트 팀이 협업**하는 시스템을 구축한다.

**달성 목표 (3가지 동시):**
1. **토큰 효율** — 긴 도메인 지침을 필요한 시점에만 로드 (서브에이전트 분리)
2. **결과물 속도** — BE/FE 등 독립 작업은 병렬 수행 가능
3. **품질 분리** — 각 역할의 전문성 격리로 실수 감소

### 1.2 범위
**포함 — 신규 개발 모드 (create):**
- 자연어 → 정제된 요구사항 명세
- 스택 선정 (요구사항 기반 자율 결정)
- 아키텍처·API·DB 스키마 설계
- 백엔드·프론트엔드 코드 생성
- 테스트 작성 및 실행
- Docker 이미지 + Compose 매니페스트 생성

**포함 — 고도화 모드 (enhance):**
- 기존 코드베이스 분석 및 현황 리포트
- 기능 추가·리팩토링·코드 안정화
- DB 마이그레이션 (db.json/파일 기반 → SQLite/PostgreSQL 등)
- 기존 Docker 구성 개선
- 회귀 테스트 (기존 기능 보호)

**제외 (v0.2 한정):**
- K8s/Helm/클라우드별 매니페스트 (확장 로드맵)
- 실제 클라우드 배포 실행
- BE/FE 통합 동작 데모 게이트 (확장 옵션)

### 1.3 입출력 정의

| 구분 | 신규 개발 (create) | 고도화 (enhance) |
|------|------------------|----------------|
| **입력** | 자연어 1차 요청 | 자연어 요청 + **기존 코드 경로** |
| **중간 산출물** | `requirements.md`, `api-spec.yaml`, `schema.sql` | `analysis-report.md`, `enhancement-requirements.md`, `migration-plan.md` |
| **최종 출력** | 새 코드베이스 + Docker 패키지 | **수정된** 코드베이스 + `migrations/` + Docker 패키지 |

> **출력 격리 정책:** 매 실행마다 타임스탬프 폴더(`/output/YYYY-MM-DD-HHMMSS/`)를 생성하여 이전 결과를 보존한다. 고도화 모드에서 수정된 코드도 이 폴더에 복사하여 원본을 보호한다. 중단된 실행은 해당 폴더의 `state.json` `current_step` 값으로 재개 가능하다.

### 1.4 제약 조건
- **사용자가 비개발자** → 코드 수준 리뷰 불가 → 게이트는 "자연어로 이해 가능한 시점"에 배치
- **사용자가 인프라 엔지니어** → 배포 매니페스트 검토는 가능 (G3 게이트는 사용자 강점 활용)
- **토큰 비용 의식적 관리** → 서브에이전트별 컨텍스트 격리, 데이터 전달은 파일 기반
- **확장성** → 요구사항 모드, 배포 타겟, code-reviewer 등을 설정으로 토글 가능

### 1.5 용어 정의

| 용어 | 정의 |
|------|------|
| **메인 에이전트** | `CLAUDE.md`에 정의된 오케스트레이터. 전체 흐름 제어, 서브에이전트 호출 |
| **서브에이전트** | `/.claude/agents/*/AGENT.md`에 정의된 역할별 전문 에이전트 |
| **스킬** | `/.claude/skills/*/SKILL.md`에 정의된 결정론적 도구 (스크립트 묶음) |
| **게이트(G0~G3)** | 사용자 승인이 필요한 워크플로우 정지점 (G0는 고도화 모드 전용) |
| **모드 (A/B/C)** | 요구사항 수집 방식. A=인터뷰, B=템플릿, C=하이브리드 |
| **workflow_type** | `create` (신규 개발) 또는 `enhance` (고도화). state.json에 저장 |
| **state.json** | 워크플로우 진행 상태와 설정을 담은 단일 진실 공급원 (SSOT) |

---

## 2. 워크플로우 정의

### 2.1 워크플로우 타입 분기

```
[사용자 요청 (자연어)]
        ↓
[워크플로우 감지: create(신규) / enhance(고도화)]
        ↓
  [create] ──────────────→ 2.2 신규 개발 흐름
  [enhance] ─────────────→ 2.3 고도화 흐름 (NEW)
```

**신규 개발 감지 키워드**: 만들어줘, 빌드해줘, 개발해줘, 새로 만들기
**고도화 감지 키워드**: 고도화, 개선해줘, 리팩토링, 기능 추가, 기존 코드, 업그레이드

---

### 2.2 신규 개발 흐름 (create)

```
[사용자 1차 요청 (자연어)]
        ↓
[요구사항 수집 모드 선택: A(인터뷰) / B(템플릿) / C(하이브리드)]
        ↓
┌─────────────────────────────────────────┐
│ 1. requirements-analyst                  │
│    → /output/requirements.md             │
└─────────────────────────────────────────┘
        ↓
[G1: 사용자 승인] "이게 원하시는 거 맞나요?"
        ↓
┌─────────────────────────────────────────┐
│ 2. architect                             │
│    → architecture.md / api-spec.yaml /   │
│      schema.sql / stack-decision.md      │
└─────────────────────────────────────────┘
        ↓
[G2: 사용자 승인] "이 스택·구조로 진행할까요?"
        ↓
┌──────────────────┐  ┌──────────────────┐
│ 3. backend-       │  │ 4. frontend-     │  ← 병렬 실행
│    developer      │  │    developer     │
│   → /output/src/be│  │  → /output/src/fe│
└──────────────────┘  └──────────────────┘
        ↓                      ↓
        └──────────┬───────────┘
                   ↓
        [code_review_enabled?]
                   ↓ (Yes)
┌─────────────────────────────────────────┐
│ 5. code-reviewer (옵션, 토글)            │
│    → review-report.md → BE/FE 수정 사이클│
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│ 6. qa-engineer                           │
│    → /output/tests/ + test-report.md     │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│ 7. devops-engineer                       │
│    → Dockerfile + docker-compose.yml     │
└─────────────────────────────────────────┘
                   ↓
[G3: 사용자 승인] "이대로 배포 패키지 묶을까요?"
                   ↓
       [최종 산출물 제시 + 실행 가이드]
```

---

### 2.3 고도화 흐름 (enhance) — NEW

```
[사용자 요청 + 기존 코드 경로 입력]
        ↓
┌─────────────────────────────────────────┐
│ 1. requirements-analyst (고도화 요구사항) │
│    → enhancement-requirements.md        │
└─────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────┐
│ 2. code-analyst (기존 코드 분석) ★NEW   │
│    → analysis-report.md                 │
│      (현황: 아키텍처·DB·문제점·위험요소) │
└─────────────────────────────────────────┘
        ↓
[G0: 사용자 확인] "현재 이런 상태예요, 이렇게 개선할까요?"
        ↓
┌─────────────────────────────────────────┐
│ 3. architect (고도화 계획 수립)           │
│    → migration-plan.md                  │
│    → api-spec.yaml (기존+신규 통합)      │
│    → schema.sql (마이그레이션 포함)      │
└─────────────────────────────────────────┘
        ↓
[G1: 사용자 승인] "이 계획으로 진행할까요?"
        ↓
┌──────────────────┐  ┌──────────────────┐
│ 4. backend-       │  │ 5. frontend-     │  ← 병렬 실행
│    developer      │  │    developer     │
│   (기존코드 수정) │  │  (기존코드 수정) │
└──────────────────┘  └──────────────────┘
        ↓                      ↓
        └──────────┬───────────┘
                   ↓
        [code_review_enabled?]
                   ↓ (Yes)
┌─────────────────────────────────────────┐
│ 6. code-reviewer (옵션)                  │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│ 7. qa-engineer                           │
│    → 회귀 테스트 + 신규 기능 테스트      │
└─────────────────────────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│ 8. devops-engineer                       │
│    → 기존 Docker 구성 개선 또는 신규 생성│
└─────────────────────────────────────────┘
                   ↓
[G2: 사용자 승인] "완료, 이대로 확정할까요?"
                   ↓
       [최종 산출물 제시 + 실행 가이드]
```

---

### 2.4 단계별 상세 정의

각 단계의 **LLM 판단 영역 / 코드 처리 영역 / 성공 기준 / 검증 방법 / 실패 처리**를 정의한다.

#### 단계 1: `requirements-analyst`
| 항목 | 내용 |
|------|------|
| **목적** | 비개발자의 자연어 요청을 구조화된 명세로 정제 |
| **모드** | A(인터뷰) 기본 / B(템플릿) / C(하이브리드) — `state.json`의 `requirements_mode`로 선택 |
| **Mode A 동작** | AI가 입력의 복잡도를 분석하여 질문 수·형식을 자율 결정. 단순 요청은 적게, 복잡한 요청은 더 많이 질문 |
| **Mode B 동작** | 도메인별 사전 제작 Markdown 양식 제시 → 사용자가 직접 빈칸 작성 (v0.2 정식 구현) |
| **Mode C 동작** | AI가 자유 입력을 분석한 후, 명확한 부분은 그대로 쓰고 애매한 부분만 선별 질문 (v0.2 정식 구현) |
| **LLM 판단** | 누락된 정보 식별, 추가 질문 생성, 사용자 답변 해석, 명세 작성 |
| **코드 처리** | 명세 템플릿 로딩, 필수 섹션 스키마 검증 |
| **성공 기준** | `requirements.md`에 필수 섹션 5개(목적, 사용자·페르소나, 핵심 기능, 비기능 요구, 제약) 모두 채워짐 (규칙 기반 체크) |
| **검증** | 필수 섹션 5개 존재 여부 규칙 검증 + 사람 검토(G1) |
| **실패 처리** | 누락 섹션 발견 시 자동 재인터뷰 (최대 2회) → 그래도 미달이면 에스컬레이션 |
| **고도화 모드 차이** | 인터뷰 초점이 "무엇을 추가/개선하고 싶은가"로 좁혀짐. 기존 앱 설명은 code-analyst가 별도 분석 |

#### 단계 1.5: `code-analyst` ★ 고도화 모드 전용
| 항목 | 내용 |
|------|------|
| **목적** | 기존 코드베이스를 분석하여 현황 리포트(`analysis-report.md`) 생성 |
| **트리거** | `state.json`의 `workflow_type: "enhance"`일 때만 실행 |
| **LLM 판단** | 아키텍처 패턴 식별, 문제점 진단, 신규 요구사항과의 충돌 탐지 |
| **코드 처리** | 파일 트리 생성, 의존성 분석, db.json/파일 기반 DB 패턴 감지, Docker 구성 파싱 |
| **성공 기준** | `analysis-report.md`에 현황 아키텍처·DB 현황·문제점 목록·개선 권고 포함 |
| **검증** | 규칙 기반 (필수 섹션 존재) + 사람 검토(G0) |
| **실패 처리** | 분석 불가 파일/경로 오류 시 사용자에게 경로 재확인 요청 |
| **원본 보호** | 기존 소스를 절대 수정하지 않음. 읽기 전용으로만 접근 |

#### 단계 2: `architect`
| 항목 | 내용 |
|------|------|
| **목적** | 요구사항 기반 기술 스택 선정 + 시스템 아키텍처·API 계약·DB 스키마 결정 |
| **LLM 판단** | 요구사항-스택 매핑, 아키텍처 패턴 선택, 도메인 모델링, API 설계 |
| **코드 처리** | OpenAPI 스펙 문법 검증, SQL 파싱 검증, 스택 매트릭스 룩업 |
| **성공 기준** | (1) 스택 결정 근거 명시, (2) OpenAPI 3.x valid, (3) SQL 파싱 성공, (4) 아키텍처 다이어그램 또는 텍스트 구조도 포함 |
| **검증** | 규칙 기반 (스펙 valid) + 사람 검토(G2) |
| **실패 처리** | 검증 실패 시 자동 재생성 (최대 2회) → 에스컬레이션 |
| **고도화 모드 차이** | 기존 스택을 최대한 유지. 변경이 필요한 부분만 선택적 교체. `migration-plan.md` 추가 산출 (DB 마이그레이션 단계 포함) |

#### 단계 3·4: `backend-developer` / `frontend-developer` (병렬)
| 항목 | 내용 |
|------|------|
| **목적** | 아키텍처·API·스키마를 입력으로 받아 실제 코드 생성 |
| **LLM 판단** | 코드 구현 방식, 라이브러리 선택(스택 내), 폴더 구조 결정 |
| **코드 처리** | 프로젝트 스캐폴딩 (`npm create`, `uv init` 등), 빌드, 타입체크, 린트 |
| **성공 기준** | (1) 빌드 성공, (2) 타입체크 통과, (3) 린트 에러 0, (4) API 계약 준수 |
| **검증** | 규칙 기반 (빌드/타입체크/린트 종료 코드 0) |
| **실패 처리** | 자동 재시도 (최대 3회, 매 시도마다 에러 로그 컨텍스트 주입) → 에스컬레이션 |
| **병렬화** | BE/FE는 `api-spec.yaml`을 공통 계약으로 사용하므로 독립 실행 가능 |
| **고도화 모드 차이** | 기존 코드를 `output/{run_id}/src/`에 복사 후 수정. **외과적 수정 원칙**: 기존 동작을 깨는 변경 금지. DB 마이그레이션 스크립트 `output/{run_id}/migrations/` 에 생성 |
| **완료 마커** | 완료 시 `/output/.../logs/be.done` 또는 `fe.done` 빈 파일 생성. 메인 에이전트가 두 파일 존재 여부를 폴링하여 다음 단계 진행 |
| **상태 슬롯** | `state.json`의 `be_status` / `fe_status` 키를 각자 갱신 (`running` → `done` / `failed`) |

#### 단계 5: `code-reviewer` (옵션 — 토글)
| 항목 | 내용 |
|------|------|
| **트리거** | `state.json`의 `code_review_enabled: true`일 때만 실행 |
| **목적** | BE/FE 코드의 정성적 품질 검토 (보안·컨벤션·가독성·로직 오류) |
| **LLM 판단** | 정성 평가, 개선 코멘트, 우선순위 분류 (Critical / Major / Minor) |
| **코드 처리** | 코드 diff 수집, 정적 분석기 실행 결과 수집 |
| **성공 기준** | Critical 이슈 0건 (Major/Minor는 리포트로만 기록) |
| **검증** | LLM 자기 검증 |
| **실패 처리** | Critical 발견 시 BE/FE 수정 사이클로 회귀 (최대 2회) → 에스컬레이션 |

#### 단계 6: `qa-engineer`
| 항목 | 내용 |
|------|------|
| **목적** | 단위·통합·smoke 테스트 작성 및 실행, 회귀 확인 |
| **LLM 판단** | 테스트 시나리오 도출(요구사항 기반), 케이스 우선순위 |
| **코드 처리** | 테스트 실행, 커버리지 측정, 결과 집계 |
| **smoke 테스트** | `api-spec.yaml` 기반으로 핵심 엔드포인트 smoke test 스크립트를 `/output/.../tests/smoke/` 에 생성. devops-engineer가 compose up 중 실행 |
| **API 글라이싱 감지** | BE 구현과 api-spec.yaml 간 불일치(누락 필드, 다른 응답 형식)는 통합 테스트 실행 시 자동 감지 |
| **성공 기준** | (1) 핵심 기능별 최소 1개 통합 테스트 존재, (2) 전체 테스트 통과, (3) 핵심 모듈 커버리지 ≥ 60% (조정 가능) |
| **검증** | 규칙 기반 (테스트 통과 + 커버리지 임계) |
| **실패 처리** | 실패 테스트 → BE/FE에 수정 요청 (최대 2회) → 에스컬레이션 |

#### 단계 7: `devops-engineer`
| 항목 | 내용 |
|------|------|
| **목적** | Dockerfile 및 docker-compose.yml 생성 + 실행 검증 |
| **LLM 판단** | 베이스 이미지 선택, 멀티스테이지 구성, 서비스 의존성 정의, 환경변수 구성 |
| **코드 처리** | `docker build`, `docker compose config` 검증, `docker compose up` 헬스체크 + smoke test 실행 |
| **성공 기준** | (1) `docker build` 성공, (2) `compose config` valid, (3) `compose up` 후 헬스체크 통과, (4) QA가 작성한 `/tests/smoke/` 스크립트 통과 |
| **최종 산출물 안내** | G3 승인 후 자연어 실행 가이드 + 출력 폴더 경로를 텍스트로 제시 (`cd /output/YYYY-MM-DD-HHMMSS && docker compose up -d` 형식) |
| **검증** | 규칙 기반 + 사람 검토(G3) |
| **실패 처리** | 자동 재시도 (최대 2회) → 에스컬레이션 |

### 2.3 사람 승인 게이트 (G1 / G2 / G3)

| 게이트 | 위치 | 사용자에게 묻는 내용 | 적용 모드 |
|--------|------|---------------------|---------|
| **G0** | code-analyst 직후 | "현재 앱 상태 확인: 이런 문제가 있어요, 이렇게 고칠까요?" | enhance 전용 |
| **G1** | requirements-analyst 직후 | "정제된 요구사항이 이게 맞나요?" | create / enhance 공통 |
| **G2** | architect 직후 | "이 스택·구조(또는 개선 계획)로 진행할까요?" | create / enhance 공통 |
| **G3** | devops-engineer 직후 | "이 배포 패키지로 확정할까요?" | create / enhance 공통 |

**게이트 UX 형식:**
- **기본 제시물**: 자연어 요약 카드 (기술 용어 최소화, 핵심 결정 사항 1~5문장)
- **상세 보기**: 원하면 열람 가능한 산출물 파일 경로 링크 (예: `requirements.md`, `architecture.md`)
- 사용자는 전체 스펙을 읽지 않아도 승인 가능

**G1 수정 요청 처리:**
- **소규모 수정** (단어 변경, 문장 수정): 메인 에이전트가 `requirements.md` 직접 수정 → G1 재확인
- **대규모 수정** (기능 추가, 범위 변경): `requirements-analyst` 재호출 (원래 입력 + 수정 요청을 컨텍스트로 전달)

**확장 옵션:** `state.json`의 `demo_gate_enabled: true`로 BE/FE 직후 데모 게이트 G2.5 추가 가능 (v0.2 예정).

### 2.4 실패 처리 정책 통합표

| 실패 유형 | 처리 |
|----------|------|
| **형식 오류** (스키마 누락, 문법 오류) | 자동 재시도 (단계별 명시된 횟수) |
| **빌드/테스트 실패** | 에러 로그를 컨텍스트로 주입하여 재시도 |
| **판단 불확실성** (예: 모호한 요구사항) | 에스컬레이션 → 사용자에게 자연어 질문 |
| **재시도 한도 초과 (빌드/DevOps)** | AI가 에러 로그를 분석하여 더 단순한 스택으로 다운그레이드를 자연어로 제안 → 사용자 승인 시 재시도 / 거부 시 `escalation.md` 기록 후 중단 |
| **재시도 한도 초과 (기타)** | `/output/.../escalation.md`에 사유·컨텍스트 기록 후 사용자 호출 |
| **선택적 단계 실패** (code-reviewer 등) | 스킵 + `/output/.../logs/`에 사유 로그 |

**에스컬레이션 메시지 형식 (사용자 대상):**
> "백엔드 빌드가 3번 시도에도 실패했어요. Next.js 대신 vanilla HTML+JS로, FastAPI 대신 Flask로 변경하면 더 안정적으로 생성할 수 있습니다. 변경할까요, 아니면 현재 스택으로 계속 시도할까요?"

---

## 3. 구현 스펙 (구조 개요)

> **주의:** 본 절은 구조·역할 정의 수준까지만 다룬다. 각 파일의 상세 내용은 구현 단계(Claude Code)에서 작성한다.

### 3.1 폴더 구조

```
/project-root
├── CLAUDE.md                              # 메인 에이전트(오케스트레이터) 지침
├── /.claude
│   ├── /skills
│   │   ├── /requirements-elicitor         # 요구사항 인터뷰·템플릿·하이브리드
│   │   │   ├── SKILL.md                   # 서브에이전트 프롬프트에 컨텍스트로 로드
│   │   │   ├── /scripts                   # 모드 분기, 스키마 검증
│   │   │   └── /references                # 명세 템플릿, 인터뷰 질문 풀
│   │   ├── /stack-selector                # 요구사항→스택 매핑
│   │   │   ├── SKILL.md
│   │   │   ├── /scripts                   # 스택 매트릭스 룩업
│   │   │   └── /references                # 스택 비교표, 선정 기준
│   │   ├── /api-spec-generator            # OpenAPI 생성·검증
│   │   ├── /schema-designer               # DB 스키마 생성·검증
│   │   ├── /scaffold-runner               # 프로젝트 스캐폴딩 실행
│   │   ├── /build-runner                  # 빌드·타입체크·린트 실행
│   │   ├── /test-runner                   # 테스트 실행·커버리지 측정
│   │   ├── /docker-packager               # Dockerfile + Compose 생성·검증
│   │   ├── /human-gate                    # 사용자 승인 요청·답변 처리
│   │   └── /state-manager                 # state.json 읽기/쓰기
│   └── /agents
│       ├── /requirements-analyst/AGENT.md
│       ├── /architect/AGENT.md
│       ├── /backend-developer/AGENT.md
│       ├── /frontend-developer/AGENT.md
│       ├── /code-reviewer/AGENT.md        # 토글 가능 (최초 실행 시 사용자에게 질문)
│       ├── /qa-engineer/AGENT.md
│       └── /devops-engineer/AGENT.md
├── /output                                 # 실행 결과 누적 (실행마다 타임스탬프 폴더 생성)
│   └── /YYYY-MM-DD-HHMMSS                 # 실행별 격리 폴더 (예: 2025-01-15-143022)
│       ├── state.json                     # 워크플로우 상태·설정 SSOT (run_id 포함)
│       ├── requirements.md
│       ├── architecture.md
│       ├── stack-decision.md
│       ├── api-spec.yaml
│       ├── schema.sql
│       ├── /src
│       │   ├── /backend
│       │   └── /frontend
│       ├── /tests
│       │   └── /smoke                     # QA가 작성한 핵심 API smoke test 스크립트
│       ├── review-report.md               # code-reviewer 활성 시
│       ├── test-report.md
│       ├── Dockerfile
│       ├── docker-compose.yml
│       ├── escalation.md                  # 에스컬레이션 발생 시
│       └── /logs                          # 스킵·실패·재시도 로그
│           ├── be.done                    # BE 완료 마커 (병렬 완료 감지용)
│           └── fe.done                    # FE 완료 마커 (병렬 완료 감지용)
├── /config
│   └── settings.json                      # 영구 설정 (자연어 명령으로 업데이트 가능)
└── /docs                                  # 참고 문서 (선택)
```

### 3.2 CLAUDE.md 핵심 섹션 목록

> CLAUDE.md는 메인 에이전트(오케스트레이터)의 지침이다. 다음 섹션만 정의하고 상세 내용은 구현 시 작성한다.

| # | 섹션명 | 역할 |
|---|--------|------|
| 1 | **시스템 개요** | 본 설계서 요약·링크 |
| 2 | **오케스트레이션 정책** | 서브에이전트 호출 순서, 병렬 규칙, 게이트 처리 |
| 3 | **state.json 계약** | 상태 파일의 스키마와 갱신 책임 |
| 4 | **모드·토글 처리** | `requirements_mode`, `code_review_enabled`, `demo_gate_enabled` 분기 로직 |
| 5 | **게이트 처리 (G1/G2/G3)** | 사용자 승인 요청·재진입 처리 흐름 |
| 6 | **실패·에스컬레이션 정책** | 재시도 횟수, 에스컬레이션 트리거, 로그 위치 |
| 7 | **서브에이전트 호출 규약** | 입력 파일 경로 / 출력 파일 경로 / 호출 컨벤션 |
| 8 | **스킬 참조 규약** | 메인 에이전트가 직접 호출 가능한 스킬 목록과 사용 시점 |

### 3.3 에이전트 구조: 멀티 서브에이전트 + 오케스트레이터

| 구성 | 결정 사항 |
|------|----------|
| **구조** | 1 오케스트레이터(CLAUDE.md) + 8 서브에이전트(/agents/*) — code-analyst 추가 |
| **호출 방향** | 메인 → 서브에이전트 (서브에이전트 간 직접 호출 금지) |
| **데이터 전달** | **파일 기반** (`/output/YYYY-MM-DD-HHMMSS/` 하위, 경로만 프롬프트로 전달) |
| **상태 공유** | `state.json` 단일 파일 (메인이 갱신, 서브는 읽기 우선; BE/FE는 각자의 `be_status`/`fe_status` 슬롯만 갱신) |
| **병렬 실행** | BE/FE 단계에서만 병렬, 나머지는 순차 |
| **병렬 완료 감지** | 메인이 `be.done` / `fe.done` 마커 파일 존재 여부를 폴링하여 양쪽 완료 확인 후 다음 단계 진행 |
| **스킬 호출 방식** | 서브에이전트 호출 시 해당 SKILL.md 내용을 프롬프트 컨텍스트에 포함 (슬래시 커맨드·Bash 직접 실행 아님) |

### 3.4 작업 단계별 처리 방식 (에이전트 판단 vs 스크립트)

| 단계 | 에이전트 판단 (LLM) | 스크립트 처리 |
|------|-------------------|--------------|
| 요구사항 정제 | 인터뷰 질문 생성, 명세 작성 | 스키마 검증, 템플릿 로딩 |
| 아키텍처 설계 | 스택·패턴 선정, 도메인 모델링 | OpenAPI·SQL 문법 검증 |
| 코드 생성 (BE/FE) | 구현 방식, 코드 작성 | 스캐폴딩, 빌드, 타입체크, 린트 |
| 코드 리뷰 | 정성 평가, 개선안 도출 | diff 수집, 정적 분석 실행 |
| QA | 테스트 시나리오 도출 | 테스트 실행, 커버리지 집계 |
| DevOps | 베이스 이미지·구성 결정 | docker build, compose config·헬스체크 |

### 3.5 스킬 목록 (/.claude/skills)

> **스킬 호출 방식:** 서브에이전트를 호출할 때 해당 SKILL.md 파일 내용을 프롬프트 컨텍스트에 포함시켜 에이전트가 스킬 지침을 따르도록 한다. 슬래시 커맨드나 Bash 직접 실행이 아님.

| 스킬 | 역할 | 트리거 조건 (언제 호출되는가) | 호출 주체 |
|------|------|---------------------------|----------|
| `code-reader` | 기존 코드 구조 파악·의존성 분석·DB 패턴 감지 | enhance 모드 code-analyst 단계 | code-analyst |
| `requirements-elicitor` | 요구사항 모드별 처리·스키마 검증 | 단계 1 진입 시 | requirements-analyst |
| `stack-selector` | 요구사항 기반 스택 추천 매트릭스 | 단계 2 초기 | architect |
| `api-spec-generator` | OpenAPI 3.x 생성·검증 | 단계 2 중반 | architect |
| `schema-designer` | DB 스키마 생성·문법 검증 | 단계 2 중반 | architect |
| `scaffold-runner` | 선정된 스택으로 프로젝트 초기화 | 단계 3·4 시작 시 | backend/frontend-developer |
| `build-runner` | 빌드·타입체크·린트 실행 | 단계 3·4 검증 시점 | backend/frontend-developer |
| `test-runner` | 테스트 실행·커버리지 측정 | 단계 6 | qa-engineer |
| `docker-packager` | Dockerfile/Compose 생성·검증·smoke test 실행 | 단계 7 | devops-engineer |
| `human-gate` | 사용자에게 승인 요청·답변 파싱 | G1·G2·G3 도달 시 | 메인 에이전트 |
| `state-manager` | state.json 읽기·갱신 | 모든 단계 전이 시점 | 메인 + 서브에이전트 |

### 3.6 서브에이전트 정의 (입출력 계약)

| 에이전트 | 입력 (파일) | 출력 (파일) | 참조 스킬 | 모드 |
|---------|-----------|-----------|----------|------|
| `code-analyst` | 기존 코드 경로, `enhancement-requirements.md` | `analysis-report.md` | code-reader, human-gate, state-manager | enhance 전용 |
| `requirements-analyst` | 사용자 자연어 메시지, `state.json` | `requirements.md` 또는 `enhancement-requirements.md` | requirements-elicitor, human-gate, state-manager | 공통 |
| `architect` | `requirements.md` | `architecture.md`, `stack-decision.md`, `api-spec.yaml`, `schema.sql` | stack-selector, api-spec-generator, schema-designer, state-manager |
| `backend-developer` | `api-spec.yaml`, `schema.sql`, `stack-decision.md` | `/output/src/backend/*` | scaffold-runner, build-runner, state-manager |
| `frontend-developer` | `api-spec.yaml`, `stack-decision.md`, `requirements.md` | `/output/src/frontend/*` | scaffold-runner, build-runner, state-manager |
| `code-reviewer` | `/output/src/**` | `review-report.md` | build-runner (정적 분석), state-manager |
| `qa-engineer` | `/output/src/**`, `requirements.md`, `api-spec.yaml` | `/output/tests/*`, `test-report.md` | test-runner, state-manager |
| `devops-engineer` | `stack-decision.md`, `/output/src/**` | `Dockerfile`, `docker-compose.yml` | docker-packager, state-manager |

### 3.7 주요 산출물 파일 형식

| 파일 | 형식 | 비고 |
|------|------|------|
| `state.json` | JSON | 워크플로우 상태·설정 SSOT |
| `requirements.md` | Markdown (정해진 섹션 스키마) | 비개발자도 읽을 수 있는 자연어 명세 |
| `architecture.md` | Markdown + 텍스트 구조도 | 컴포넌트·데이터 흐름 |
| `stack-decision.md` | Markdown | 선정된 스택과 근거 |
| `api-spec.yaml` | OpenAPI 3.x | BE/FE 공통 계약 |
| `schema.sql` | SQL DDL | DB 스키마 |
| `review-report.md` | Markdown | Critical/Major/Minor 분류 |
| `test-report.md` | Markdown | 통과/실패/커버리지 |
| `tests/smoke/` | 스크립트 (curl/pytest 등) | QA 작성, DevOps가 compose up 중 실행 |
| `Dockerfile` | Dockerfile | 멀티스테이지 권장 |
| `docker-compose.yml` | YAML | 서비스·네트워크·볼륨 |
| `escalation.md` | Markdown | 사용자 호출 시 사유·컨텍스트 |
| `logs/be.done`, `logs/fe.done` | 빈 마커 파일 | 병렬 완료 감지용 |

**state.json 스키마 예시:**
```json
{
  "run_id": "2025-01-15-143022",
  "workflow_type": "create",
  "source_path": null,
  "mode": "A",
  "current_step": "architect",
  "code_review_enabled": true,
  "demo_gate_enabled": false,
  "deploy_target": "compose",
  "gates_passed": ["G0", "G1"],
  "be_status": "running",
  "fe_status": "done",
  "retries": { "requirements": 0, "analyst": 0, "architect": 0, "code": 1, "devops": 0 }
}
```

> `workflow_type`: `"create"` (신규) 또는 `"enhance"` (고도화)
> `source_path`: 고도화 모드 시 기존 코드 경로 (예: `"/Users/user/my-app"`)
> `retries.analyst`: code-analyst 재시도 횟수 (enhance 모드 전용)

### 3.8 설정 토글 (settings.json)

| 키 | 기본값 | 설명 |
|----|-------|------|
| `workflow_type` | `"create"` | `"create"` (신규) / `"enhance"` (고도화) — 실행 시 자동 감지로 덮어씀 |
| `source_path` | `null` | 고도화 모드 시 기존 앱의 절대 경로 |
| `requirements_mode` | `"A"` | A=인터뷰 / B=템플릿 / C=하이브리드 |
| `code_review_enabled` | `true` | code-reviewer 단계 활성화 여부 |
| `demo_gate_enabled` | `false` | BE/FE 직후 데모 게이트 추가 (v0.2 옵션) |
| `deploy_target` | `"compose"` | v0.1은 compose만 / 추후 `k8s` / `helm` / `cloud:<name>` |
| `retry_limits` | `{requirements:2, architect:2, code:3, devops:2}` | 단계별 최대 재시도 |
| `coverage_threshold` | `60` | QA 커버리지 임계값(%) |

**설정 변경 방식:**
- **자연어 명령**: 사용자가 "코드 리뷰 꺼줘", "요구사항 모드 B로 바꿔줘" 등 자연어로 요청하면 메인 에이전트가 `settings.json`을 직접 업데이트 (영구 적용)
- **최초 실행 시 코드 리뷰 질문**: 워크플로우 시작 전 "코드 리뷰를 진행할까요? (토큰 비용이 추가됩니다)" 질문 → 답변에 따라 `settings.json`의 `code_review_enabled` 저장

### 3.9 진입점 및 오케스트레이터 트리거 가드

| 항목 | 내용 |
|------|------|
| **진입점** | 사용자가 Claude Code 세션에 자유로운 자연어를 입력하면 CLAUDE.md가 수신 |
| **트리거 조건** | 앱·시스템 생성 의도 감지 시에만 오케스트레이션 시작 (예: '만들어줘', '빌드해줘', '어플 만들기', '시스템 구축' 등) |
| **비앱 요청 처리** | 의도가 앱 생성이 아닌 경우 (날씨 질문, 다른 파일 수정 등) CLAUDE.md는 일반 Claude Code로 응답하고 워크플로우를 시작하지 않음 |
| **운영 모델** | 단일 운영자(주로 본인)가 실행하고 `/output/YYYY-MM-DD-HHMMSS/` 산출물을 팀에 공유 |
| **재실행 동작** | 같은 프로젝트 루트에서 재실행 시 이전 타임스탬프 폴더는 보존; 중단된 실행은 `state.json`의 `current_step`으로 이어달리기 가능 |

---

## 4. 확장 로드맵 (v0.2 이후)

| 버전 | 추가 사항 | 트리거 |
|------|----------|--------|
| **v0.2** ✅ | 고도화 모드(enhance) 추가: code-analyst 에이전트, code-reader 스킬, G0 게이트, DB 마이그레이션 플로우 | 2026-05-16 구현 완료 |
| **v0.3** | Mode B(도메인별 Markdown 양식) 정식 구현, Mode C(애매한 부분만 질문) 정식 구현, 데모 게이트(G2.5) 활성화 옵션 | 사용자 피드백 후 |
| **v0.3** | 배포 타겟 확장: K8s raw YAML 매니페스트 | 어플리케이션 운영 후 필요성 발생 시 |
| **v0.4** | Helm 차트 생성, 멀티 환경(dev/stg/prd) 파라미터화 | 운영 환경 다변화 시 |
| **v0.5** | 클라우드별 매니페스트 (NCP/AWS/Azure 등) | 특정 클라우드 표준화 결정 시 |
| **(기타)** | 기존 코드베이스 수정 모드, 추가 역할(Security Reviewer 등) | 별도 검토 |

---

## 5. 구현 시 우선순위 (제안)

> 본 설계서는 계획서이며, 실제 구현은 Claude Code에서 진행한다.

1. **1차**: `state-manager`, `human-gate` 스킬 + 메인 에이전트(CLAUDE.md) 골격
2. **2차**: `requirements-analyst` + `architect` (G1/G2까지 동작 가능)
3. **3차**: BE/FE 개발자 에이전트 + `scaffold-runner` / `build-runner`
4. **4차**: `qa-engineer` + `code-reviewer` (토글 검증)
5. **5차**: `devops-engineer` + `docker-packager` (G3까지 완성)

---

## 부록 A. 검증 패턴 통합표

| 단계 | 검증 유형 | 구체 방법 |
|------|----------|----------|
| requirements-analyst | 규칙 기반 + 사람 검토 | 필수 섹션 5개 존재 확인 + G1 |
| architect | 규칙 기반 + 사람 검토 | OpenAPI/SQL 문법 검증 + G2 |
| backend/frontend-developer | 규칙 기반 | 빌드 + 타입체크 + 린트 종료 코드; 완료 시 be.done / fe.done 마커 파일 생성 |
| code-reviewer | LLM 자기 검증 | Critical 분류 결과 (최초 실행 시 사용자가 ON/OFF 선택) |
| qa-engineer | 규칙 기반 | 테스트 통과 + 커버리지 임계 + smoke test 스크립트 생성 |
| devops-engineer | 규칙 기반 + 사람 검토 | docker build + compose config + 헬스체크 + smoke test 실행 + G3 |

## 부록 B. 데이터 전달 패턴

**원칙: 파일 기반 (`/output/YYYY-MM-DD-HHMMSS/` 하위), 프롬프트에는 경로만 전달**

| 출처 → 도착 | 전달 데이터 | 방식 |
|------------|-----------|------|
| 메인 → 서브에이전트 | 입력 파일 경로 + 작업 컨텍스트 + 사용할 스킬의 SKILL.md 내용 | 프롬프트 인라인 |
| 서브 → 서브 (간접) | 모든 산출물 | `/output/YYYY-MM-DD-HHMMSS/`에 파일로 누적, 다음 서브가 읽음 |
| 메인 ↔ 모든 단계 | 진행 상태·설정 | `state.json` 읽기/쓰기 (BE/FE는 각자 슬롯만 갱신) |
| BE/FE → 메인 (병렬 완료) | 완료 신호 | `logs/be.done` / `logs/fe.done` 파일 생성 → 메인이 폴링 |
| 에이전트 → 사용자 (게이트) | 자연어 요약 카드 + 산출물 경로 링크 + 액션 선택지 | `human-gate` 스킬 |
| 메인 → 사용자 (설정 변경) | 변경된 settings.json 확인 | 자연어 명령 처리 후 settings.json 업데이트 확인 메시지 |

---

**문서 끝.**
