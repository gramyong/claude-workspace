# 풀스택 멀티 에이전트 빌더 설계서

> **자연어 요구사항을 입력하면, 3개의 전문 에이전트 팀이 협업하여 컨테이너화된 풀스택 어플리케이션을 신규 생성하거나, 기존 서비스를 고도화하는 시스템.**

| 항목 | 값 |
|------|---|
| 버전 | v0.3.0 (8에이전트 → 3에이전트 통합 — 2026-05-19) |
| 사용자 | 비개발자 PM(본인) + 타팀 비개발자 |
| 범위 | **신규 개발** (0→1) + **기존 서비스 고도화** (기능 추가·리팩토링·DB 마이그레이션) |
| 산출물 형태 | Docker Compose 기반 풀스택 어플리케이션 |
| 구현 플랫폼 | Claude Code (서브에이전트 + 스킬) |

---

## 1. 작업 컨텍스트

### 1.1 배경 및 목적
비개발자 사용자가 코드 작성 없이 풀스택 어플리케이션을 만들 수 있도록, **역할이 분리된 3개 에이전트 팀이 협업**하는 시스템을 구축한다.

**에이전트 통합 이유 (8→3):**
- 8에이전트 순차 실행: ~30,000+ 토큰 소비, 컨텍스트 전달 오버헤드 누적
- 3에이전트 구조: ~16,000 토큰 (architect ~3,000 + developer ~8,000 + reviewer ~5,000)
- 역할 경계를 "설계 / 구현 / 검증"으로 명확히 분리하여 책임 과부하 없음

**달성 목표 (3가지 동시):**
1. **토큰 효율** — 필요한 시점에만 도메인 지침 로드, 컨텍스트 폭발 방지
2. **결과물 속도** — 각 에이전트가 자신의 전체 역할을 완결해서 넘김
3. **품질 분리** — 설계자가 구현하지 않고, 구현자가 자기 코드를 리뷰하지 않음

### 1.2 범위

**포함 — 신규 개발 모드 (create):**
- 자연어 → 정제된 요구사항 명세
- 스택 선정 (요구사항 기반 자율 결정)
- 아키텍처·API·DB 스키마 설계
- 백엔드·프론트엔드 코드 생성 + 테스트 작성
- Docker 이미지 + Compose 매니페스트 생성
- 코드 리뷰 + 보안 취약점 스캔 + 성능 프로파일링

**포함 — 고도화 모드 (enhance):**
- 기존 코드베이스 분석 및 현황 리포트
- 기능 추가·리팩토링·코드 안정화
- DB 마이그레이션 (db.json/파일 기반 → SQLite/PostgreSQL 등)
- 기존 Docker 구성 개선
- 회귀 테스트 (기존 기능 보호)

**제외 (v0.3 한정):**
- K8s/Helm/클라우드별 매니페스트
- 실제 클라우드 배포 실행

### 1.3 입출력 정의

| 구분 | 신규 개발 (create) | 고도화 (enhance) |
|------|------------------|----------------|
| **입력** | 자연어 1차 요청 | 자연어 요청 + **기존 코드 경로** |
| **중간 산출물** | `design.md`, `api-spec.yaml`, `schema.sql` | `analysis-report.md`, `design.md`, `migration-plan.md` |
| **최종 출력** | 새 코드베이스 + Docker 패키지 | **수정된** 코드베이스 + `migrations/` + Docker 패키지 |

> **출력 격리 정책:** 매 실행마다 타임스탬프 폴더(`/output/YYYY-MM-DD-HHMMSS/`)를 생성하여 이전 결과를 보존한다. 중단된 실행은 해당 폴더의 `state.json` `current_step` 값으로 재개 가능하다.

### 1.4 제약 조건
- **사용자가 비개발자** → 게이트는 "자연어로 이해 가능한 시점"에 배치
- **토큰 비용 의식적 관리** → 에이전트별 컨텍스트 격리, 데이터 전달은 파일 기반
- **외과적 수정 원칙** (고도화) → 기존 동작을 깨는 변경 금지

### 1.5 용어 정의

| 용어 | 정의 |
|------|------|
| **메인 에이전트** | `CLAUDE.md`에 정의된 오케스트레이터. 전체 흐름 제어, 서브에이전트 호출 |
| **서브에이전트** | `/.claude/agents/*/AGENT.md`에 정의된 역할별 전문 에이전트 (3개) |
| **스킬** | `/.claude/skills/*/SKILL.md`에 정의된 결정론적 도구 |
| **게이트(G0~G3)** | 사용자 승인이 필요한 워크플로우 정지점 |
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
  [enhance] ─────────────→ 2.3 고도화 흐름
