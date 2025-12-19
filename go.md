# Ultimate Go Crash Course for FAANG Interviews (2024-2025)

---

## 1. Core Concepts You Must Know Cold

### 1.1 Variables, Types & Zero Values

Go is statically typed with type inference. Every type has a **zero value**.

```go
// Variable declarations
var name string           // zero value: ""
var age int               // zero value: 0
var isActive bool         // zero value: false
var ptr *int              // zero value: nil

// Short declaration (inside functions only)
count := 10               // type inferred as int
message := "hello"        // type inferred as string

// Multiple declarations
var x, y, z int = 1, 2, 3
a, b := "foo", 42

// Constants
const Pi = 3.14159
const (
    StatusOK    = 200
    StatusError = 500
)

// iota - auto-incrementing constant generator
const (
    Sunday = iota    // 0
    Monday           // 1
    Tuesday          // 2
)
```

**Interview Trap**: Uninitialized variables get zero values, not garbage. This is guaranteed.

---

### 1.2 Slices vs Arrays (CRITICAL - Asked 80%+ of interviews)

```go
// ARRAY: Fixed size, value type (copied on assignment)
arr := [5]int{1, 2, 3, 4, 5}      // length is part of type
arr2 := [...]int{1, 2, 3}         // compiler counts elements

// SLICE: Dynamic, reference type (shares underlying array)
slice := []int{1, 2, 3, 4, 5}     // no size specified
slice2 := make([]int, 5)          // len=5, cap=5, all zeros
slice3 := make([]int, 0, 10)      // len=0, cap=10

// Slice internals - THREE components:
// 1. Pointer to underlying array
// 2. Length (len)
// 3. Capacity (cap)

s := []int{1, 2, 3, 4, 5}
fmt.Println(len(s), cap(s))  // 5, 5

// Slicing creates a NEW slice header, SHARES underlying array
sub := s[1:3]   // [2, 3], len=2, cap=4 (from index 1 to end of original)
sub[0] = 999    // MODIFIES original! s is now [1, 999, 3, 4, 5]

// Append - may or may not allocate new array
s = append(s, 6)           // single element
s = append(s, 7, 8, 9)     // multiple elements
s = append(s, other...)    // spread another slice

// CRITICAL: Append returns new slice - always reassign!
// Wrong: append(s, x)
// Right: s = append(s, x)

// Copy - copies elements, returns count copied
dst := make([]int, len(src))
n := copy(dst, src)  // n = min(len(dst), len(src))
```

**Slice Capacity Growth**: When capacity exceeded, Go allocates new array (usually 2x for small slices, ~1.25x for large).

```go
// Full slice expression - controls capacity
s := []int{1, 2, 3, 4, 5}
sub := s[1:3:4]  // [2, 3], len=2, cap=3 (cap = 4-1)
// Prevents append on sub from overwriting s[4]
```

---

### 1.3 Maps

```go
// Declaration & initialization
var m map[string]int           // nil map - can read but NOT write!
m = make(map[string]int)       // initialized, ready to use
m2 := map[string]int{"a": 1}   // literal initialization

// Operations
m["key"] = 42                  // set
val := m["key"]                // get (returns zero value if missing)
val, exists := m["key"]        // comma-ok idiom - ALWAYS USE THIS
delete(m, "key")               // delete (no-op if key missing)

// Iteration - ORDER IS RANDOM (by design)
for key, value := range m {
    fmt.Println(key, value)
}

// Check existence pattern
if val, ok := m["key"]; ok {
    // key exists, use val
} else {
    // key doesn't exist
}

// Maps are NOT safe for concurrent use
// Use sync.Map or mutex for concurrent access
```

**Interview Trap**: Nil map reads return zero value, but writes panic!

---

### 1.4 Structs & Methods

```go
// Struct definition
type Person struct {
    Name    string
    Age     int
    Email   string `json:"email"` // struct tag
}

// Creating structs
p1 := Person{Name: "Alice", Age: 30}
p2 := Person{"Bob", 25, "bob@example.com"}  // positional (fragile)
p3 := new(Person)      // returns *Person, all zero values
p4 := &Person{}        // same as new(Person)

// Anonymous structs (great for tests, one-off data)
point := struct{ X, Y int }{10, 20}

// Methods - receiver goes before function name
// VALUE receiver - operates on copy
func (p Person) Greet() string {
    return "Hello, " + p.Name
}

// POINTER receiver - can modify original
func (p *Person) Birthday() {
    p.Age++
}

// Go auto-dereferences: p.Birthday() works even if p is value
// Go auto-references: (&p).Greet() works but unnecessary

// Embedding (composition over inheritance)
type Employee struct {
    Person           // embedded - fields/methods promoted
    EmployeeID int
}

e := Employee{Person: Person{Name: "Alice"}, EmployeeID: 123}
fmt.Println(e.Name)        // accessed directly (promoted)
fmt.Println(e.Person.Name) // also works
```

