# FreeLang v2 언어 정책 (Language Policy)

**Version**: 2.1.0
**Status**: Official Policy with HTTP Support
**Date**: 2026-03-03 (Updated: 2026-03-03)

---

## 📌 언어 정책이란?

**언어 정책** = 언어 설계/실행 원칙 + 문법/타입 안전성 + 메모리/FFI 처리 + 확장 정책

FreeLang v2의 모든 의사결정, 기능 구현, 성능 최적화는 이 정책을 따릅니다.

---

## 🎯 1. 언어 철학 & 설계 원칙

### 1.1 기록이 증명이다

모든 동작, 아키텍처, 테스트, 메모리/네트워크/DB 상태는 **명시적으로 기록**됨.

**목적**:
- 설계 결정의 근거 명확화
- 배포/실행의 증명 가능성
- 버그 원인 추적 용이

**구현**:
- Gogs 저장소에 모든 변경사항 커밋
- 테스트 결과 및 성능 지표 기록
- 설계 문서 지속적 업데이트

**예시**:
```
commit 32b7626 ✨ VM Database Extended: WHERE, ORDER BY, JOIN 구현
Author: Claude <claude@dclub.kr>
Date: 2026-03-03

VM Database 모듈에 WHERE, ORDER BY, JOIN 기능 추가
- QueryBuilder로 SELECT WHERE 조건 지원
- ORDER BY로 결과 정렬 가능
- JOIN으로 테이블 간 연관 쿼리 지원

테스트: 15/15 성공 (100%)
메모리: 1MB 힙에서 10K 행 처리 가능
```

---

### 1.2 순수 구현 우선

VM의 모든 기능은 **FFI(Foreign Function Interface) 없이 순수 FreeLang**으로 구현되어야 함.

**목적**:
- 언어의 완성도와 자립성 증명
- 크로스 플랫폼 호환성
- 보안 (외부 의존 최소화)

**구현 범위**:
| 기능 | 순수 FreeLang | FFI 허용 |
|------|--------------|---------|
| 메모리 관리 | ✅ Stack/Heap 할당 | ⚠️ OS malloc (최적화) |
| 네트워크 | ✅ 소켓 구조/상태관리 | ⚠️ BSD 소켓 호출 |
| 데이터베이스 | ✅ 테이블/행/인덱스 | ⚠️ MariaDB/SQLite 드라이버 |
| 파일 I/O | ✅ 버퍼 관리 | ⚠️ OS 파일 시스템 |

**원칙**:
- 핵심 로직은 **항상 순수 FreeLang**
- FFI는 **성능/OS 연계 최적화**에만 사용
- FFI 의존 코드는 **추상화층으로 격리**

**예시**:
```freelang
// 순수 FreeLang - 메모리 관리
class VMMemory {
  fn malloc(size: int): int {
    // 스택에서 메모리 구간 할당 로직
    // FFI 없이 포인터 계산
    return this.heap_top + size
  }
}

// FFI - OS 수준 최적화 (선택)
external fn os_mmap(addr: Ptr, size: int): Ptr
```

---

### 1.3 모듈화 (Modularity)

메모리, 네트워크, DB, Web Resource 등 **기능별 VM 모듈화**

**목적**:
- 코드 복잡도 관리
- 재사용성 극대화
- 의존성 최소화

**모듈 구조**:
```
vm-core/
├── vm-memory.fl      (메모리: Stack/Heap/GC)
├── vm-net.fl         (네트워크: Socket/TCP/UDP)
├── vm-db.fl          (DB: Table/Row/Query)
├── vm-json.fl        (JSON: Parse/Stringify)
├── vm-http.fl        (HTTP: Request/Response)
└── vm-resource.fl    (Web Resource: Cache/Fetch)
```

**의존성 규칙**:
- `vm-core` ← 다른 모듈 의존 불가
- `vm-memory` ← `vm-core`만 의존
- `vm-db` ← `vm-memory` 의존 가능
- `vm-http` ← `vm-net`, `vm-json` 의존 가능

**원칙**:
- 순환 의존성 금지
- 최상위 애플리케이션만 여러 모듈 통합

---

### 1.4 안전성 & 유연성 (Safety & Flexibility)

Stack/Heap 포인터, Value 타입 관리로 **기본 타입 안전성 보장**
필요시 Unsafe 모드로 **고성능/저수준 접근 가능**

