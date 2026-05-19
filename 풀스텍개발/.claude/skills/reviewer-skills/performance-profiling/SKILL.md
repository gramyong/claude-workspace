# performance-profiling

## 목적
코드 기반 정적 분석으로 성능 병목을 식별하고 "변경 전/후 벤치마크" 수치로 제안한다.

## 분석 항목

### 1. DB 쿼리 효율성
**N+1 쿼리 패턴 (Major):**
```javascript
// ❌ N+1: 루프 안에서 DB 호출
const users = await User.findAll();
for (const user of users) {
  user.posts = await Post.findAll({ where: { userId: user.id } }); // N번 쿼리
}

// ✅ JOIN 또는 include 사용
const users = await User.findAll({ include: Post });  // 1번 쿼리
```

**인덱스 미활용 패턴:**
- WHERE 절에 자주 사용하는 컬럼에 인덱스가 없는 경우
- `LIKE '%keyword%'` 패턴 (인덱스 미사용)

### 2. 응답시간 예상 기준
| 쿼리 유형 | 예상 시간 | 기준 |
|---------|---------|------|
| 인덱스 조회 (단건) | < 5ms | 정상 |
| 인덱스 조회 (목록) | < 50ms | 정상 |
| 풀 스캔 (소규모) | < 200ms | 경고 |
| N+1 쿼리 | ~N×10ms | Major |

### 3. 메모리 누수 패턴
```javascript
// ❌ 이벤트 리스너 해제 안 함
class MyService {
  constructor() {
    process.on('message', this.handler);  // 해제 없음
  }
}

// ✅ 명시적 해제
class MyService {
  constructor() {
    this.handler = this.handler.bind(this);
    process.on('message', this.handler);
  }
  destroy() {
    process.off('message', this.handler);
  }
}
```

### 4. 프론트엔드 번들 사이즈
```bash
# 번들 분석
npm run build -- --report  # Vite
npx webpack-bundle-analyzer dist/stats.json
```
- 단일 번들 > 500KB: 코드 스플리팅 권고
- 라이브러리 중복: lodash vs lodash-es 확인

## 성능 제안 작성 형식
```
변경 전: User.findAll() 후 루프에서 Post 조회 (N+1 쿼리)
         예상 응답시간: ~50 × N ms (사용자 50명 기준 ~2,500ms)

변경 후: User.findAll({ include: Post }) (단일 JOIN 쿼리)
         예상 응답시간: ~30ms
```