```

**신규 개발 감지 키워드**: 만들어줘, 빌드해줘, 개발해줘, 새로 만들기
**고도화 감지 키워드**: 고도화, 개선해줘, 리팩토링, 기능 추가, 기존 코드, 업그레이드

---

### 2.2 신규 개발 흐름 (create)

```
[사용자 1차 요청 (자연어)]
        ↓
┌─────────────────────────────────────────┐
│ 1. architect                             │
│    · 요구사항 수집 (인터뷰 or 템플릿)   │
│    · 기술 스택 선정                      │
│    · 아키텍처 다이어그램 (Mermaid)       │
│    · API 명세서 (OpenAPI)                │
│    · DB 스키마 (SQL DDL)                 │
│    → design.md / api-spec.yaml /         │
│      schema.sql / stack-decision.md      │
└─────────────────────────────────────────┘
        ↓
[G1: 사용자 승인] "이 설계로 진행할까요?"
        ↓
┌─────────────────────────────────────────┐
│ 2. developer                             │
│    · 백엔드 코드 (Node.js, Python 등)   │
│    · 프론트엔드 코드 (React, Vue 등)     │
│    · 에러 핸들링 & 로깅                  │
│    · 통합 테스트 작성                    │
│    · Dockerfile + docker-compose.yml     │
│    → /output/src/ + /output/tests/ +     │
│      Dockerfile + docker-compose.yml     │
└─────────────────────────────────────────┘
        ↓
[G2: 사용자 승인] "구현 완료, 검증 진행할까요?"
        ↓
┌─────────────────────────────────────────┐
│ 3. reviewer                              │
│    · 코드 리뷰 (보안, 성능, 컨벤션)     │
│    · 단위·통합 테스트 실행               │
│    · 성능 프로파일링 (응답시간, 메모리) │
│    · 보안 취약점 스캔                    │
│    · 개선 제안 피드백                    │
│    → review-report.md (승인 or 이슈)    │
└─────────────────────────────────────────┘
        ↓
[G3: 사용자 승인] "이대로 최종 확정할까요?"
        ↓
       [최종 산출물 제시 + 실행 가이드]
```

---

### 2.3 고도화 흐름 (enhance)

```
[사용자 요청 + 기존 코드 경로 입력]
        ↓
┌─────────────────────────────────────────┐
│ 1. architect                             │
│    · 고도화 요구사항 수집                │
│    · 기존 코드베이스 분석                │
│      (아키텍처·DB·문제점·위험요소)      │
│    · 개선 계획 수립 + 마이그레이션 설계  │
│    → analysis-report.md / design.md /   │
│      migration-plan.md / api-spec.yaml   │
└─────────────────────────────────────────┘
        ↓
[G0: 현황 확인] "현재 이런 상태, 이렇게 고칠까요?"
        ↓
[G1: 계획 승인] "이 개선 계획으로 진행할까요?"
        ↓
┌─────────────────────────────────────────┐
│ 2. developer                             │
│    · 기존 코드 복사 → 수정 (외과적)     │
│    · 신규 기능 추가                      │
│    · DB 마이그레이션 스크립트            │
│    · 회귀 테스트 보강                    │
│    · Docker 구성 개선                    │
│    → /output/src/ (수정) + migrations/  │
└─────────────────────────────────────────┘
        ↓
[G2: 구현 확인] "변경 내용 확인, 검증 진행할까요?"
        ↓
┌─────────────────────────────────────────┐
│ 3. reviewer                              │
│    · 회귀 테스트 실행 (기존 기능 보호)  │
│    · 신규 기능 테스트                    │
│    · 보안·성능 검토                      │
│    → review-report.md                   │
└─────────────────────────────────────────┘
        ↓
[G3: 사용자 승인] "최종 확정할까요?"
        ↓
       [최종 산출물 제시 + 실행 가이드]