**타입 안전성**:
```freelang
// ✅ 안전한 코드
let val: Value = Value("int", 42)
let int_val: int = val.toInt()  // 타입 체크

// ❌ 타입 불안전
let int_val: int = val.data  // 컴파일 에러
```

**Unsafe 모드** (필요시만):
```freelang
unsafe {
  let ptr: Ptr = ... as Ptr  // 강제 캐스팅
  let data = ptr.deref()     // 포인터 역참조
  // OS 수준 접근 가능하나 책임은 개발자
}
```

**원칙**:
- 기본은 Safe (타입 체크 강제)
- Unsafe는 명시적으로만 허용
- Unsafe 코드는 문서화 + 테스트 필수

---

### 1.5 데이터 중심 (Data-Centric)

JSON, CSV, TXT, Web Resource, DB 데이터를 **동일한 방식으로 VM에서 처리**

**목적**:
- 데이터 형식 간 일관된 처리
- Zero-copy 구현으로 성능 최적화
- 메모리 효율 극대화

**통일된 처리 방식**:
```
JSON (웹) → VM Value Tree
        ↓
CSV (파일) → VM Value Array
        ↓
DB (MariaDB) → VM Value Rows
        ↓
모두 동일 인터페이스로 변환/저장/질의
```

**메모리 효율**:
- **버퍼 재사용**: 파일 읽기 → JSON 파싱 → DB 저장 중 메모리 재할당 최소화
- **Zero-copy**: 데이터 복사 대신 포인터 참조
- **Lazy Evaluation**: 필요한 데이터만 메모리에 로드

**예시**:
```freelang
// 파일 → JSON → DB 통합 처리
let file_data = vm.file.read("data.json")  // 바이너리 로드
let json_obj = vm.json.parse(file_data)     // 인메모리 파싱
vm.db.insertRows("table", json_obj.rows)   // 동일 형식으로 저장
// 메모리 재할당 0회, 데이터 복사 0회
```

---

### 1.6 네트워크 & HTTP 모듈

**목적**: 웹 서버, REST API, 실시간 통신을 **순수 FreeLang**으로 구현 가능

**아키텍처**:
```
Application Layer
     ↓
HTTP Module (@freelang/http-server)
  - Request 파싱
  - Router (GET/POST/PUT/DELETE)
  - Response 빌더 (JSON/HTML/Text)
     ↓
Network Layer (@freelang/network)
  - TCP/UDP 소켓
  - 포트 바인딩
  - 패킷 송수신
     ↓
VM Memory
  - 버퍼 관리
  - Zero-copy 데이터 처리
```

**HTTP 지원 범위**:

| 기능 | 구현 | 상태 |
|------|------|------|
| HTTP/1.1 파싱 | ✅ 요청/응답 | v1.0.0 |
| REST 라우팅 | ✅ GET/POST/PUT/DELETE | v1.0.0 |
| 응답 포맷 | ✅ JSON/HTML/Text | v1.0.0 |
| 미들웨어 | 🔄 인증/CORS | v1.1.0 |
| WebSocket | 📅 실시간 통신 | v2.0.0 |
| HTTP/2 | 📅 멀티플렉싱 | v2.0.0 |

**사용 예시**:
```freelang
use "http-module" as HTTP

fn main(): void {
  let server = HTTP.Server(8080)

  server.route()
    .get("/api/users", fn(req) {
      return HTTP.ok("[{\"id\":1,\"name\":\"Alice\"}]")
    })
    .post("/api/users", fn(req) {
      return HTTP.created("{\"id\":2,\"name\":\"Bob\"}")
    })

  server.listen(nil)
}
```

**원칙**:
- ✅ 핵심 로직: 순수 FreeLang (HTTP 파싱, 라우팅)
- ⚠️ I/O: FFI 가능 (포트 바인딩, 소켓 연결)
- ✅ 응답: 완전히 순수 FreeLang

---

## 💻 2. 문법 & 실행 정책

### 2.1 자료형 체계

**기본 타입**:
```
Int     → 64비트 정수 (-9223372036854775808 ~ 9223372036854775807)
Float   → 64비트 실수 (IEEE 754)
Str     → UTF-8 문자열 (동적 길이)
Bool    → true / false
```

**복합 타입**:
```
Ptr     → 메모리 주소 (64비트 포인터)
Value   → 타입+데이터 쌍 (메타타입)
         enum { int(i64), float(f64), str(Str), null }
```

