# api-spec-generator 스킬

## 목적
요구사항과 아키텍처 결정을 바탕으로 OpenAPI 3.x 명세를 생성하고 검증한다.
BE/FE 공통 계약으로 사용되므로 정확하고 완전해야 한다.

---

## OpenAPI 3.x 기본 구조

```yaml
openapi: "3.0.3"
info:
  title: "{앱 이름} API"
  version: "1.0.0"
  description: "{앱 설명}"

servers:
  - url: "http://localhost:8000"
    description: "개발 서버"

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    # 공통 스키마 정의
    Error:
      type: object
      properties:
        message:
          type: string
        code:
          type: string

security:
  - bearerAuth: []

paths:
  /health:
    get:
      tags: [System]
      summary: 헬스체크
      security: []
      responses:
        "200":
          description: 서버 정상
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                    example: ok
```

---

## 엔드포인트 설계 원칙

### RESTful 규칙
- 리소스는 복수 명사: `/users`, `/articles`
- HTTP 메서드 의미 준수:
  - `GET`: 조회 (멱등)
  - `POST`: 생성
  - `PUT`: 전체 교체
  - `PATCH`: 부분 수정
  - `DELETE`: 삭제
- 중첩 리소스: `/users/{userId}/posts`

### 응답 코드
| 상황 | 코드 |
|------|------|
| 조회 성공 | 200 |
| 생성 성공 | 201 |
| 수정/삭제 성공 (응답 없음) | 204 |
| 잘못된 요청 | 400 |
| 인증 필요 | 401 |
| 권한 없음 | 403 |
| 리소스 없음 | 404 |
| 서버 오류 | 500 |

### 페이지네이션 (목록 조회)
```yaml
parameters:
  - name: page
    in: query
    schema: { type: integer, default: 1 }
  - name: size
    in: query
    schema: { type: integer, default: 20, maximum: 100 }
responses:
  "200":
    content:
      application/json:
        schema:
          type: object
          properties:
            items:
              type: array
              items: { $ref: '#/components/schemas/ResourceName' }
            total: { type: integer }
            page: { type: integer }
            size: { type: integer }
```

---

## 검증 방법

생성 후 아래 방법으로 문법 검증:

```bash
# Python으로 검증
pip install openapi-spec-validator
python -c "
from openapi_spec_validator import validate
import yaml
with open('api-spec.yaml') as f:
    spec = yaml.safe_load(f)
validate(spec)
print('Valid!')
"

# 또는 npx
npx @redocly/cli lint api-spec.yaml
```

검증 실패 시 에러 메시지를 분석하여 수정 후 재검증.
