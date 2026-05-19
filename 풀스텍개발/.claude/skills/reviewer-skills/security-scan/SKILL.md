# security-scan

## 목적
OWASP Top 10 기반으로 코드의 보안 취약점을 체계적으로 스캔한다.

## 스캔 체크리스트

### 1. 인증·인가
- [ ] JWT 시크릿이 환경변수로 분리되어 있는가 (`.env`에 있고 `.gitignore` 처리)
- [ ] 토큰 만료 시간이 설정되어 있는가 (권장: 24h 이하)
- [ ] 인증이 필요한 엔드포인트에 미들웨어가 적용되어 있는가
- [ ] 관리자 기능에 역할(role) 검사가 있는가

### 2. 입력 유효성 검사
- [ ] 모든 사용자 입력에 유효성 검사가 있는가
- [ ] SQL 쿼리에 파라미터 바인딩을 사용하는가 (raw query 금지)
- [ ] 파일 업로드 시 확장자·크기 제한이 있는가

### 3. XSS 방지
- [ ] 백엔드 응답이 Content-Type: application/json인가
- [ ] 프론트엔드에서 `dangerouslySetInnerHTML` 또는 `innerHTML` 직접 사용이 없는가
- [ ] 사용자 입력을 DOM에 삽입할 때 이스케이프 처리가 되는가

### 4. 민감 정보 보호
- [ ] 비밀번호가 bcrypt(cost ≥ 12)로 해싱되어 있는가
- [ ] 로그에 비밀번호·토큰이 출력되지 않는가
- [ ] `.env` 파일이 `.gitignore`에 포함되어 있는가
- [ ] 에러 응답에 스택 트레이스가 노출되지 않는가 (운영 환경)

### 5. CORS
- [ ] 허용 오리진이 `*` (와일드카드)가 아닌 특정 도메인으로 제한되어 있는가
- [ ] 운영 환경에서 `credentials: true` + 와일드카드 조합이 없는가

## 자동 스캔 명령어
```bash
# Node.js 의존성 취약점
npm audit --audit-level=moderate

# Python 의존성 취약점
pip-audit

# 하드코딩 시크릿 탐지
grep -rn "password\s*=\s*['\"]" src/ --include="*.js" --include="*.py"
grep -rn "secret\s*=\s*['\"]" src/
```

## 심각도 분류
| 발견 항목 | 심각도 |
|---------|--------|
| 평문 비밀번호 저장 | Critical |
| SQL Injection 가능 | Critical |
| 하드코딩 시크릿 | Critical |
| JWT 미검증 | Critical |
| CORS 와일드카드 (운영) | Major |
| 로그에 민감정보 | Major |
| 만료 없는 토큰 | Major |