**When to use pointer receiver**:
1. Method modifies receiver
2. Receiver is large struct (avoid copying)
3. Consistency - if one method has pointer receiver, all should

---

### 1.5 Interfaces (THE MOST ASKED CONCEPT)

```go
// Interface definition - set of method signatures
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Interface embedding
type ReadWriter interface {
    Reader
    Writer
}

// IMPLICIT implementation - no "implements" keyword
type MyReader struct{}

func (r MyReader) Read(p []byte) (int, error) {
    // implement...
    return 0, nil
}
// MyReader now implements Reader automatically!

// Empty interface - accepts ANY type
var any interface{}  // or 'any' in Go 1.18+
any = 42
any = "hello"
any = struct{}{}

// Type assertion
val, ok := any.(string)  // safe - check ok
if ok {
    fmt.Println(val)
}

str := any.(string)  // unsafe - panics if wrong type

// Type switch
switch v := any.(type) {
case int:
    fmt.Println("int:", v)
case string:
    fmt.Println("string:", v)
default:
    fmt.Println("unknown type")
}

// Interface values have two components: (type, value)
// Interface is nil only if BOTH are nil
var r Reader           // nil interface
var p *MyReader        // nil pointer
r = p                  // r is NOT nil! (type=*MyReader, value=nil)
```

**Key interfaces to know**:
- `io.Reader`, `io.Writer` - I/O operations
- `error` - just `Error() string`
- `fmt.Stringer` - `String() string`
- `sort.Interface` - `Len()`, `Less(i,j)`, `Swap(i,j)`

---

### 1.6 Goroutines (Lightweight Threads)

```go
// Launch goroutine with 'go' keyword
go func() {
    fmt.Println("I'm running concurrently!")
}()

// Goroutines are multiplexed onto OS threads by Go runtime
// Cost: ~2KB stack (grows as needed) vs ~1MB for OS thread

// CRITICAL: Main goroutine exit kills all others
func main() {
    go doWork()  // might not complete!
    // main exits immediately
}

// Wait for goroutines with WaitGroup
var wg sync.WaitGroup

for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(id int) {  // MUST pass i as parameter!
        defer wg.Done()
        fmt.Println("Worker", id)
    }(i)
}
wg.Wait()  // blocks until all Done() called

// CLASSIC BUG - loop variable capture
for i := 0; i < 5; i++ {
    go func() {
        fmt.Println(i)  // BUG: all print 5 (or random)
    }()
}
// Fix: pass as parameter OR use Go 1.22+ (fixed by default)
```

---

### 1.7 Channels (INTERVIEW FAVORITE)

```go
// Channels are typed conduits for communication
ch := make(chan int)       // unbuffered - send blocks until receive
ch := make(chan int, 10)   // buffered - send blocks when full

// Send and receive
ch <- 42      // send
val := <-ch   // receive
val, ok := <-ch  // ok=false if channel closed and empty

// Close channel - signals no more values
close(ch)
// Rules:
// - Only sender should close
// - Sending to closed channel PANICS
// - Receiving from closed channel returns zero value immediately

// Range over channel - loops until closed
for val := range ch {
    fmt.Println(val)
}

// Select - multiplexing channels (CRITICAL)
select {
case msg := <-ch1:
    fmt.Println("received", msg)
case ch2 <- 42:
    fmt.Println("sent")
case <-time.After(time.Second):
    fmt.Println("timeout")
default:
    fmt.Println("no communication ready")
}

// Channel directions (for function parameters)
func send(ch chan<- int) { ch <- 1 }    // send-only
func recv(ch <-chan int) { <-ch }       // receive-only

// Nil channel behavior
var ch chan int  // nil
// ch <- 1       // blocks forever
// <-ch          // blocks forever
// Useful in select to disable cases dynamically
```

**Unbuffered vs Buffered**:
- Unbuffered: Synchronization point (sender waits for receiver)
- Buffered: Decoupling (sender continues until buffer full)

