# Go Concurrency Patterns

## Goroutines and the Go Scheduler

A goroutine is a lightweight, cooperatively-scheduled execution unit managed by the Go runtime — not the OS. The runtime uses an M:N scheduler: M goroutines multiplexed onto N OS threads.

- **M** = OS threads (controlled by `GOMAXPROCS`, defaults to number of CPU cores)
- **N** = goroutines (can be millions — each starts at ~2KB stack)
- **P** = logical processor, the bridge between goroutines and OS threads

```
Goroutines (G)  →  Logical Processors (P)  →  OS Threads (M)
```

`GOMAXPROCS` determines how many goroutines can run in parallel (not concurrently). Set it for benchmarks:

```go
import "runtime"

func init() {
    runtime.GOMAXPROCS(4) // run on 4 OS threads in parallel
}
```

The scheduler is preemptive since Go 1.14: goroutines can be interrupted at any safe point, not just at function calls.

---

## Channels

### Unbuffered vs Buffered

| | Unbuffered `make(chan T)` | Buffered `make(chan T, n)` |
|---|---|---|
| Send blocks? | Until receiver is ready | Until buffer is full |
| Receive blocks? | Until sender is ready | Until buffer is empty |
| Synchronization | Guaranteed rendezvous | Decoupled |
| Use when | Handoff / synchronization | Smoothing bursts |

```go
package main

import "fmt"

func main() {
    // Unbuffered: sender and receiver must meet
    ch := make(chan int)
    go func() { ch <- 42 }() // goroutine blocks until main receives
    fmt.Println(<-ch)

    // Buffered: sender can proceed without a receiver waiting
    buf := make(chan string, 3)
    buf <- "a"
    buf <- "b"
    buf <- "c"
    // buf <- "d" would block — buffer full
    fmt.Println(<-buf) // "a"
}
```

### Directional Channels

Restrict what a function can do with a channel. This is a compile-time guarantee.

```go
func producer(out chan<- int) { // send-only
    for i := 0; i < 5; i++ {
        out <- i
    }
    close(out)
}

func consumer(in <-chan int) { // receive-only
    for v := range in {
        fmt.Println(v)
    }
}

func main() {
    ch := make(chan int, 5)
    go producer(ch)
    consumer(ch)
}
```

### Closing Channels and Ranging

```go
func main() {
    ch := make(chan int, 5)
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch) // signals no more sends; receivers drain the buffer

    // range stops when channel is closed and drained
    for v := range ch {
        fmt.Println(v)
    }

    // Two-value receive: check if channel is open
    v, ok := <-ch
    fmt.Println(v, ok) // 0 false — zero value, channel closed
}
```

**Rules:**
- Only senders should close — closing a send-only channel is convention
- Closing a closed channel panics
- Sending on a closed channel panics
- Receiving from a closed channel returns zero value immediately

---

## select Statement

`select` is like a switch for channels. It blocks until one case is ready; if multiple are ready, one is chosen at random.

### Fan-In

```go
func fanIn(cs ...<-chan int) <-chan int {
    out := make(chan int)
    for _, c := range cs {
        c := c // capture
        go func() {
            for v := range c {
                out <- v
            }
        }()
    }
    return out
}
```

### Timeout

```go
import "time"

func withTimeout(ch <-chan string, timeout time.Duration) (string, bool) {
    select {
    case v := <-ch:
        return v, true
    case <-time.After(timeout):
        return "", false
    }
}
```

### Non-Blocking Receive

```go
func tryReceive(ch <-chan int) (int, bool) {
    select {
    case v := <-ch:
        return v, true
    default:
        return 0, false
    }
}
```

---

## sync Package

### Mutex and RWMutex

```go
import "sync"

type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

func NewSafeMap() *SafeMap {
    return &SafeMap{m: make(map[string]int)}
}

func (s *SafeMap) Set(k string, v int) {
    s.mu.Lock()   // exclusive write lock
    defer s.mu.Unlock()
    s.m[k] = v
}

func (s *SafeMap) Get(k string) (int, bool) {
    s.mu.RLock()  // shared read lock — multiple readers OK
    defer s.mu.RUnlock()
    v, ok := s.m[k]
    return v, ok
}
```

### WaitGroup