```

---

### 2.4 단계별 상세 정의

#### Agent 1: `architect` — 설계 & 계획

| 항목 | 내용 |
|------|------|
| **목적** | 요구사항 → 기술 설계서 생성 |
| **요구사항 수집** | 사용자 자연어 분석 → 모호한 부분만 선별 질문 (Mode C 기본) |
| **LLM 판단** | 누락 정보 식별, 스택·패턴 선정, 도메인 모델링, API 설계, DB 설계 |
| **고도화 추가** | 기존 코드 분석(읽기 전용), 문제점 진단, 신규 요구사항과의 충돌 탐지 |
| **성공 기준** | (1) design.md에 아키텍처 다이어그램(Mermaid) 포함, (2) api-spec.yaml OpenAPI 3.x valid, (3) schema.sql 파싱 성공, (4) 기술 결정 사유 명시 |
| **실패 처리** | 검증 실패 시 자동 재생성 (최대 2회) → 에스컬레이션 |
| **출력** | `design.md`, `api-spec.yaml`, `schema.sql`, `stack-decision.md` (고도화 추가: `analysis-report.md`, `migration-plan.md`) |
| **스킬** | api-specification, database-schema-design, azure-architecture-pattern, (enhance) code-reader, requirements-elicitor, stack-selector, state-manager |

**CLAUDE.md 지침:**
- 마크다운 형식의 설계서만 출력
- 다이어그램은 Mermaid 문법 사용
- 기술 결정 사유를 항상 포함
- 보안/성능 제약사항 명시
- 기존 코드 분석 시 원본 절대 수정 금지 (읽기 전용)

---

#### Agent 2: `developer` — 구현

| 항목 | 내용 |
|------|------|
| **목적** | architect 설계서 → 동작하는 코드 |
| **입력** | `design.md`, `api-spec.yaml`, `schema.sql`, `stack-decision.md` |
| **LLM 판단** | 구현 방식, 라이브러리 선택(스택 내), 폴더 구조 결정 |
| **코드 처리** | 프로젝트 스캐폴딩, 빌드, 타입체크, 린트, Docker 빌드·compose config 검증 |
| **성공 기준** | (1) 빌드 성공, (2) 타입체크·린트 에러 0, (3) `docker build` 성공, (4) `compose config` valid, (5) API 계약 준수 |
| **실패 처리** | 에러 로그 주입 후 자동 재시도 (최대 3회) → 에스컬레이션 (스택 다운그레이드 제안) |
| **출력** | `/output/src/backend/*`, `/output/src/frontend/*`, `/output/tests/*`, `Dockerfile`, `docker-compose.yml` (고도화 추가: `migrations/*`) |
| **스킬** | nodejs-backend, react-frontend, unit-testing, error-handling, scaffold-runner, build-runner, docker-packager, state-manager |

**CLAUDE.md 지침:**
- architect의 설계서를 입력으로 받아 100% 구현
- 커밋 메시지는 Conventional Commits 형식
- 테스트 커버리지 최소 80% 목표
- 복잡한 로직에는 주석 필수
- 고도화 모드: 기존 코드를 `output/{run_id}/src/`에 복사 후 수정 (외과적 수정 원칙)

---

#### Agent 3: `reviewer` — 검증 & 최적화

| 항목 | 내용 |
|------|------|
| **목적** | developer 코드 → 프로덕션 준비 완료 |
| **입력** | `/output/src/**`, `api-spec.yaml`, `requirements.md` |
| **LLM 판단** | 정성 평가(Critical/Major/Minor 분류), 개선 우선순위 결정 |
| **코드 처리** | 테스트 실행, 커버리지 측정, 정적 분석기, 보안 스캔, 성능 프로파일링 |
| **성공 기준** | (1) Critical 이슈 0건, (2) 전체 테스트 통과, (3) 핵심 모듈 커버리지 ≥ 80%, (4) smoke test 통과 |
| **실패 처리** | Critical 발견 시 developer에 수정 요청 (최대 2회) → 에스컬레이션 |
| **출력** | `review-report.md` (승인된 코드 + 이슈 리스트) |
| **스킬** | security-scan, performance-profiling, code-style-lint, integration-test, test-runner, state-manager |

**CLAUDE.md 지침:**
- 찾은 이슈는 Critical / Major / Minor로 분류
- 첫 번째 우선순위 이슈 3개를 요약 카드로 제시
- 성능 개선은 "변경 전/후 벤치마크" 수치로 표시
- 승인 조건: 모든 Critical 이슈 해결 완료, 테스트 전체 통과

---

### 2.5 게이트 (G0 / G1 / G2 / G3)

| 게이트 | 위치 | 사용자에게 묻는 내용 | 적용 모드 |
|--------|------|---------------------|---------|
| **G0** | architect 직후 | "현재 앱 상태 확인: 이런 문제가 있어요, 이렇게 고칠까요?" | enhance 전용 |
| **G1** | architect 직후 | "이 설계(또는 개선 계획)로 진행할까요?" | create / enhance 공통 |
| **G2** | developer 직후 | "구현 완료, 검증 진행할까요?" | create / enhance 공통 |
| **G3** | reviewer 직후 | "최종 확정할까요?" | create / enhance 공통 |

**게이트 UX 형식:**
- **기본 제시물**: 자연어 요약 카드 (기술 용어 최소화, 핵심 결정 사항 1~5문장)
- **상세 보기**: 원하면 열람 가능한 산출물 파일 경로 링크
- 사용자는 전체 스펙을 읽지 않아도 승인 가능

**G1 수정 요청 처리:**
- **소규모** (단어 변경, 문장 수정): 메인 에이전트가 `design.md` 직접 수정 → G1 재확인
- **대규모** (기능 추가, 범위 변경): `architect` 재호출

---

### 2.6 실패 처리 정책

| 실패 유형 | 처리 |
|----------|------|
| **형식 오류** (스키마 누락, 문법 오류) | 자동 재시도 (단계별 명시된 횟수) |
| **빌드/테스트 실패** | 에러 로그를 컨텍스트로 주입하여 재시도 |
| **판단 불확실성** (모호한 요구사항) | 에스컬레이션 → 사용자에게 자연어 질문 |
| **재시도 한도 초과 (빌드)** | 더 단순한 스택으로 다운그레이드 제안 |
| **재시도 한도 초과 (기타)** | `/output/.../escalation.md`에 사유 기록 후 사용자 호출 |
| **reviewer Critical 발견** | developer에 수정 요청 회귀 (최대 2회) |

---

## 3. 구현 스펙

### 3.1 폴더 구조

```
/project-root
├── CLAUDE.md                              # 메인 에이전트(오케스트레이터) 지침
├── /.claude
│   ├── /skills
│   │   ├── /architect-skills
│   │   │   ├── /azure-architecture-pattern  # 클라우드 아키텍처 패턴
│   │   │   ├── /database-schema-design      # DB 스키마 설계
│   │   │   ├── /api-specification           # OpenAPI 명세 생성·검증
│   │   │   ├── /stack-selector              # 요구사항→스택 매핑
│   │   │   ├── /requirements-elicitor       # 요구사항 수집·검증
│   │   │   └── /code-reader                 # 기존 코드 분석 (enhance 전용)
│   │   ├── /developer-skills
│   │   │   ├── /nodejs-backend              # Node.js 백엔드 구현 패턴
│   │   │   ├── /react-frontend              # React 프론트엔드 구현 패턴
│   │   │   ├── /unit-testing                # 테스트 작성 가이드
│   │   │   ├── /error-handling              # 에러 처리·로깅 패턴
│   │   │   ├── /scaffold-runner             # 프로젝트 스캐폴딩 실행
│   │   │   ├── /build-runner                # 빌드·타입체크·린트 실행
│   │   │   └── /docker-packager             # Dockerfile/Compose 생성·검증
│   │   ├── /reviewer-skills
│   │   │   ├── /security-scan               # 보안 취약점 스캔
│   │   │   ├── /performance-profiling       # 성능 프로파일링
│   │   │   ├── /code-style-lint             # 코드 스타일 검토
│   │   │   ├── /integration-test            # 통합 테스트 실행
│   │   │   └── /test-runner                 # 테스트 실행·커버리지 측정
│   │   └── /common
│   │       ├── /human-gate                  # 사용자 승인 요청·답변 처리
│   │       └── /state-manager               # state.json 읽기/쓰기
│   └── /agents
│       ├── /architect/AGENT.md
│       ├── /developer/AGENT.md
│       └── /reviewer/AGENT.md
├── /output                                 # 실행 결과 누적
│   └── /YYYY-MM-DD-HHMMSS
│       ├── state.json                     # 워크플로우 상태·설정 SSOT
│       ├── design.md                      # 아키텍처 설계서 (Mermaid 포함)
│       ├── stack-decision.md
│       ├── api-spec.yaml
│       ├── schema.sql
│       ├── analysis-report.md             # enhance 전용
│       ├── migration-plan.md              # enhance 전용
│       ├── /src
│       │   ├── /backend
│       │   └── /frontend
│       ├── /tests
│       ├── /migrations                    # enhance 전용
│       ├── review-report.md
│       ├── Dockerfile
│       ├── docker-compose.yml
│       ├── escalation.md
│       └── /logs
├── /config
│   └── settings.json
└── /docs
```

### 3.2 CLAUDE.md 핵심 섹션 목록

| # | 섹션명 | 역할 |
|---|--------|------|
| 1 | **시스템 개요** | 3에이전트 구조 요약·설계서 링크 |
| 2 | **오케스트레이션 정책** | 에이전트 호출 순서, 게이트 처리 |
| 3 | **state.json 계약** | 스키마와 갱신 책임 |
| 4 | **모드 처리** | create / enhance 분기 로직 |
| 5 | **게이트 처리 (G0~G3)** | 승인 요청·재진입 처리 흐름 |
| 6 | **실패·에스컬레이션 정책** | 재시도 횟수, 에스컬레이션 트리거 |
| 7 | **에이전트 호출 규약** | 입출력 파일 경로, 스킬 로드 방식 |

### 3.3 에이전트 구조

| 구성 | 결정 사항 |
|------|----------|
| **구조** | 1 오케스트레이터(CLAUDE.md) + 3 서브에이전트(/agents/*) |
| **호출 방향** | 메인 → 서브에이전트 (서브에이전트 간 직접 호출 금지) |
| **데이터 전달** | **파일 기반** (`/output/YYYY-MM-DD-HHMMSS/` 하위, 경로만 프롬프트로 전달) |
| **상태 공유** | `state.json` 단일 파일 (메인이 갱신, 서브는 읽기 우선) |
| **실행 순서** | 순차 (architect → developer → reviewer) |
| **스킬 호출 방식** | 에이전트 호출 시 해당 SKILL.md 내용을 프롬프트 컨텍스트에 포함 (슬래시 커맨드·Bash 직접 실행 아님) |

### 3.4 서브에이전트 입출력 계약

| 에이전트 | 입력 (파일) | 출력 (파일) | 스킬 (도메인) | 스킬 (운영) |
|---------|-----------|-----------|------------|-----------|
| `architect` | 사용자 요청, (enhance) 기존코드경로 | `design.md`, `api-spec.yaml`, `schema.sql`, `stack-decision.md`, (enhance) `analysis-report.md`, `migration-plan.md` | api-specification, database-schema-design, azure-architecture-pattern | stack-selector, requirements-elicitor, (enhance) code-reader, state-manager |
| `developer` | `design.md`, `api-spec.yaml`, `schema.sql`, `stack-decision.md` | `/src/backend/*`, `/src/frontend/*`, `/tests/*`, `Dockerfile`, `docker-compose.yml`, (enhance) `migrations/*` | nodejs-backend, react-frontend, unit-testing, error-handling | scaffold-runner, build-runner, docker-packager, state-manager |
| `reviewer` | `/src/**`, `api-spec.yaml`, `design.md` | `review-report.md` | security-scan, performance-profiling, code-style-lint, integration-test | test-runner, state-manager |

### 3.5 주요 산출물 파일 형식

| 파일 | 형식 | 비고 |
|------|------|------|
| `state.json` | JSON | 워크플로우 상태·설정 SSOT |
| `design.md` | Markdown + Mermaid | 아키텍처 설계서, 비개발자도 읽을 수 있는 자연어 |
| `stack-decision.md` | Markdown | 선정된 스택과 근거 |
| `api-spec.yaml` | OpenAPI 3.x | BE/FE 공통 계약 |
| `schema.sql` | SQL DDL | DB 스키마 |
| `analysis-report.md` | Markdown | 기존 코드 현황 (enhance 전용) |
| `migration-plan.md` | Markdown | DB 마이그레이션 단계 (enhance 전용) |
| `review-report.md` | Markdown | Critical/Major/Minor 분류 + 승인 여부 |
| `Dockerfile` | Dockerfile | 멀티스테이지 권장 |
| `docker-compose.yml` | YAML | 서비스·네트워크·볼륨 |
| `escalation.md` | Markdown | 에스컬레이션 사유·컨텍스트 |

**state.json 스키마:**
```json
{
  "run_id": "2026-05-19-143022",
  "workflow_type": "create",
  "source_path": null,
  "current_step": "architect",
  "gates_passed": ["G1"],
  "retries": { "architect": 0, "developer": 0, "reviewer": 0 }
}
```

### 3.6 설정 토글 (settings.json)

| 키 | 기본값 | 설명 |
|----|-------|------|
| `requirements_mode` | `"C"` | C=하이브리드(기본) / A=인터뷰 / B=템플릿 |
| `deploy_target` | `"compose"` | v0.3은 compose만 |
| `retry_limits` | `{architect:2, developer:3, reviewer:2}` | 단계별 최대 재시도 |
| `coverage_threshold` | `80` | reviewer 커버리지 임계값(%) |

**설정 변경:** 자연어 명령 → 메인 에이전트가 `settings.json` 업데이트 (예: "요구사항 모드 A로 바꿔줘")

### 3.7 진입점 및 오케스트레이터 트리거

| 항목 | 내용 |
|------|------|
| **진입점** | 사용자가 Claude Code 세션에 자유로운 자연어를 입력하면 CLAUDE.md가 수신 |
| **트리거 조건** | 앱·시스템 생성/고도화 의도 감지 시에만 오케스트레이션 시작 |
| **비앱 요청 처리** | 의도가 앱 생성이 아닌 경우 일반 Claude Code로 응답 |
| **재실행 동작** | 이전 타임스탬프 폴더 보존; 중단된 실행은 `state.json`의 `current_step`으로 이어달리기 |

---

## 4. 토큰 효율 모델

### 3에이전트 vs 구(舊) 8에이전트 비교

```
[3에이전트 구조] — 총 ~16,000 토큰
──────────────────────────────────────
architect  → ~3,000  (설계서 1회 생성)
developer  → ~8,000  (설계서 문맥 유지 + 코드 생성)
reviewer   → ~5,000  (코드 리뷰 + 테스트)

[구 8에이전트 순차] — 총 ~30,000+ 토큰
──────────────────────────────────────
requirements-analyst → ~2,000
code-analyst         → ~3,000  (enhance 전용)
architect            → ~4,000
backend-developer    → ~5,000
frontend-developer   → ~5,000
code-reviewer        → ~4,000
qa-engineer          → ~4,000
devops-engineer      → ~3,000
(컨텍스트 전달 오버헤드 누적 포함)
```

**절감 이유:**
- 요구사항·분석·설계를 architect 1개 에이전트가 처리 → 중복 컨텍스트 제거
- BE/FE를 developer 1개 에이전트가 처리 → 공유 설계서를 1번만 로드
- 리뷰·테스트·보안을 reviewer 1개 에이전트가 처리 → 코드를 1번만 분석

---

## 5. 확장 로드맵

| 버전 | 추가 사항 | 트리거 |
|------|----------|--------|
| **v0.3** ✅ | 3에이전트 통합 (architect/developer/reviewer) | 2026-05-19 |
| **v0.4** | BE/FE 병렬 실행 옵션 (developer 내부에서 서브태스크 분기) | 대형 프로젝트 필요 시 |
| **v0.5** | Mode B(도메인별 Markdown 양식) 정식 구현 | 사용자 피드백 후 |
| **v0.6** | 배포 타겟 확장: K8s raw YAML 매니페스트 | 운영 환경 다변화 시 |
| **v0.7** | Helm 차트 생성, 멀티 환경(dev/stg/prd) 파라미터화 | 클라우드 표준화 결정 시 |

---

## 부록 A. 검증 패턴 통합표

| 단계 | 검증 유형 | 구체 방법 |
|------|----------|----------|
| architect | 규칙 기반 + 사람 검토 | OpenAPI/SQL 문법 검증 + Mermaid 존재 여부 + G0(enhance)/G1 |
| developer | 규칙 기반 | 빌드 + 타입체크 + 린트 + docker build + compose config 종료 코드 |
| reviewer | 규칙 기반 + LLM 판단 | 테스트 통과 + 커버리지 80% + Critical 0건 + G3 |

## 부록 B. 데이터 전달 패턴

**원칙: 파일 기반, 프롬프트에는 경로만 전달**

| 출처 → 도착 | 전달 데이터 | 방식 |
|------------|-----------|------|
| 메인 → 서브에이전트 | 입력 파일 경로 + 작업 컨텍스트 + 스킬 SKILL.md 내용 | 프롬프트 인라인 |
| architect → developer | design.md, api-spec.yaml, schema.sql, stack-decision.md | 파일 (경로 전달) |
| developer → reviewer | /src/**, /tests/**, Dockerfile | 파일 (경로 전달) |
| 메인 ↔ 모든 단계 | 진행 상태·설정 | state.json 읽기/쓰기 |
| 에이전트 → 사용자 (게이트) | 자연어 요약 카드 + 산출물 경로 링크 + 액션 선택지 | human-gate 스킬 |

---

**문서 끝.**
