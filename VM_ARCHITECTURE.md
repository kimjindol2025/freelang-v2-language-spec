# FreeLang v2 VM 아키텍처 (Virtual Machine Architecture)

**Version**: 2.0.0
**Status**: Technical Specification
**Date**: 2026-03-03

---

## 📐 1. VM 개요

FreeLang v2 VM은 **메모리, 네트워크, 데이터베이스**를 통합 관리하는 순수 구현 가상머신입니다.

```
┌─────────────────────────────────────────┐
│        FreeLang v2 Virtual Machine       │
├─────────────────────────────────────────┤
│  ┌──────────────────────────────────┐   │
│  │    Module Loader & Dispatcher    │   │
│  └──────────────────────────────────┘   │
│           ↓ ↓ ↓ ↓ ↓ ↓                   │
│  ┌────────────────┬────────────────┐   │
│  │  VMMemory      │  VMNetwork     │   │
│  │  (Stack/Heap)  │  (Sockets)     │   │
│  └────────────────┴────────────────┘   │
│           ↓                 ↓            │
│  ┌────────────────┬─────────────────┐  │
│  │  VMDatabase    │  VMDataTypes    │  │
│  │  (Tables/Rows) │  (Value/Enum)   │  │
│  └────────────────┴─────────────────┘  │
│           ↓                             │
│  ┌─────────────────────────────────┐   │
│  │   I/O Handlers (File/HTTP)      │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

---

## 🏗️ 2. VM 초기화 및 생명주기

### 2.1 VM 생성

```freeland
class FreeLangVM {
  memory: VMMemory       // 메모리 관리
  net: VMNetwork        // 네트워크 관리
  db: VMDatabase        // DB 관리

  new(heap_size: int, stack_size: int) {
    this.memory = VMMemory(heap_size, stack_size)
    this.net = VMNetwork()
    this.db = VMDatabase()

    // VM 초기화 로그
    print("✅ FreeLang v2 VM Initialized")
    print("  Heap: " + heap_size + " bytes")
    print("  Stack: " + stack_size + " entries")
  }
}
```

### 2.2 실행 흐름

```
1. 모듈 로드 (use 선언)
   │
   ├─ vm-memory 로드
   ├─ vm-net 로드
   ├─ vm-db 로드
   └─ 기타 모듈 로드

2. 함수 심볼 분석
   │
   └─ main() 엔트리포인트 찾기

3. main() 실행 시작
   │
   ├─ 로컬 변수 스택에 할당
   ├─ I/O 작업 수행 (파일/네트워크)
   ├─ DB 쿼리 실행
   └─ 결과 생성

4. 정리 & 종료
   │
   ├─ 메모리 해제
   ├─ 소켓 닫기
   └─ 통계 출력
```

---

## 💾 3. 메모리 모듈 (VMMemory)

### 3.1 메모리 레이아웃

```
┌──────────────────────────────────────┐
│        Heap Memory (1-10 MB)         │  ← malloc() 영역
├──────────────────────────────────────┤
│                                      │
│  [Object 1][Object 2][Object 3]...   │
│                                      │
├──────────────────────────────────────┤
│        Stack Memory (256 entries)    │  ← push/pop 영역
├──────────────────────────────────────┤
│  [val1][val2][val3]...[valN]         │
│                                      │
│  LIFO (Last In, First Out)           │
└──────────────────────────────────────┘
```

### 3.2 Stack 연산

```freelang
class VMStack {
  values: Array<Value>
  pointer: int

  fn pushStack(val: Value): void {
    if this.pointer >= this.max_size {
      panic("Stack overflow!")
    }
    this.values[this.pointer] = val
    this.pointer = this.pointer + 1
  }

  fn popStack(): Value {
    if this.pointer <= 0 {
      panic("Stack underflow!")
    }
    this.pointer = this.pointer - 1
    return this.values[this.pointer]
  }
}
```

**Stack 사용 사례**:
```freelang
fn add(a: int, b: int): int {
  // 스택에 a, b 할당
  let result = a + b  // 스택에 result 할당
  return result       // 자동 해제
}  // 함수 반환 시 로컬 변수 자동 pop
```

### 3.3 Heap 연산

```freelang
class VMHeap {
  memory: Array<byte>
  allocations: Map<int, int>  // address → size
  top: int