```go
func main() {
    var wg sync.WaitGroup
    results := make([]int, 10)

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(idx int) {
            defer wg.Done()
            results[idx] = idx * idx // safe: each goroutine writes to its own index
        }(i)
    }

    wg.Wait() // block until all goroutines call Done
    fmt.Println(results)
}
```

### sync.Once — exactly-once initialization

```go
type Singleton struct{ config string }

var (
    instance *Singleton
    once     sync.Once
)

func GetInstance() *Singleton {
    once.Do(func() {
        instance = &Singleton{config: "loaded"}
    })
    return instance
}
```

### atomic — lock-free primitives

```go
import "sync/atomic"

var counter int64

func increment() {
    atomic.AddInt64(&counter, 1)  // atomic, no mutex needed
}

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            increment()
        }()
    }
    wg.Wait()
    fmt.Println(atomic.LoadInt64(&counter)) // 1000
}
```

---

## Classic Go Concurrency Patterns

### Pipeline

Stages connected by channels. Each stage consumes from upstream, produces to downstream.

```go
package main

import "fmt"

// Stage 1: generate numbers
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

// Stage 2: square each number
func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

// Stage 3: filter even numbers
func evens(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            if n%2 == 0 {
                out <- n
            }
        }
        close(out)
    }()
    return out
}

func main() {
    // compose pipeline
    nums := generate(1, 2, 3, 4, 5, 6, 7, 8)
    squared := square(nums)
    filtered := evens(squared)

    for v := range filtered {
        fmt.Println(v) // 4, 16, 36, 64
    }
}
```

### Fan-Out / Fan-In

Distribute work across multiple goroutines, collect results.

```go
package main

import (
    "fmt"
    "sync"
)

func fanOut(in <-chan int, workers int) []<-chan int {
    outs := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        out := make(chan int)
        outs[i] = out
        go func(out chan<- int) {
            for v := range in {
                out <- v * v // do work
            }
            close(out)
        }(out)
    }
    return outs
}

func merge(cs []<-chan int) <-chan int {
    var wg sync.WaitGroup
    merged := make(chan int)

    output := func(c <-chan int) {
        defer wg.Done()
        for v := range c {
            merged <- v
        }
    }

    wg.Add(len(cs))
    for _, c := range cs {
        go output(c)
    }

    go func() {
        wg.Wait()
        close(merged)
    }()
    return merged
}

func main() {
    in := make(chan int, 10)
    for i := 1; i <= 10; i++ {
        in <- i
    }
    close(in)

    outs := fanOut(in, 3)
    for v := range merge(outs) {
        fmt.Println(v)
    }
}
```

### Worker Pool

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Job struct {
    ID    int
    Input string
}

type Result struct {
    Job    Job
    Output string
}

func worker(id int, jobs <-chan Job, results chan<- Result, wg *sync.WaitGroup) {
    defer wg.Done()
    for job := range jobs {
        time.Sleep(10 * time.Millisecond) // simulate work
        results <- Result{
            Job:    job,
            Output: fmt.Sprintf("worker-%d processed job-%d", id, job.ID),
        }
    }
}

func main() {
    const numWorkers = 5
    const numJobs = 20

    jobs := make(chan Job, numJobs)
    results := make(chan Result, numJobs)

    var wg sync.WaitGroup
    for i := 1; i <= numWorkers; i++ {
        wg.Add(1)
        go worker(i, jobs, results, &wg)
    }

    for i := 1; i <= numJobs; i++ {
        jobs <- Job{ID: i, Input: fmt.Sprintf("data-%d", i)}
    }
    close(jobs) // signal workers: no more jobs

    go func() {
        wg.Wait()
        close(results)
    }()

    for r := range results {
        fmt.Println(r.Output)
    }
}
```

### Semaphore with Buffered Channel

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Semaphore chan struct{}

func NewSemaphore(n int) Semaphore {
    return make(chan struct{}, n)
}

func (s Semaphore) Acquire() { s <- struct{}{} }
func (s Semaphore) Release() { <-s }

func main() {
    sem := NewSemaphore(3) // at most 3 concurrent operations
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            sem.Acquire()
            defer sem.Release()
            fmt.Printf("goroutine %d: running\n", id)
            time.Sleep(100 * time.Millisecond)
            fmt.Printf("goroutine %d: done\n", id)
        }(i)
    }
    wg.Wait()
}
```