---

### 1.8 Context (Required for Production Code)

```go
import "context"

// Context carries deadlines, cancellation signals, and values
// ALWAYS first parameter by convention

func doWork(ctx context.Context) error {
    select {
    case <-ctx.Done():
        return ctx.Err()  // context.Canceled or context.DeadlineExceeded
    case result := <-doExpensiveOperation():
        return nil
    }
}

// Creating contexts
ctx := context.Background()  // root context, never canceled
ctx := context.TODO()        // placeholder when unsure

// With cancellation
ctx, cancel := context.WithCancel(parentCtx)
defer cancel()  // ALWAYS defer cancel to avoid leaks

// With timeout
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
defer cancel()

// With deadline
ctx, cancel := context.WithDeadline(parentCtx, time.Now().Add(5*time.Second))
defer cancel()

// With values (use sparingly - for request-scoped data only)
ctx = context.WithValue(ctx, "userID", 123)
userID := ctx.Value("userID").(int)

// HTTP example
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()  // request context, canceled when client disconnects
    
    select {
    case <-ctx.Done():
        http.Error(w, "Request canceled", http.StatusRequestTimeout)
        return
    case result := <-process(ctx):
        json.NewEncoder(w).Encode(result)
    }
}
```

---

### 1.9 Error Handling

```go
// Errors are values, not exceptions
// error interface: just Error() string
type error interface {
    Error() string
}

// Return error as last return value (convention)
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// Check errors immediately
result, err := divide(10, 0)
if err != nil {
    log.Fatal(err)  // or return err, or handle
}

// Custom errors
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

// Wrapping errors (Go 1.13+)
if err != nil {
    return fmt.Errorf("failed to process: %w", err)  // %w wraps
}

// Unwrapping
errors.Unwrap(err)           // get wrapped error
errors.Is(err, targetErr)    // check if err or wrapped equals target
errors.As(err, &target)      // check if err or wrapped is type of target

// Sentinel errors
var ErrNotFound = errors.New("not found")

if errors.Is(err, ErrNotFound) {
    // handle not found
}
```

---

### 1.10 Defer, Panic, Recover

```go
// Defer - executes when function returns, LIFO order
func example() {
    defer fmt.Println("1")
    defer fmt.Println("2")
    defer fmt.Println("3")
    // Output: 3, 2, 1
}

// Arguments evaluated when defer executed, not when called
func example2() {
    x := 1
    defer fmt.Println(x)  // prints 1, not 2
    x = 2
}

// Common uses: cleanup, unlock, close files
func readFile() error {
    f, err := os.Open("file.txt")
    if err != nil {
        return err
    }
    defer f.Close()  // guaranteed cleanup
    
    // work with file...
}

// Panic - for unrecoverable errors
func mustPositive(n int) {
    if n < 0 {
        panic("negative number")
    }
}

// Recover - catch panics, only works in deferred function
func safeCall() (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered: %v", r)
        }
    }()
    
    riskyFunction()  // might panic
    return nil
}
```

**Rule**: Don't use panic for normal error handling. Only for truly unrecoverable situations or programmer errors.

---

### 1.11 sync Package (Concurrency Primitives)

```go
import "sync"

// Mutex - mutual exclusion lock
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

// RWMutex - multiple readers OR one writer
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

func (c *Cache) Get(key string) string {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.data[key]
}

func (c *Cache) Set(key, val string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[key] = val
}

// WaitGroup - wait for goroutines
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    // work
}()
wg.Wait()

// Once - execute exactly once (singleton, init)
var once sync.Once
var instance *Database

func GetDB() *Database {
    once.Do(func() {
        instance = connectDatabase()
    })
    return instance
}

// Pool - object pool for reuse
var bufPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

buf := bufPool.Get().(*bytes.Buffer)
buf.Reset()
// use buffer
bufPool.Put(buf)

// Cond - condition variable (rarely used directly)
cond := sync.NewCond(&sync.Mutex{})
cond.Wait()    // releases lock and waits
cond.Signal()  // wakes one waiter
cond.Broadcast() // wakes all waiters
```

---

### 1.12 Generics (Go 1.18+)