**사용자 정의 타입**:
```
struct  → 필드 모음 (고정 레이아웃)
enum    → 선택적 타입 (하나만 활성)
class   → 메서드 포함 (상태+행동)
```

**타입 안전성**:
- 컴파일 타임: 타입 검사 (불가피한 경우만 `as` 캐스팅)
- 런타임: Value 태그 확인 (enum 패턴 매칭)

---

### 2.2 제어 흐름

**조건문**:
```freelang
if condition {
  // true 경로
} else if other_condition {
  // 다른 경로
} else {
  // 기본 경로
}
```

**반복문**:
```freelang
// while 루프
while counter < 10 {
  counter = counter + 1
}

// for 루프 (간단한 반복)
for i in 0..10 {
  print(i)
}
```

**함수**:
```freelang
fn add(a: int, b: int): int {
  return a + b
}
```

**예외 처리**:
```freelang
fn risky(): void {
  if data == null {
    panic("Data is null!")  // 프로그램 중지
  }
  // 정상 처리
}
```

---

### 2.3 모듈 시스템

**모듈 임포트**:
```freelang
use "vm.memory" as VMMemory   // 메모리 모듈
use "vm.net" as VMNet         // 네트워크 모듈
use "vm.db" as VMDB           // DB 모듈
use "vm.json" as VMJSON       // JSON 모듈
```

**모듈 구조**:
```freelang
// vm-memory.fl
module VMMemory {
  class Stack { ... }
  class Heap { ... }
  fn malloc(size: int): Ptr { ... }
}
```

**사용**:
```freelang
use "vm.memory" as Mem

fn main(): void {
  let addr = Mem.malloc(1024)
  Mem.heapWrite(addr, "Hello")
}
```

---

### 2.4 실행 모델

**초기화 단계**:
1. VM 생성 (메모리 할당: 힙/스택)
2. 모듈 로드 (모든 `use` 선언 처리)
3. `main()` 함수 실행

**실행 루프**:
```
┌─────────────────────────┐
│ 1. 입력 수신 (파일/네트워크)  │
├─────────────────────────┤
│ 2. 데이터 파싱 (JSON/CSV) │
├─────────────────────────┤
│ 3. 로직 실행 (비즈니스)     │
├─────────────────────────┤
│ 4. 결과 저장 (DB/파일)    │
├─────────────────────────┤
│ 5. 응답 송신 (HTTP/stdout) │
└─────────────────────────┘
```

**동시성**:
- 단일 스레드 모델 (협력적 멀티태스킹 가능)
- 비동기 I/O로 블로킹 최소화
- 외부 I/O 대기 중 다른 작업 처리

---

## 🔧 3. 메모리 & FFI 정책

### 3.1 메모리 모델

**Stack (스택)**:
- 함수 호출 매개변수, 로컬 변수
- LIFO (Last In, First Out)
- 자동 해제 (함수 반환 시)

**Heap (힙)**:
- 동적 할당된 데이터 (배열, 구조체 인스턴스)
- 수동 할당/해제 또는 GC
- 프로그래머가 생명주기 관리

**가비지 컬렉션** (선택):
- 기본: 수동 해제 (malloc/free 스타일)
- 선택: GC 활성화로 자동 해제

---

### 3.2 FFI (Foreign Function Interface) 정책

**FFI 허용 범위**:
| 목적 | 허용 | 예시 |
|------|-----|------|
| 성능 최적화 | ✅ | C 수학 라이브러리 호출 |
| OS 연계 | ✅ | POSIX 파일 I/O |
| 기존 라이브러리 | ✅ | SQLite, OpenSSL |
| 핵심 로직 구현 | ❌ | VM 메모리 관리를 C로 짜기 |
| 기능 회피 | ❌ | 복잡한 파싱을 Python으로 넘기기 |

**FFI 선언**:
```freelang
// C 함수 외부 선언
external fn sqrt(x: float): float

// C 구조체 바인딩
external struct {
  int error_code;
  char *message;
} CError

// 사용
let result = sqrt(16.0)  // 4.0
```

**FFI 안전성**:
- 모든 FFI 호출은 문서화 필수
- FFI 코드는 Unsafe 블록 내에만 사용
- 타입 변환 책임은 개발자

---

## 🚀 4. 확장 정책 (Extension Policy)

### 4.1 기능 확장