### Context for Cancellation and Timeout

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func doWork(ctx context.Context, id int) error {
    select {
    case <-time.After(2 * time.Second): // simulates long operation
        fmt.Printf("worker %d finished\n", id)
        return nil
    case <-ctx.Done():
        fmt.Printf("worker %d cancelled: %v\n", id, ctx.Err())
        return ctx.Err()
    }
}

func main() {
    // Cancellation
    ctx, cancel := context.WithCancel(context.Background())
    go func() {
        time.Sleep(500 * time.Millisecond)
        cancel() // cancel after 500ms
    }()
    doWork(ctx, 1) // worker 1 cancelled

    // Timeout
    ctx2, cancel2 := context.WithTimeout(context.Background(), 300*time.Millisecond)
    defer cancel2()
    doWork(ctx2, 2) // worker 2 cancelled by timeout

    // Deadline
    deadline := time.Now().Add(1 * time.Second)
    ctx3, cancel3 := context.WithDeadline(context.Background(), deadline)
    defer cancel3()
    doWork(ctx3, 3)
}
```

### Rate Limiter with time.Ticker

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // Allow 5 requests per second
    ticker := time.NewTicker(200 * time.Millisecond)
    defer ticker.Stop()

    requests := make(chan int, 20)
    for i := 1; i <= 10; i++ {
        requests <- i
    }
    close(requests)

    for req := range requests {
        <-ticker.C // block until next tick
        fmt.Printf("request %d at %v\n", req, time.Now().Format("15:04:05.000"))
    }
}
```

### Pub/Sub with Channels

```go
package main

import (
    "fmt"
    "sync"
)

type PubSub struct {
    mu          sync.RWMutex
    subscribers map[string][]chan interface{}
}

func NewPubSub() *PubSub {
    return &PubSub{subscribers: make(map[string][]chan interface{})}
}

func (ps *PubSub) Subscribe(topic string) <-chan interface{} {
    ps.mu.Lock()
    defer ps.mu.Unlock()
    ch := make(chan interface{}, 10)
    ps.subscribers[topic] = append(ps.subscribers[topic], ch)
    return ch
}

func (ps *PubSub) Publish(topic string, msg interface{}) {
    ps.mu.RLock()
    defer ps.mu.RUnlock()
    for _, ch := range ps.subscribers[topic] {
        select {
        case ch <- msg:
        default:
            // subscriber's buffer full — drop message or handle
        }
    }
}

func (ps *PubSub) Close(topic string) {
    ps.mu.Lock()
    defer ps.mu.Unlock()
    for _, ch := range ps.subscribers[topic] {
        close(ch)
    }
    delete(ps.subscribers, topic)
}

func main() {
    ps := NewPubSub()

    sub1 := ps.Subscribe("orders")
    sub2 := ps.Subscribe("orders")

    var wg sync.WaitGroup
    for i, sub := range []<-chan interface{}{sub1, sub2} {
        wg.Add(1)
        go func(id int, ch <-chan interface{}) {
            defer wg.Done()
            for msg := range ch {
                fmt.Printf("subscriber %d: %v\n", id, msg)
            }
        }(i+1, sub)
    }

    ps.Publish("orders", map[string]int{"id": 1})
    ps.Publish("orders", map[string]int{"id": 2})
    ps.Close("orders")

    wg.Wait()
}
```

---

## Common Mistakes

### Goroutine Leaks

A goroutine leak happens when a goroutine is blocked forever with no way to exit. It's the most common production bug with goroutines.

```go
// BUG: goroutine leaks if nobody receives from ch
func leaky() {
    ch := make(chan int)
    go func() {
        result := compute() // takes a long time
        ch <- result        // blocks forever if caller gave up
    }()
    select {
    case v := <-ch:
        fmt.Println(v)
    case <-time.After(1 * time.Second):
        return // goroutine still blocked on ch <- result
    }
}

// FIX: use a buffered channel or context cancellation
func fixed(ctx context.Context) {
    ch := make(chan int, 1) // buffer of 1: sender never blocks
    go func() {
        result := compute()
        select {
        case ch <- result:
        case <-ctx.Done(): // respect cancellation
        }
    }()
    select {
    case v := <-ch:
        fmt.Println(v)
    case <-ctx.Done():
        return
    }
}
```