```go
// Type parameters in functions
func Min[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

// Usage
Min[int](1, 2)    // explicit type
Min(1.5, 2.5)     // type inference

// Type constraints
type Number interface {
    int | int64 | float64
}

func Sum[T Number](nums []T) T {
    var sum T
    for _, n := range nums {
        sum += n
    }
    return sum
}

// Generic types
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

// Common constraints
// any - alias for interface{}
// comparable - supports == and !=
// constraints.Ordered - supports < > <= >=
```

---

## 2. Most Frequently Asked Interview Questions & Topics

### Easy / Basic Questions

| Topic | Question | Key Point |
|-------|----------|-----------|
| Slices | Difference between array and slice? | Array fixed, value type; Slice dynamic, reference type |
| Slices | What happens when you append beyond capacity? | New underlying array allocated (~2x size) |
| Maps | What's the zero value of a map? | nil - can read, cannot write |
| Interfaces | How does Go implement interfaces? | Implicitly - no "implements" keyword |
| Goroutines | What's the cost of a goroutine? | ~2KB stack, very lightweight |
| Error | How are errors handled in Go? | Return value, not exceptions |

### Medium / System Questions

| Topic | Question | Key Point |
|-------|----------|-----------|
| Channels | Unbuffered vs buffered channels? | Unbuffered syncs, buffered decouples |
| Select | What if multiple cases ready in select? | Random selection |
| Context | When would you use context? | Timeouts, cancellation, request-scoped values |
| Concurrency | How to safely access shared data? | Mutex or channels ("share by communicating") |
| Memory | Does Go have garbage collection? | Yes, concurrent tri-color mark-and-sweep |
| Nil | What's a nil interface vs nil pointer? | Interface nil only if type AND value nil |

### Hard / Deep-Dive Questions

| Topic | Question | Key Point |
|-------|----------|-----------|
| Race | How to detect race conditions? | `go run -race`, `go test -race` |
| Scheduler | How does the Go scheduler work? | M:N threading, work-stealing, GOMAXPROCS |
| Memory | What causes a goroutine leak? | Blocked forever on channel, never garbage collected |
| GC | How to reduce GC pressure? | Object pooling, avoid allocations, sync.Pool |
| Channels | Can you range over a channel? When does it stop? | Yes, stops when closed |
| Defer | Order of multiple defers? When are args evaluated? | LIFO, args evaluated immediately |

### System Design Questions

1. **Design a Rate Limiter in Go** - Use `time.Ticker`, channels, or `golang.org/x/time/rate`
2. **Design a Worker Pool** - Bounded goroutines, job channel, result channel
3. **Design a Cache with Expiration** - sync.RWMutex, time.AfterFunc, or dedicated goroutine
4. **Design a Pub/Sub System** - Channels per subscriber, broker pattern
5. **Why choose Go over Java/Python?** - Simplicity, fast compilation, native concurrency, single binary deployment

---

## 3. One-Page Cheat Sheet

### Type Quick Reference

| Type | Zero Value | Make Required | Notes |
|------|------------|---------------|-------|
| `bool` | `false` | No | |
| `int, float` | `0` | No | |
| `string` | `""` | No | Immutable |
| `pointer` | `nil` | No | |
| `slice` | `nil` | Yes for non-nil | Can append to nil |
| `map` | `nil` | Yes | Read OK, write panics |
| `channel` | `nil` | Yes | Blocks forever if nil |
| `interface` | `nil` | No | Nil if type AND value nil |
| `struct` | fields zero | No | |

### Operations Complexity

| Operation | Slice | Map |
|-----------|-------|-----|
| Access | O(1) | O(1) avg |
| Append | O(1) amortized | N/A |
| Insert | O(n) | O(1) avg |
| Delete | O(n) | O(1) avg |
| Search | O(n) | O(1) avg |
| Iteration | O(n) | O(n) |

### Channel Operations

| Operation | Nil Channel | Closed Channel | Active Channel |
|-----------|-------------|----------------|----------------|
| Send `ch<-` | Block forever | **PANIC** | Block/succeed |
| Receive `<-ch` | Block forever | Zero value, ok=false | Block/succeed |
| Close | **PANIC** | **PANIC** | Succeed |

### Essential Patterns

```go
// Comma-ok idiom
val, ok := m[key]
val, ok := <-ch
val, ok := x.(Type)

// Error check
if err != nil { return err }

// Defer for cleanup
defer file.Close()
defer mu.Unlock()

// Select timeout
select {
case r := <-ch:
case <-time.After(timeout):
}

// Done channel pattern
done := make(chan struct{})
close(done)  // broadcast to all listeners
```