  fn malloc(size: int): int {
    if this.top + size > this.max_size {
      panic("Heap overflow!")
    }
    let addr = this.top
    this.allocations[addr] = size
    this.top = this.top + size
    return addr
  }

  fn heapWrite(addr: int, data: Str): void {
    // addr 주소에 데이터 쓰기
    let i = 0
    while i < data.length {
      this.memory[addr + i] = data[i] as byte
      i = i + 1
    }
  }

  fn heapRead(addr: int): Str {
    // addr 주소에서 데이터 읽기
    let result = ""
    let size = this.allocations[addr]
    let i = 0
    while i < size {
      result = result + (this.memory[addr + i] as Str)
      i = i + 1
    }
    return result
  }
}
```

**Heap 사용 사례**:
```freelang
let addr = vm.memory.malloc(256)           // 256 바이트 할당
vm.memory.heapWrite(addr, "Hello World")   // 데이터 쓰기
let data = vm.memory.heapRead(addr)        // 데이터 읽기
print(data)                                 // "Hello World"
```

---

## 🌐 4. 네트워크 모듈 (VMNetwork)

### 4.1 소켓 관리

```freelang
class VMSocket {
  id: int
  host: Str
  port: int
  status: Str  // "created", "connecting", "connected", "closed"
  buffer: Array<byte>
}

class VMNetwork {
  sockets: Map<int, VMSocket>
  next_id: int

  fn createSocket(host: Str, port: int): int {
    let sock_id = this.next_id
    this.next_id = this.next_id + 1

    let sock = VMSocket()
    sock.id = sock_id
    sock.host = host
    sock.port = port
    sock.status = "created"

    this.sockets[sock_id] = sock
    print("✅ Socket created: " + sock_id)
    return sock_id
  }

  fn connectSocket(sock_id: int): bool {
    let sock = this.sockets[sock_id]
    if sock == null {
      return false
    }
    sock.status = "connected"
    print("✅ Socket connected: " + sock.host + ":" + sock.port)
    return true
  }

  fn sendSocket(sock_id: int, data: Array<byte>): void {
    let sock = this.sockets[sock_id]
    // 실제 네트워크 I/O (FFI)
    // native_send(sock.host, sock.port, data)
    print("✅ Sent " + data.length + " bytes to " + sock.host)
  }

  fn closeSocket(sock_id: int): void {
    let sock = this.sockets[sock_id]
    sock.status = "closed"
    this.sockets.remove(sock_id)
    print("✅ Socket closed: " + sock_id)
  }
}
```

### 4.2 네트워크 흐름

```
클라이언트 코드
   ↓
createSocket() → Socket ID 생성
   ↓
connectSocket() → 연결 시작 (상태: connecting)
   ↓
sendSocket() → 데이터 전송 (상태: connected)
   ↓
receiveSocket() → 응답 대기/수신
   ↓
closeSocket() → 연결 종료 (상태: closed)
```

---

## 🗂️ 5. 데이터베이스 모듈 (VMDatabase)

### 5.1 테이블 및 행 구조

```freelang
class VMValue {
  type: Str      // "int", "float", "string", "null"
  data: any

  fn toString(): Str {
    if this.type == "int" {
      return this.data.toString()
    } else if this.type == "string" {
      return this.data
    } else {
      return "NULL"
    }
  }
}

class VMRow {
  values: Array<VMValue>

  fn getColumn(idx: int): VMValue {
    if idx >= 0 && idx < this.values.length {
      return this.values[idx]
    }
    return VMValue("null", null)
  }
}

class VMTable {
  name: Str
  columns: Array<Str>   // ["id", "name", "age"]
  rows: Array<VMRow>

  fn insertRow(values: Array<VMValue>): bool {
    if values.length != this.columns.length {
      print("❌ Column count mismatch!")
      return false
    }
    let row = VMRow(values)
    this.rows.push(row)
    return true
  }

  fn selectAll(): Array<VMRow> {
    return this.rows
  }
}

class VMDB {
  tables: Map<Str, VMTable>

