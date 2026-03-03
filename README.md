# FreeLang v2 공식 언어 명세

**Version**: 2.1.0
**Status**: Stable (HTTP Server Support)
**Last Updated**: 2026-03-03
**Repository**: https://gogs.dclub.kr/kim/freelang-v2-language-spec.git

---

## 📖 개요

FreeLang v2는 **순수 구현, 기록 기반, 안전/유연성 보장, 모듈화, 데이터 중심, 확장 가능**을 핵심 원칙으로 하는 시스템 프로그래밍 언어입니다.

### 핵심 특징
- ✅ **순수 FreeLang 구현**: FFI 없이 메모리, 네트워크, 데이터베이스 관리 가능
- ✅ **강타입 시스템**: 기본타입(Int, Float, Str) + 복합타입(Struct, Enum)
- ✅ **모듈화 구조**: VM 기능별 독립 모듈 (memory, network, database)
- ✅ **안전/유연성**: Stack/Heap 관리 + Unsafe 모드 선택 가능
- ✅ **데이터 중심**: JSON, CSV, DB, Web Resource 동일하게 처리

---

## 📚 문서 구조

| 문서 | 설명 |
|------|------|
| **LANGUAGE_POLICY.md** | 언어 철학, 설계 원칙, HTTP/네트워크 정책 |
| **VM_ARCHITECTURE.md** | VM 내부 구조, 메모리/네트워크/DB/HTTP 모듈 |
| **TYPE_SYSTEM.md** | 자료형, 타입 안전성, 변환 규칙 |
| **STDLIB_MODULES.md** | 표준 라이브러리 모듈 목록 |
| **EXAMPLES/** | 샘플 코드 (메모리, 네트워크, DB, HTTP 서버) |

---

## 🚀 빠른 시작

### 설치
```bash
kpm install freelang
freelang --version  # v2.0.0
```

### 기본 프로그램
```freelang
// hello.fl
fn main(): void {
  print("Hello, FreeLang v2!")
}
```

실행:
```bash
freelang hello.fl
```

---

## 🏗️ 핵심 원칙

### 1️⃣ 기록이 증명이다
- 모든 동작, 아키텍처, 테스트는 **명시적으로 기록**
- Gogs 저장소 활용 → 설계/실험/배포 증명 가능

### 2️⃣ 순수 구현 우선
- VM 기능은 **FFI 없이 순수 FreeLang**으로 동작 가능
- 필요시 FFI/C 연동 → 최적화, OS 연계

### 3️⃣ 모듈화
- 메모리, 네트워크, DB, Web Resource 기능별 VM 모듈화
- 서로 의존 최소화 + 재사용성 보장

### 4️⃣ 안전 & 유연성
- Stack/Heap 포인터, Value 타입 관리 → 기본 타입 안전 보장
- Unsafe/Raw Pointer 선택적 제공 → 고성능/저수준 접근

### 5️⃣ 데이터 중심
- JSON, CSV, TXT, Web Resource, DB를 **동일하게 처리**
- Zero-copy, 버퍼 재활용, 메모리 효율 최적화

---

## 📋 언어 구조

### 기본 타입
```
Int        - 정수 (64비트)
Float      - 실수 (64비트)
Str        - 문자열 (UTF-8)
Bool       - 불린 (true/false)
Ptr        - 포인터 (주소)
Value      - 타입 enum (메타타입)
```

### 제어 흐름
```
if/else    - 조건문
while      - 반복문
for        - 반복문 (간단한 루프)
fn         - 함수 선언
return     - 반환
```

### 모듈 시스템
```
use "vm.memory"    - 메모리 모듈 임포트
use "vm.net"       - 네트워크 모듈 임포트
use "vm.db"        - 데이터베이스 모듈 임포트
```

---

## 🔧 사용 사례

### 1. 메모리 관리
```freelang
let vm = FreeLangVM(1024 * 1024, 256)  // 1MB 힙, 256 스택
vm.memory.pushStack(Value("int", 42))
```

### 2. 네트워크 I/O
```freelang
let sock = vm.net.createSocket("127.0.0.1", 3306)
vm.net.connectSocket(sock)
vm.net.sendSocket(sock, [72, 101, 108, 108, 111])  // "Hello"
```

### 3. 데이터베이스
```freelang
vm.db.createTable("users", ["id", "name", "age"])
vm.db.insertRow("users", [Value("int", 1), Value("str", "Alice")])
```

### 4. 통합 처리
```freelang
// 파일 읽기 → JSON 파싱 → DB 저장 → HTTP 응답
// 모두 VM 내에서 순수 FreeLang으로 처리
```

---

## 📊 성능 특성

| 항목 | 특성 |
|------|------|
| **메모리 오버헤드** | ~100KB (VM 초기화) |
| **바이너리 크기** | ~1-3MB (FreeLang 런타임) |
| **스택 크기** | 조정 가능 (기본 256 엔트리) |
| **힙 크기** | 조정 가능 (기본 1-10MB) |

---

## 🔮 확장 정책

### FFI/저수준 연동
성능 최적화 필요 시 C/OS 함수 연동 가능
FreeLang 코드는 **언제나 독립 실행 가능**

### 언어 확장
HTML/CSS/JS 처리, JSON/CSV 변환, 네트워크 API
→ 모두 **VM 모듈**로 확장 가능

### 통합 실시간 처리
파일 + DB + HTTP + Web Resource
→ 단일 VM 스크립트에서 동시 실행

---

## 📦 관련 패키지

| 패키지 | ID | 설명 |
|--------|----|----|
| **@freelang/http-server** | 12007 | Pure FreeLang HTTP 서버 (★ NEW) |
| **@freelang/vm-core** | 12005 | VM 핵심 (메모리/네트워크/DB) |
| **@freelang/mariadb-driver** | 12006 | MariaDB/MySQL 드라이버 |
| **@freelang/http** | 12001 | HTTP 모듈 (레거시) |
| **@freelang/network** | 12002 | 네트워크 모듈 |
| **@freelang/socket** | 12003 | 소켓 저수준 제어 |
| **@freelang/server** | 12004 | 웹 서버 프레임워크 (레거시) |

---

## 🤝 기여

FreeLang v2는 **순수 구현, 기록 기반** 정책을 따릅니다.

1. 기능 제안 → Gogs Issue
2. 구현 → Feature Branch
3. 테스트 → 전체 검증
4. 기록 → Commit + 명세 업데이트

---

## 📄 라이센스

**MIT License** - 자유로운 사용, 수정, 배포 가능

---

## 🔗 리소스

- **공식 저장소**: https://gogs.dclub.kr/kim/freelang-v2-language-spec.git
- **KPM 레지스트리**: `kpm search freelang`
- **VM 코어**: https://gogs.dclub.kr/kim/freelang-vm-core.git
- **커뮤니티**: https://guestbook.dclub.kr

---

**FreeLang v2 - 순수 구현, 기록 기반, 데이터 중심 언어** 🚀