### Common Imports

```go
import (
    "context"      // cancellation, timeouts
    "errors"       // error handling
    "fmt"          // printing
    "io"           // interfaces
    "log"          // logging
    "net/http"     // web server
    "sync"         // mutex, waitgroup
    "time"         // time operations
    "encoding/json" // JSON
)
```

---

## 4. Best Practices & Common Mistakes

### ✅ What Senior Engineers Do

```go
// 1. Handle errors explicitly
result, err := doSomething()
if err != nil {
    return fmt.Errorf("context: %w", err)  // wrap with context
}

// 2. Use defer for cleanup immediately after resource acquisition
f, err := os.Open(name)
if err != nil {
    return err
}
defer f.Close()  // right after successful open

// 3. Pass context as first parameter
func ProcessRequest(ctx context.Context, req Request) error

// 4. Use pointer receiver consistently
func (s *Service) Method1() {}
func (s *Service) Method2() {}  // all pointer receivers

// 5. Accept interfaces, return structs
func Process(r io.Reader) *Result  // flexible input, concrete output

// 6. Use table-driven tests
tests := []struct{
    name string
    input int
    want  int
}{
    {"positive", 5, 10},
    {"zero", 0, 0},
}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) { ... })
}

// 7. Make zero values useful
type Buffer struct {
    data []byte  // nil slice works with append
}
var b Buffer
b.data = append(b.data, 'a')  // works!
```

### ❌ Common Mistakes That Get You Rejected

```go
// 1. Ignoring errors
result, _ := doSomething()  // NEVER do this in production code

// 2. Loop variable capture in goroutine
for _, v := range values {
    go func() {
        process(v)  // BUG: v shared by all goroutines
    }()
}
// FIX: pass as parameter
for _, v := range values {
    go func(v Value) {
        process(v)
    }(v)
}

// 3. Not closing channels (causes goroutine leaks)
ch := make(chan int)
go producer(ch)
// producer must close(ch) when done!

// 4. Forgetting to unlock mutex
mu.Lock()
// code that might return early or panic
// mutex never unlocked!
// FIX: defer mu.Unlock() immediately after Lock()

// 5. Sending on closed channel
close(ch)
ch <- 1  // PANIC!

// 6. Using map for concurrent access without sync
// RACE CONDITION
go func() { m[key] = value }()
go func() { _ = m[key] }()

// 7. Nil map write
var m map[string]int
m["key"] = 1  // PANIC!

// 8. Comparing slices with ==
s1 := []int{1, 2}
s2 := []int{1, 2}
// s1 == s2  // COMPILE ERROR
// Use reflect.DeepEqual or slices.Equal

// 9. Modifying slice in range
for i, v := range slice {
    slice = append(slice, v*2)  // infinite loop risk, undefined behavior
}

// 10. Context value for wrong purposes
ctx = context.WithValue(ctx, "db", dbConn)  // Wrong! Use dependency injection
```

---

## 5. Minimal but Complete Code Examples

### Example 1: Worker Pool Pattern (Asked in 70% of Go interviews)

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Job represents work to be done
type Job struct {
    ID   int
    Data string
}

// Result represents completed work
type Result struct {
    JobID  int
    Output string
}

// Worker processes jobs from jobs channel, sends results to results channel
func worker(id int, jobs <-chan Job, results chan<- Result, wg *sync.WaitGroup) {
    defer wg.Done()
    
    for job := range jobs {  // exits when jobs closed
        // Simulate work
        time.Sleep(100 * time.Millisecond)
        
        results <- Result{
            JobID:  job.ID,
            Output: fmt.Sprintf("Worker %d processed: %s", id, job.Data),
        }
    }
}

func main() {
    const numWorkers = 3
    const numJobs = 10
    
    jobs := make(chan Job, numJobs)
    results := make(chan Result, numJobs)
    
    // Start workers
    var wg sync.WaitGroup
    for w := 1; w <= numWorkers; w++ {
        wg.Add(1)
        go worker(w, jobs, results, &wg)
    }
    
    // Send jobs
    for j := 1; j <= numJobs; j++ {
        jobs <- Job{ID: j, Data: fmt.Sprintf("data-%d", j)}
    }
    close(jobs)  // signal no more jobs
    
    // Wait for workers then close results
    go func() {
        wg.Wait()
        close(results)
    }()
    
    // Collect results
    for result := range results {
        fmt.Println(result.Output)
    }
}
```

---

### Example 2: HTTP Server with Context & Graceful Shutdown

```go
package main