**신규 기능 추가 절차**:
```
1. 설계 단계
   - RFC (Request for Comments) 작성
   - 기존 정책과 호환성 검토

2. 구현 단계
   - 기능 브랜치에서 개발
   - 단위 테스트 작성
   - 성능 측정

3. 통합 단계
   - 코드 리뷰
   - 통합 테스트
   - 명세 문서 업데이트

4. 배포 단계
   - 마이너/메이저 버전 증가
   - Changelog 기록
   - Gogs에 커밋
   - KPM 레지스트리 업데이트
```

---

### 4.2 언어 문법 확장

**금지된 확장**:
- ❌ 문법 재설계 (하위호환성 파괴)
- ❌ 타입 시스템 근본 변경
- ❌ 메모리 모델 변경

**허용된 확장**:
- ✅ 신규 키워드 (기존 문법과 충돌 없음)
- ✅ 신규 빌트인 함수 (기존 함수는 변경 금지)
- ✅ 신규 모듈 (기존 모듈 호환성 유지)

**예시** (허용):
```freelang
// v2.1: 신규 키워드 match (패턴 매칭)
match value {
  case Value("int", x) => print(x)
  case Value("str", s) => print(s)
}
```

---

### 4.3 라이브러리 확장

**표준 라이브러리 (stdlib)**:
- `vm.memory` - 메모리 관리
- `vm.net` - 네트워크 I/O
- `vm.db` - 데이터베이스
- `vm.json` - JSON 처리
- `vm.http` - HTTP 프로토콜
- `vm.file` - 파일 I/O

**서드파티 라이브러리**:
- KPM 레지스트리를 통해 배포
- 명명: `@author/package` 형식
- 버전 관리: Semantic Versioning

---

### 4.4 플랫폼 확장

**지원 플랫폼**:
- Linux (주 플랫폼)
- macOS (클라이언트)
- Windows (WSL2 기반)
- 라즈베리파이, 아두이노 (임베디드)

**플랫폼별 FFI**:
```freelang
#[cfg(os = "linux")]
external fn epoll_create(): int

#[cfg(os = "windows")]
external fn CreateIoCompletionPort(): int
```

---

## 📊 5. 버전 관리 & 호환성

### 5.1 Semantic Versioning

```
v2.MINOR.PATCH

v2.0.0    →  v2.1.0    →  v2.1.1
메이저    신규기능     버그픽스
변경없음  호환유지     호환유지
```

### 5.2 하위호환성 정책

**Major (v2 → v3)**: 하위호환성 보장 안 함
**Minor (v2.0 → v2.1)**: 완전 하위호환성 보장
**Patch (v2.0.0 → v2.0.1)**: 버그픽스만

---

## 🎯 6. 성능 & 최적화 정책

### 6.1 성능 목표

| 항목 | 목표 |
|------|------|
| 시작 시간 | < 100ms |
| 힙 크기 | 1-10MB (조정 가능) |
| 처리량 | > 100K 연산/초 |
| 메모리 효율 | Native C의 80% 이상 |

### 6.2 최적화 전략

1. **Zero-copy**: 데이터 복사 최소화
2. **Lazy Loading**: 필요할 때만 로드
3. **Inlining**: 작은 함수는 인라인 처리
4. **Cache Locality**: 메모리 접근 패턴 최적화

---

## 🔍 7. 정책 준수 검증

**정책 준수 확인 방법**:

```bash
# 1. 코드 스타일 검증
freelang --lint code.fl

# 2. 타입 안전성 검증
freelang --typecheck code.fl

# 3. 메모리 안전성 검증
freelang --memcheck code.fl

# 4. 성능 프로파일링
freelang --profile code.fl
```

---

## 📝 결론

FreeLang v2 언어 정책은 **순수 구현, 기록 기반, 안전/유연성 보장, 모듈화, 데이터 중심**을 핵심으로 하며, 모든 의사결정과 기능 구현은 이 정책을 따릅니다.

**핵심 원칙 요약**:
1. ✅ **기록이 증명** - 모든 것이 명시적으로 기록됨
2. ✅ **순수 구현** - FFI 없이 독립 실행 가능
3. ✅ **모듈화** - 의존성 최소화, 재사용성 극대화
4. ✅ **안전성** - 타입 안전, 메모리 안전
5. ✅ **데이터 중심** - 모든 데이터 형식을 동일하게 처리

---

**문서 버전**: 2.0.0
**마지막 업데이트**: 2026-03-03
**담당자**: FreeLang 커뮤니티