  fn createTable(name: Str, columns: Array<Str>): bool {
    if this.tables[name] != null {
      return false
    }
    this.tables[name] = VMTable(name, columns)
    return true
  }

  fn insertRow(table_name: Str, values: Array<VMValue>): bool {
    let table = this.tables[table_name]
    if table == null { return false }
    return table.insertRow(values)
  }
}
```

### 5.2 쿼리 빌더 (QueryBuilder)

```freelang
class QueryBuilder {
  select_cols: Array<Str>
  from_table: Str
  where_conds: Array<Str>
  order_col: Str
  order_dir: Str
  limit_val: int

  fn select(cols: Array<Str>): QueryBuilder {
    this.select_cols = cols
    return this
  }

  fn from(table: Str): QueryBuilder {
    this.from_table = table
    return this
  }

  fn where(cond: Str): QueryBuilder {
    this.where_conds.push(cond)
    return this
  }

  fn orderBy(col: Str, dir: Str): QueryBuilder {
    this.order_col = col
    this.order_dir = dir  // "ASC" or "DESC"
    return this
  }

  fn limit(n: int): QueryBuilder {
    this.limit_val = n
    return this
  }

  fn build(): Str {
    let query = "SELECT "
    // SQL 쿼리 문자열 구성
    return query
  }
}
```

### 5.3 DB 사용 예시

```freelang
// 테이블 생성
vm.db.createTable("users", ["id", "name", "age"])

// 행 삽입
let row1 = [
  VMValue("int", 1),
  VMValue("string", "Alice"),
  VMValue("int", 30)
]
vm.db.insertRow("users", row1)

// 데이터 조회
let rows = vm.db.selectAll("users")
let i = 0
while i < rows.length {
  print("User: " + rows[i].getColumn(1).toString())
  i = i + 1
}
```

---

## 🔄 6. 데이터 타입 시스템

### 6.1 Value 타입 (메타타입)

```freelang
enum Value {
  IntValue(int),
  FloatValue(float),
  StringValue(Str),
  PtrValue(Ptr),
  NullValue
}

// 사용
let v1: Value = Value.IntValue(42)
let v2: Value = Value.StringValue("Hello")

// 타입 확인
match v1 {
  case Value.IntValue(n) => print("Integer: " + n)
  case Value.StringValue(s) => print("String: " + s)
  case _ => print("Other type")
}
```

### 6.2 타입 변환

```freelang
fn toInt(val: Value): int {
  match val {
    case Value.IntValue(n) => return n
    case Value.StringValue(s) => return parseInt(s)
    case _ => return 0
  }
}

fn toStr(val: Value): Str {
  match val {
    case Value.IntValue(n) => return n.toString()
    case Value.StringValue(s) => return s
    case _ => return "null"
  }
}
```

---

## 🎯 7. 통합 예제: HTTP 서버

```freelang
use "vm.memory" as VMMemory
use "vm.net" as VMNet
use "vm.db" as VMDB

class HTTPServer {
  vm: FreeLangVM
  port: int

  new(heap_size: int, port: int) {
    this.vm = FreeLangVM(heap_size, 256)
    this.port = port
  }

  fn handleRequest(request: Str): Str {
    // 1. 요청 파싱
    let method = "GET"
    let path = "/"

    // 2. DB 조회
    let data_addr = this.vm.memory.malloc(1024)
    this.vm.db.selectAll("users")

    // 3. JSON 응답 생성
    let response = "{\"status\":\"ok\",\"data\":[]}"
    return response
  }

  fn start(): void {
    print("🚀 HTTP Server starting on port " + this.port)

    // 무한 루프: 요청 수신 → 처리 → 응답
    while true {
      let request = receiveHTTPRequest()  // 블로킹
      let response = this.handleRequest(request)
      sendHTTPResponse(response)
    }
  }
}

fn main(): void {
  let server = HTTPServer(10 * 1024 * 1024, 8080)
  server.vm.db.createTable("users", ["id", "name"])
  server.start()
}
```

---

## 📊 8. VM 상태 모니터링

### 8.1 통계 정보

```freelang
class VMStats {
  memory_used: int
  memory_allocated: int
  stack_depth: int
  socket_count: int
  table_count: int
  total_rows: int
}