**Detecting leaks with pprof:**

```go
import _ "net/http/pprof"
import "net/http"

func main() {
    go http.ListenAndServe(":6060", nil)
    // ...
}
// Then: go tool pprof http://localhost:6060/debug/pprof/goroutine
// Look for goroutines in chan receive/send with growing counts
```

### Sending on a Closed Channel

```go
// This panics:
ch := make(chan int)
close(ch)
ch <- 1 // panic: send on closed channel

// Pattern: use a done channel to signal closure, not the data channel
done := make(chan struct{})
go func() {
    for {
        select {
        case <-done:
            return
        default:
            // do work, send to data channel
        }
    }
}()
close(done) // signals goroutine to stop — never panics
```

### Range Over Nil Channel

```go
var ch chan int // nil channel
// This blocks forever (not a panic):
for v := range ch {
    fmt.Println(v)
}

// Common when conditionally setting up a channel
var optional <-chan int
if condition {
    optional = realChannel
}
// Select on nil channel case never triggers — use this intentionally:
select {
case v := <-optional: // never selected if optional is nil
    fmt.Println(v)
default:
    fmt.Println("no value")
}
```

### Capturing Loop Variable in Goroutine

```go
// BUG: all goroutines print the same final value of i
for i := 0; i < 5; i++ {
    go func() { fmt.Println(i) }() // captures i by reference
}
// prints something like: 5 5 5 5 5

// FIX 1: pass as argument (before Go 1.22)
for i := 0; i < 5; i++ {
    go func(i int) { fmt.Println(i) }(i)
}

// FIX 2: shadow in loop body (before Go 1.22)
for i := 0; i < 5; i++ {
    i := i
    go func() { fmt.Println(i) }()
}

// Go 1.22+: loop variable is per-iteration automatically
```

---

## Go Concurrency Interview Questions

**Q: What is the difference between concurrency and parallelism in Go?**

Concurrency is about structuring a program as independently executing components (goroutines). Parallelism is about actually running them simultaneously. Go programs are concurrent by design; whether they run in parallel depends on `GOMAXPROCS` and available CPU cores. A single-core machine with `GOMAXPROCS=1` still runs concurrent goroutines — just not in parallel.

**Q: What happens if you close a channel twice?**

Panic. Only one goroutine should be responsible for closing a channel. Use `sync.Once` to ensure a channel is closed exactly once, or use a dedicated done-channel pattern where you signal via close rather than closing the data channel.

**Q: How do you prevent goroutine leaks?**

Always give goroutines a way to exit: pass a `context.Context` and check `ctx.Done()`, use a done channel that you close when finished, or ensure the channels they block on will eventually receive or be closed. Use `pprof` goroutine endpoint to audit in production.

**Q: When would you use a mutex vs a channel?**

Use a channel when passing ownership of data between goroutines or coordinating lifecycle (start, stop, signal). Use a mutex when protecting shared state that many goroutines need to access in place — it's simpler and has less overhead for that pattern. Rob Pike's rule: "Don't communicate by sharing memory; share memory by communicating" — but don't use channels where a mutex is clearly the right tool.

**Q: What is a select with a default case used for?**

Non-blocking channel operations. Without `default`, `select` blocks until a case is ready. With `default`, it executes the default case immediately if no channel is ready. Use it for polling, trying a send/receive without blocking, or building non-blocking APIs.

**Q: How does context propagation work?**

`context.WithCancel`, `WithTimeout`, and `WithDeadline` create child contexts. Cancelling a parent cancels all children. Pass contexts as the first argument to functions that do I/O or long computation. When `ctx.Done()` is closed, the function should clean up and return `ctx.Err()`. Never store context in a struct — pass it explicitly.

**Q: What is the difference between `sync.WaitGroup` and a done channel?**

`WaitGroup` is appropriate when you know upfront how many goroutines to wait for (`Add` before `go`). A done channel (`close(done)`) broadcasts to any number of goroutines at once — good for cancellation where you don't know how many listeners there are. They solve different problems and are often used together.
