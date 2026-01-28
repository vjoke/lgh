---
description: Displays high-performance Go programming patterns and best practices from VictoriaMetrics
arguments: true
---

# Go High-Performance Programming

This command provides guidance from the VictoriaMetrics codebase - one of the highest-performance Go applications.

{{if eq (len $ARGUMENTS) 0}}

## Core Patterns

### Memory Management
- **sync.Pool** - Object reuse to reduce GC pressure
- **Leveled Buffer Pools** - Size-specific buffer pools (powers of 2)
- **[:0] reset pattern** - Reuse slices without losing capacity
- **Resize strategies** - Overallocate, exact, or no-copy

### Concurrency
- **Sharded locks** - Reduce lock contention by splitting data structures
- **Atomic operations** - 10-100x faster than mutexes for simple operations
- **Cache-line padding** - Prevent false sharing between CPU cores
- **Local worker pools** - Each worker has its own channel
- **Channel semaphores** - Buffered channels as concurrency limiters

### Hot Path Optimizations
- **Unsafe zero-copy** - Direct string/bytes conversion
- **Custom encoding** - Nearest Delta encoding for time series
- **Fast time** - Atomic timestamp updated once per second

## Usage

Provide a specific topic for detailed patterns:

{{else}}
## {{ $ARGUMENTS }}

Below are key patterns for **{{ $ARGUMENTS }}** from VictoriaMetrics:

### Memory Management

#### Object Reuse with sync.Pool
```go
var bbPool = sync.Pool{
    New: func() any {
        return &ByteBuffer{}
    },
}

func Get() *ByteBuffer {
    bb := bbPool.Get().(*ByteBuffer)
    bb.B = bb.B[:0]  // Reset while keeping capacity
    return bb
}

func Put(bb *ByteBuffer) {
    bb.Reset()
    bbPool.Put(bb)
}
```

#### Slice Reuse with [:0]
```go
// Reset slice without losing capacity
data = data[:0]

// Append reuses underlying array
data = append(data, newData...)
```

#### Leveled Buffer Pools
```go
// pools[0] for 0-256 bytes
// pools[1] for 257-512 bytes
// pools[2] for 513-1024 bytes
var pools [10]sync.Pool

func Get(dataLen int) *ByteBuffer {
    id, _ := getPoolIDAndCapacity(dataLen)
    for i := 0; i < 2; i++ {
        if v := pools[id].Get(); v != nil {
            return v.(*ByteBuffer)
        }
        id++
    }
    return &ByteBuffer{B: make([]byte, 0, capacity)}
}
```

### Concurrency

#### Sharded Locks
```go
type Cache struct {
    shards []*cacheShard  // Each shard has its own lock
}

func (c *Cache) Get(key string) Value {
    idx := hash(key) % uint64(len(c.shards))
    return c.shards[idx].Get(key)
}
```

#### Atomic Operations (CAS Pattern)
```go
func (f *filter) Add(h uint64) bool {
    for {
        w := atomic.LoadUint64(&bits[i])
        if w&mask != 0 {
            return false  // Already set
        }
        wNew := w | mask
        if atomic.CompareAndSwapUint64(&bits[i], w, wNew) {
            return true  // Successfully set
        }
    }
}
```

#### Cache-Line Padding
```go
const CacheLineSize = 64

type Uint64 struct {
    _ [CacheLineSize - unsafe.Sizeof(atomic.Uint64{})%CacheLineSize]byte
    atomic.Uint64
    _ [CacheLineSize - unsafe.Sizeof(atomic.Uint64{})%CacheLineSize]byte
}
```

#### Channel Semaphore
```go
var concurrencyLimitCh = make(chan struct{}, maxConcurrent)

func Acquire() error {
    select {
    case concurrencyLimitCh <- struct{}{}:
        return nil
    default:
        return errors.New("limit reached")
    }
}

func Release() {
    <-concurrencyLimitCh
}
```

### Hot Path Optimizations

#### Unsafe Zero-Copy
```go
func ToUnsafeString(b []byte) string {
    return unsafe.String(unsafe.SliceData(b), len(b))
}

func ToUnsafeBytes(s string) []byte {
    return unsafe.Slice(unsafe.StringData(s), len(s))
}
```

**Warning**: Must ensure source data remains valid and unchanged.

#### Fast Time
```go
var currentTimestamp atomic.Int64

func init() {
    go func() {
        ticker := time.NewTicker(time.Second)
        defer ticker.Stop()
        for range ticker.C {
            currentTimestamp.Store(time.Now().Unix())
        }
    }()
}

func UnixTimestamp() uint64 {
    return currentTimestamp.Load()
}
```

### Logging

#### Rate-Limited Logging
```go
type logLimiter struct {
    mu       sync.Mutex
    counters map[string]uint64
}

func (ll *logLimiter) needSuppress(location string, limit uint64) bool {
    ll.mu.Lock()
    defer ll.mu.Unlock()

    count := ll.counters[location]
    if count >= limit {
        return true  // Suppress
    }
    ll.counters[location]++
    return false
}
```

## Resources

- VictoriaMetrics: https://github.com/VictoriaMetrics/VictoriaMetrics
- Documentation: https://docs.victoriametrics.com/

{{end}}