import (
    "context"
    "encoding/json"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

type Response struct {
    Message string `json:"message"`
    Time    string `json:"time"`
}

func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    // Simulate long operation that respects context
    select {
    case <-time.After(2 * time.Second):
        resp := Response{
            Message: "Hello, World!",
            Time:    time.Now().Format(time.RFC3339),
        }
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(resp)
        
    case <-ctx.Done():
        // Client disconnected or timeout
        http.Error(w, "Request cancelled", http.StatusRequestTimeout)
        log.Println("Request cancelled:", ctx.Err())
    }
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/api/hello", handler)
    
    server := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  120 * time.Second,
    }
    
    // Start server in goroutine
    go func() {
        log.Println("Server starting on :8080")
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()
    
    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    log.Println("Shutting down server...")
    
    // Graceful shutdown with timeout
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    if err := server.Shutdown(ctx); err != nil {
        log.Fatal("Server forced to shutdown:", err)
    }
    
    log.Println("Server exited properly")
}
```

---

### Example 3: Interface-Based Design (Dependency Injection)

```go
package main

import (
    "fmt"
    "io"
    "strings"
)

// Repository interface - abstracts data storage
type UserRepository interface {
    GetUser(id int) (*User, error)
    SaveUser(user *User) error
}

type User struct {
    ID   int
    Name string
}

// In-memory implementation (for testing)
type MemoryRepo struct {
    users map[int]*User
}

func NewMemoryRepo() *MemoryRepo {
    return &MemoryRepo{users: make(map[int]*User)}
}

func (r *MemoryRepo) GetUser(id int) (*User, error) {
    user, exists := r.users[id]
    if !exists {
        return nil, fmt.Errorf("user %d not found", id)
    }
    return user, nil
}

func (r *MemoryRepo) SaveUser(user *User) error {
    r.users[user.ID] = user
    return nil
}

// Service depends on interface, not concrete type
type UserService struct {
    repo UserRepository  // interface field
}

func NewUserService(repo UserRepository) *UserService {
    return &UserService{repo: repo}
}

func (s *UserService) GetUserName(id int) (string, error) {
    user, err := s.repo.GetUser(id)
    if err != nil {
        return "", err
    }
    return user.Name, nil
}

// Also implement io.Writer for logging example
func (s *UserService) LogTo(w io.Writer, msg string) {
    fmt.Fprintln(w, msg)  // works with any io.Writer
}

func main() {
    // Inject memory repo (could be DB repo in production)
    repo := NewMemoryRepo()
    service := NewUserService(repo)
    
    repo.SaveUser(&User{ID: 1, Name: "Alice"})
    
    name, err := service.GetUserName(1)
    if err != nil {
        panic(err)
    }
    fmt.Println("User name:", name)
    
    // io.Writer interface - works with anything
    var builder strings.Builder
    service.LogTo(&builder, "Log message")
    fmt.Println("Logged:", builder.String())
}
```

---

### Example 4: Rate Limiter with Token Bucket

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type RateLimiter struct {
    tokens         int
    maxTokens      int
    refillRate     int           // tokens per refill
    refillInterval time.Duration
    mu             sync.Mutex
    stopCh         chan struct{}
}

func NewRateLimiter(maxTokens, refillRate int, refillInterval time.Duration) *RateLimiter {
    rl := &RateLimiter{
        tokens:         maxTokens,
        maxTokens:      maxTokens,
        refillRate:     refillRate,
        refillInterval: refillInterval,
        stopCh:         make(chan struct{}),
    }
    go rl.refillLoop()
    return rl
}

func (rl *RateLimiter) refillLoop() {
    ticker := time.NewTicker(rl.refillInterval)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            rl.mu.Lock()
            rl.tokens += rl.refillRate
            if rl.tokens > rl.maxTokens {
                rl.tokens = rl.maxTokens
            }
            rl.mu.Unlock()
        case <-rl.stopCh:
            return
        }
    }
}

// Allow checks if request is allowed and consumes a token
func (rl *RateLimiter) Allow() bool {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    
    if rl.tokens > 0 {
        rl.tokens--
        return true
    }
    return false
}

func (rl *RateLimiter) Stop() {
    close(rl.stopCh)
}

func main() {
    // 10 requests max, refill 2 tokens per second
    limiter := NewRateLimiter(10, 2, time.Second)
    defer limiter.Stop()
    
    // Simulate 15 requests
    for i := 0; i < 15; i++ {
        if limiter.Allow() {
            fmt.Printf("Request %d: ALLOWED\n", i+1)
        } else {
            fmt.Printf("Request %d: DENIED\n", i+1)
        }
    }
    
    // Wait for refill
    fmt.Println("\nWaiting for refill...")
    time.Sleep(2 * time.Second)
    
    for i := 0; i < 5; i++ {
        if limiter.Allow() {
            fmt.Printf("Request %d: ALLOWED\n", i+1)
        } else {
            fmt.Printf("Request %d: DENIED\n", i+1)
        }
    }
}
```