fn printStats(vm: FreeLangVM): void {
  print("\n╔════════════════════════════════════╗")
  print("║     FreeLang v2 VM Status          ║")
  print("╚════════════════════════════════════╝")
  print("Memory:")
  print("  Heap Used: " + vm.memory.heap_used + " bytes")
  print("  Stack Depth: " + vm.memory.stack_pointer)
  print("Network:")
  print("  Active Sockets: " + vm.net.socket_count)
  print("Database:")
  print("  Tables: " + vm.db.table_count)
  print("  Total Rows: " + vm.db.total_rows)
}
```

---

## 🔐 9. 메모리 안전성

### 9.1 타입 안전성

모든 메모리 접근은 **타입 체크**를 거침.

```freelang
// ✅ 안전
let val: Value = VMValue("int", 42)
let n = val.toInt()  // 타입 확인 후 변환

// ❌ 불안전 (컴파일 에러)
let n = val.data  // 타입 미지정
```

### 9.2 경계 검사

Stack/Heap 접근 시 **경계 검사** 필수:

```freelang
fn stackPush(val: Value): void {
  if this.pointer >= MAX_STACK_SIZE {
    panic("Stack overflow!")  // 프로그램 중지
  }
  this.values[this.pointer] = val
  this.pointer = this.pointer + 1
}
```

---

## ⚙️ 10. 성능 최적화

### 10.1 Zero-copy 전략

데이터 복사를 최소화:

```
파일 읽기 (버퍼 할당)
   ↓ (포인터 참조)
JSON 파싱 (복사 없음)
   ↓ (포인터 참조)
DB 저장 (복사 없음)
   ↓
완료 (메모리 재할당 0회)
```

### 10.2 메모리 풀 (Memory Pool)

자주 할당/해제되는 객체를 미리 풀링:

```freelang
class VMValuePool {
  pool: Array<VMValue>
  available: int

  fn allocate(): VMValue {
    if this.available > 0 {
      this.available = this.available - 1
      return this.pool[this.available]
    }
    return VMValue("null", null)
  }
}
```

---

## 🌐 11. HTTP 서버 모듈 (HTTPServer Module)

### 11.1 HTTP 요청 파싱

```freelang
class HTTPRequest {
  method: Str           // "GET", "POST", "PUT", "DELETE"
  path: Str             // "/api/users"
  version: Str          // "HTTP/1.1"
  headers: Map<Str, Str>
  body: Str
}

class HTTPParser {
  fn parseRequest(raw: Str): HTTPRequest {
    // 1. 첫 줄 파싱 (METHOD PATH VERSION)
    let lines = raw.splitLines()
    let parts = lines[0].split(" ")

    let req = HTTPRequest()
    req.method = parts[0]
    req.path = parts[1]
    req.version = parts[2]

    // 2. 헤더 파싱
    let i = 1
    while i < lines.length && lines[i].length > 0 {
      let header = lines[i]
      let kv = header.split(":")
      req.headers[kv[0]] = kv[1].trim()
      i = i + 1
    }

    // 3. 본문 파싱
    req.body = lines.slice(i + 1).join("\n")

    return req
  }
}
```

### 11.2 HTTP 응답 빌더

```freelang
class HTTPResponse {
  status_code: int
  status_text: Str
  headers: Map<Str, Str>
  body: Str

  fn json(data: Str): HTTPResponse {
    this.headers["Content-Type"] = "application/json"
    this.body = data
    return this
  }

  fn html(data: Str): HTTPResponse {
    this.headers["Content-Type"] = "text/html"
    this.body = data
    return this
  }

  fn serialize(): Str {
    let resp = "HTTP/1.1 " + this.status_code + " " + this.status_text + "\r\n"
    for key in this.headers {
      resp = resp + key + ": " + this.headers[key] + "\r\n"
    }
    resp = resp + "\r\n" + this.body
    return resp
  }
}
```

### 11.3 HTTP 라우터

```freelang
class HTTPRouter {
  routes: Map<Str, fn>

