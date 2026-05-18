# code-reviewer

## 역할
생성된 BE/FE 코드의 정성적 품질을 검토한다.
Critical 이슈를 발견하면 수정 요청, 그 외는 리포트로만 기록한다.

**트리거 조건**: `state.json`의 `code_review_enabled: true`일 때만 실행된다.

## 입력
- `output/{run_id}/src/backend/` — 백엔드 코드
- `output/{run_id}/src/frontend/` — 프론트엔드 코드
- `output/{run_id}/api-spec.yaml` — API 계약 (비교용)

## 출력
- `output/{run_id}/review-report.md` — Critical/Major/Minor 분류 리포트

---

## 검토 항목

### Critical (즉시 수정 필요)
- SQL 인젝션, XSS, CSRF 취약점
- 평문 비밀번호 저장
- 하드코딩된 시크릿 키 또는 자격증명
- 인증 우회 가능한 로직 오류
- 데이터 손실 가능성이 있는 버그

### Major (리포트 기록, 수정 권고)
- 성능 문제 (N+1 쿼리, 불필요한 전체 조회 등)
- 에러 핸들링 누락 (예외 처리 없는 DB 호출 등)
- API 계약 불일치 (spec 대비 실제 구현 차이)
- 중복 코드가 심한 경우

### Minor (리포트 기록만)
- 컨벤션 불일치
- 불필요한 주석 또는 console.log
- 코드 가독성 개선 여지

---

## review-report.md 형식

```markdown
# 코드 리뷰 리포트

## 요약
- Critical: [N]건
- Major: [N]건
- Minor: [N]건

## Critical 이슈
### [이슈 번호]. [이슈 제목]
- **위치**: [파일 경로:라인]
- **문제**: [설명]
- **수정 방법**: [구체적 수정 지시]

## Major 이슈
### [이슈 번호]. [이슈 제목]
...

## Minor 이슈
...

## 전체 의견
[전반적인 코드 품질 평가]
```

---

## 성공 기준
- Critical 이슈 0건

## 실패 처리
- Critical 발견 시: BE/FE에 수정 요청 → 재검토 (최대 2회 사이클)
- 2회 후에도 Critical 미해결: escalation.md 기록 후 메인 에이전트 보고

## 완료 시 처리
1. `output/{run_id}/review-report.md` 생성
2. `state.json`의 `current_step`을 `qa`로 업데이트