---

### Example 5: Generic LRU Cache

```go
package main

import (
    "container/list"
    "fmt"
    "sync"
)

type LRUCache[K comparable, V any] struct {
    capacity int
    cache    map[K]*list.Element
    list     *list.List
    mu       sync.RWMutex
}

type entry[K comparable, V any] struct {
    key   K
    value V
}

func NewLRUCache[K comparable, V any](capacity int) *LRUCache[K, V] {
    return &LRUCache[K, V]{
        capacity: capacity,
        cache:    make(map[K]*list.Element),
        list:     list.New(),
    }
}

func (c *LRUCache[K, V]) Get(key K) (V, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    if elem, exists := c.cache[key]; exists {
        c.list.MoveToFront(elem)
        return elem.Value.(*entry[K, V]).value, true
    }
    var zero V
    return zero, false
}

func (c *LRUCache[K, V]) Put(key K, value V) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    // Update existing
    if elem, exists := c.cache[key]; exists {
        c.list.MoveToFront(elem)
        elem.Value.(*entry[K, V]).value = value
        return
    }
    
    // Evict if at capacity
    if c.list.Len() >= c.capacity {
        oldest := c.list.Back()
        if oldest != nil {
            c.list.Remove(oldest)
            delete(c.cache, oldest.Value.(*entry[K, V]).key)
        }
    }
    
    // Add new entry
    e := &entry[K, V]{key: key, value: value}
    elem := c.list.PushFront(e)
    c.cache[key] = elem
}

func main() {
    cache := NewLRUCache[string, int](3)
    
    cache.Put("a", 1)
    cache.Put("b", 2)
    cache.Put("c", 3)
    
    if val, ok := cache.Get("a"); ok {
        fmt.Println("a =", val)  // 1, also moves "a" to front
    }
    
    cache.Put("d", 4)  // evicts "b" (least recently used)
    
    if _, ok := cache.Get("b"); !ok {
        fmt.Println("b was evicted")
    }
    
    if val, ok := cache.Get("c"); ok {
        fmt.Println("c =", val)
    }
}
```

---

## 6. Recommended Problems to Solve Today

### LeetCode Problems (Go Practice)

| # | Problem | Difficulty | Why Important |
|---|---------|------------|---------------|
| 1 | **Two Sum** | Easy | Map usage, comma-ok |
| 206 | **Reverse Linked List** | Easy | Pointers in Go |
| 20 | **Valid Parentheses** | Easy | Stack with slice |
| 146 | **LRU Cache** | Medium | Struct, map, linked list |
| 200 | **Number of Islands** | Medium | BFS/DFS, 2D slices |
| 56 | **Merge Intervals** | Medium | Sorting, custom comparator |
| 23 | **Merge K Sorted Lists** | Hard | Heap (container/heap) |
| 297 | **Serialize/Deserialize Tree** | Hard | Recursion, string handling |

### Go-Specific Concurrency Problems

| Problem | Description | Concepts Tested |
|---------|-------------|-----------------|
| **Print Numbers Alternately** | Two goroutines print 1,2,3,4,5 alternating | Channels, synchronization |
| **Web Crawler** | Concurrent web crawler with deduplication | Goroutines, mutex, map |
| **Bounded Parallel Execution** | Execute N tasks with max M parallel | Worker pool, semaphore |
| **Pipeline** | Build data processing pipeline | Channel chaining |
| **Timeout Handler** | HTTP handler with timeout | Context, select |

### System Design Mini-Projects

1. **In-Memory Key-Value Store** - With TTL, concurrent access
2. **URL Shortener** - ID generation, base62 encoding
3. **Task Queue** - Persistent, with retries
4. **Load Balancer** - Round-robin, health checks