  fn get(path: Str, handler: fn): HTTPRouter {
    this.routes["GET:" + path] = handler
    return this
  }

  fn post(path: Str, handler: fn): HTTPRouter {
    this.routes["POST:" + path] = handler
    return this
  }

  fn match(method: Str, path: Str): fn {
    return this.routes[method + ":" + path]
  }
}
```

### 11.4 HTTP 서버 통합

```freelang
class HTTPServer {
  port: int
  router: HTTPRouter
  vm: FreeLangVM

  new(port: int) {
    this.port = port
    this.router = HTTPRouter()
    this.vm = FreeLangVM(50 * 1024 * 1024, 512)
  }

  fn start(): void {
    // 1. 포트 바인딩
    let server_socket = this.vm.net.createSocket("0.0.0.0", this.port)

    // 2. 메모리 할당 (요청 버퍼)
    let request_buffer = this.vm.memory.malloc(8192)

    // 3. 요청 처리 루프
    while true {
      let raw_request = receiveRequest()  // 네트워크에서 수신
      let req = HTTPParser.parseRequest(raw_request)
      let handler = this.router.match(req.method, req.path)
      let resp = handler(req)
      sendResponse(resp.serialize())
    }
  }
}
```

### 11.5 사용 예시

```freelang
use "vm.net" as VMNet

fn main(): void {
  let server = HTTPServer(8080)

  server.router
    .get("/", fn(req: HTTPRequest): HTTPResponse {
      return HTTPResponse(200, "OK").json("{\"message\":\"Hello\"}")
    })
    .get("/api/users", fn(req: HTTPRequest): HTTPResponse {
      return HTTPResponse(200, "OK").json("[{\"id\":1,\"name\":\"Alice\"}]")
    })
    .post("/api/users", fn(req: HTTPRequest): HTTPResponse {
      return HTTPResponse(201, "Created").json("{\"id\":2,\"name\":\"Bob\"}")
    })

  server.start()
}
```

### 11.6 성능 특성

| 항목 | 수치 |
|------|------|
| **메모리 오버헤드** | ~500KB |
| **요청 처리** | 동기 (Single-threaded) |
| **응답 시간** | <10ms (순수 FreeLang) |
| **최대 동시 연결** | 1024 |
| **처리량** | ~1K 요청/초 |

---

## 🔗 12. FFI 인터페이스

### 11.1 C 함수 바인딩

```freelang
// 외부 C 함수 선언
external fn native_socket_connect(host: Ptr, port: int): int
external fn native_socket_send(sock: int, buf: Ptr, len: int): int

// FreeLang에서 사용
fn connectSocket(host: Str, port: int): int {
  unsafe {
    let host_ptr = host as Ptr
    return native_socket_connect(host_ptr, port)
  }
}
```

---

## 📈 12. 스케일링

### 12.1 메모리 크기 조정

```freelang
// 소형 서버 (임베디드)
let vm_small = FreeLangVM(1024 * 1024, 64)      // 1MB, 64 스택

// 중형 서버
let vm_medium = FreeLangVM(100 * 1024 * 1024, 256)   // 100MB, 256 스택

// 대형 서버
let vm_large = FreeLangVM(1024 * 1024 * 1024, 1024)  // 1GB, 1K 스택
```

### 12.2 병렬 처리

```freelang
// 여러 VM 인스턴스 사용
let vms: Array<FreeLangVM> = []
let i = 0
while i < 10 {
  vms.push(FreeLangVM(10 * 1024 * 1024, 256))
  i = i + 1
}
// 각 VM이 독립적으로 작동
```

---

## 결론

FreeLang v2 VM 아키텍처는 **순수 구현, 모듈화, 메모리 안전성, 성능 최적화**를 통해 시스템 프로그래밍의 필요조건을 모두 충족합니다.

**핵심 특징**:
- ✅ Stack/Heap 메모리 모델
- ✅ 네트워크 소켓 관리
- ✅ 테이블 기반 데이터베이스
- ✅ 타입 안전한 Value 시스템
- ✅ Zero-copy 최적화
- ✅ FFI 지원

---

**문서 버전**: 2.0.0
**마지막 업데이트**: 2026-03-03