---

## 7. 30-Minute Quiz

Answer these without looking at notes. Answers at the end.

**Q1.** What's the output?
```go
s := []int{1, 2, 3}
s2 := s[1:2]
s2 = append(s2, 4)
fmt.Println(s)
```

**Q2.** What happens when you send on a closed channel?

**Q3.** What's the zero value of a slice? Can you append to it?

**Q4.** How do you detect race conditions in Go?

**Q5.** What's wrong with this code?
```go
for i := 0; i < 10; i++ {
    go func() {
        fmt.Println(i)
    }()
}
```

**Q6.** When is a Go interface nil?

**Q7.** What's the difference between `sync.Mutex` and `sync.RWMutex`?

**Q8.** What does `select {}` do?

**Q9.** In what order do multiple `defer` statements execute?

**Q10.** What's the output?
```go
var m map[string]int
fmt.Println(m["key"])
```

---

### Quiz Answers

<details>
<summary>Click to reveal answers</summary>

**A1.** `[1 2 4]` - s2 shares underlying array, append modifies s[2]

**A2.** **PANIC** - Sending on closed channel panics

**A3.** `nil`. Yes, you can append to nil slice (append handles it)

**A4.** `go run -race main.go` or `go test -race`

**A5.** **Loop variable capture bug** - All goroutines share same `i`, likely all print 10. Fix: pass `i` as parameter

**A6.** Only when **both** type and value are nil. `var r io.Reader; r = (*bytes.Buffer)(nil)` is NOT nil

**A7.** Mutex is exclusive. RWMutex allows multiple concurrent readers OR one writer

**A8.** **Blocks forever** - Empty select with no cases

**A9.** **LIFO** (Last In, First Out) - Like a stack

**A10.** `0` - Reading from nil map returns zero value (doesn't panic). Writing would panic

</details>

---

## 8. Further Learning

### Top 1 Resource (Video/Article)
**"Learn Go in 12 Minutes"** by Fireship + **"Concurrency in Go"** by Jake Wright
- [Fireship Go in 100 Seconds](https://www.youtube.com/watch?v=446E-r0rXHI)
- For deeper dive: **"Go Concurrency Patterns"** by Rob Pike (Google I/O) - YouTube

### Official Documentation
**[Effective Go](https://go.dev/doc/effective_go)** - The definitive guide written by Go team
- Also: **[Go Blog](https://go.dev/blog/)** for advanced topics like generics, memory model

---

## Quick Reference Card (Print This!)

```
┌──────────────────────────────────────────────────────────────────┐
│                     GO INTERVIEW QUICK REF                       │
├──────────────────────────────────────────────────────────────────┤
│ SLICES                          │ CHANNELS                       │
│ make([]T, len, cap)             │ ch := make(chan T)  unbuffered │
│ append(s, x) → returns new      │ ch := make(chan T, n) buffered │
│ copy(dst, src)                  │ ch <- v   send                 │
│ s[low:high:max]                 │ v := <-ch receive              │
│                                 │ close(ch) sender only          │
├─────────────────────────────────┼────────────────────────────────┤
│ MAPS                            │ SELECT                         │
│ m := make(map[K]V)              │ select {                       │
│ v, ok := m[key]                 │ case v := <-ch1:               │
│ delete(m, key)                  │ case ch2 <- v:                 │
│ for k, v := range m             │ case <-time.After(t):          │
│                                 │ default: // non-blocking       │
├─────────────────────────────────┼────────────────────────────────┤
│ GOROUTINES                      │ SYNC                           │
│ go func(){}()                   │ var mu sync.Mutex              │
│ runtime.GOMAXPROCS(n)           │ mu.Lock() defer mu.Unlock()    │
│                                 │ var wg sync.WaitGroup          │
│ CONTEXT                         │ wg.Add(1) wg.Done() wg.Wait()  │
│ ctx, cancel := WithTimeout()    │ var once sync.Once             │
│ defer cancel()                  │ once.Do(func(){})              │
├─────────────────────────────────┴────────────────────────────────┤
│ MEMORY: Closed chan recv = zero+false │ Nil chan = block forever │
│ ERROR: if err != nil { return fmt.Errorf("ctx: %w", err) }       │
│ TEST: go test -race -cover ./...                                 │
└──────────────────────────────────────────────────────────────────┘
```

---

**Good luck with your interviews! 🚀**
