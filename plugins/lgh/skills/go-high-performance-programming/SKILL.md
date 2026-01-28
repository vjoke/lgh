---
name: go-high-performance-programming
description: High-Performance Go Programming patterns from VictoriaMetrics. This skill provides battle-tested patterns for writing highly efficient Go code - memory optimization, concurrency patterns, zero-allocation techniques, and performance best practices.
license: Apache-2.0
metadata:
  author: VictoriaMetrics
  repository: https://github.com/VictoriaMetrics/VictoriaMetrics
  version: "1.0.0"
  tags: go, performance, optimization, memory, concurrency, victoriametrics
---

# Go High-Performance Programming

This skill provides guidance from the VictoriaMetrics codebase - one of the highest-performance Go applications. VictoriaMetrics achieves exceptional write speeds, low memory usage, and efficient CPU utilization through specific patterns and techniques.

## Repository: VictoriaMetrics

**URL**: https://github.com/VictoriaMetrics/VictoriaMetrics
**Language**: Go 1.21+
**License**: Apache-2.0

VictoriaMetrics is a high-performance, cost-effective time series database that demonstrates Go can achieve C-like performance when optimized correctly.

## Core Philosophy

> "Performance optimization has no black magic, only deep understanding of principles and extreme attention to detail."

Key principles:
1. **Zero-allocation hot paths** - Reuse everything possible
2. **Memory reuse** - Use pools, [:0] pattern, pre-allocate
3. **Lock contention reduction** - Shard, use atomics
4. **Cache-line awareness** - Pad to prevent false sharing
5. **Rate limiting** - Protect from high-frequency operations
6. **Simple code** - Fewer abstraction layers = better inlining

## Key Patterns and Techniques

### 1. Memory Management

#### Object Reuse with sync.Pool
**Location**: `lib/bytesutil/bytebuffer.go`, `lib/leveledbytebufferpool/pool.go`

Always reset objects before returning to pool to reduce GC pressure:

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

#### Leveled Buffer Pools
**Location**: `lib/leveledbytebufferpool/pool.go`

Multiple pools for different size ranges (powers of 2) reduce memory waste:

```go
// pools[0] for 0-256 bytes
// pools[1] for 257-512 bytes
// pools[2] for 513-1024 bytes
// ...
var pools [10]sync.Pool

func Get(dataLen int) *ByteBuffer {
    id, _ := getPoolIDAndCapacity(dataLen)
    // Try current pool, then next larger if empty
    for i := 0; i < 2; i++ {
        if v := pools[id].Get(); v != nil {
            return v.(*ByteBuffer)
        }
        id++
    }
    return &ByteBuffer{B: make([]byte, 0, capacity)}
}
```

#### Slice Reuse with [:0]
**Location**: Throughout codebase

```go
// Reset slice without losing capacity
data = data[:0]

// Append reuses underlying array
data = append(data, newData...)
```

#### Resize Strategies
**Location**: `lib/bytesutil/bytesutil.go`

Three strategies for different use cases:

| Strategy | Use Case |
|----------|----------|
| `ResizeWithOverallocate` | Grow by powers of 2 - reduces future allocations |
| `ResizeNoOverallocate` | Exact allocation - saves memory |
| `ResizeNoCopy` | New buffer without copy - for full overwrite |

```go
func ResizeWithCopy(dst []byte, newSize int, strategy ResizeStrategy) []byte {
    if newSize <= cap(dst) {
        return dst[:newSize]
    }
    switch strategy {
    case ResizeWithOverallocate:
        // Grow by powers of 2
    case ResizeNoOverallocate:
        // Exact size
    case ResizeNoCopy:
        // Allocate new, don't copy
    }
}
```

### 2. Concurrency

#### Sharded Locks
**Location**: `lib/lrucache/lrucache.go`, `lib/blockcache/blockcache.go`

Split large data structures into independent shards to reduce lock contention:

```go
type Cache struct {
    shards []*cacheShard  // Each shard has its own lock
}

func NewCache() *Cache {
    // Shard count = CPU cores × multiplier (capped at 16x)
    shardsCount := cgroup.AvailableCPUs() * multiplier
    if shardsCount > 16 * cgroup.AvailableCPUs() {
        shardsCount = 16 * cgroup.AvailableCPUs()
    }
    // ...
}

func (c *Cache) Get(key string) Value {
    idx := hash(key) % uint64(len(c.shards))
    return c.shards[idx].Get(key)
}
```

#### Atomic Operations
**Location**: `lib/bloomfilter/filter.go`, `lib/atomicutil/`

Use `atomic` instead of mutexes for simple operations - 10-100x faster:

```go
// CAS pattern for bit setting in Bloom filter
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
**Location**: `lib/atomicutil/`

Prevent false sharing between CPU cores:

```go
const CacheLineSize = 64

type Uint64 struct {
    _ [CacheLineSize - unsafe.Sizeof(atomic.Uint64{})%CacheLineSize]byte
    atomic.Uint64
    _ [CacheLineSize - unsafe.Sizeof(atomic.Uint64{})%CacheLineSize]byte
}
```

#### Local Worker Pools
**Location**: `app/vmselect/netstorage/netstorage.go`

Each worker has its own channel - reduces cross-CPU cache invalidation:

```go
// Each worker gets its own channel
workChs := make([]chan *Task, workers)
for i := range workChs {
    workChs[i] = make(chan *Task, itemsPerWorker)
}

// Workers process their local channel first
for i := range workChs {
    go func(workerID int) {
        for task := range workChs[workerID] {
            process(task)
        }
    }(i)
}
```

#### Channel Semaphores
**Location**: `lib/mergeset/table.go`, `lib/writeconcurrencylimiter/`

Buffered channel as concurrency limiter:

```go
var concurrencyLimitCh = make(chan struct{}, maxConcurrent)

func Acquire() error {
    select {
    case concurrencyLimitCh <- struct{}{}:
        return nil
    default:
        // Channel full - wait or return error
    }
}

func Release() {
    <-concurrencyLimitCh
}
```

### 3. Logging

#### Rate-Limited Logging
**Location**: `lib/logger/logger.go`

Prevent "log storms" from overwhelming disk I/O:

```go
type logLimiter struct {
    mu       sync.Mutex
    counters map[string]uint64  // location -> count
}

func (ll *logLimiter) needSuppress(location string, limit uint64) (suppress bool, msg string) {
    if limit == 0 {
        return false, ""
    }
    ll.mu.Lock()
    defer ll.mu.Unlock()

    count := ll.counters[location]
    if count >= limit {
        if count == limit {
            ll.counters[location]++
            return false, "suppressing log message with rate limit..."
        }
        return true, ""  // Suppress
    }
    ll.counters[location]++
    return false, ""
}
```

#### Log Throttler
**Location**: `lib/logger/throttler.go`

Channel-based throttle with dropped counter:

```go
type LogThrottler struct {
    ch      chan struct{}  // Size 1 = one message per duration
    dropped atomic.Uint64
}

func (lt *LogThrottler) Errorf(msg string, args ...any) {
    select {
    case lt.ch <- struct{}{}:
        dropped := lt.dropped.Swap(0)
        if dropped > 0 {
            msg += fmt.Sprintf(" (%d similar messages suppressed)", dropped)
        }
        logError(msg)
    default:
        lt.dropped.Add(1)
    }
}
```

### 4. Configuration

#### Environment Variable Flags
**Location**: `lib/envflag/envflag.go`

Command-line flags override environment variables:

```go
// Enable with -envflag.enable
// Set prefix with -envflag.prefix=MY_APP

// Flag "listen-addr" becomes environment variable:
// - LISTEN_ADDR (no prefix)
// - MY_APP_LISTEN_ADDR (with prefix)

func ParseFlagSet(fs *flag.FlagSet, args []string) error {
    fs.Parse(args)

    // Track explicitly set flags
    setFlags := make(map[string]bool)
    fs.Visit(func(f *flag.Flag) {
        setFlags[f.Name] = true
    })

    // Fill unset flags from environment
    fs.VisitAll(func(f *flag.Flag) {
        if setFlags[f.Name] {
            return  // Command-line has priority
        }
        envName := getEnvFlagName(prefix, f.Name)
        if v, ok := os.LookupEnv(envName); ok {
            fs.Set(f.Name, v)
        }
    })
}
```

### 5. Data Structures

#### Chunked Buffer
**Location**: `lib/chunkedbuffer/buffer.go`

Large data in fixed-size chunks with efficient pooling:

```go
const chunkSize = 64 * 1024  // 64KB chunks

type Buffer struct {
    chunks     *[chunkSize]byte
    chunksPool sync.Pool
}
```

#### Working Set Cache
**Location**: `lib/workingsetcache/cache.go`

Dual-cache with automatic rotation:

```go
type Cache struct {
    curr atomic.Pointer[fastcache.Cache]
    prev atomic.Pointer[fastcache.Cache]
    mode atomic.Uint32  // modeSplit, modeSwitching, modeWhole
}

// Three modes:
// 1. modeSplit: Use both curr and prev (50/50 split)
// 2. modeSwitching: Transitioning to single cache
// 3. modeWhole: Use only curr (full size)
```

#### Persistent Queue
**Location**: `lib/persistentqueue/`

In-memory channel with file fallback:

```go
type FastQueue struct {
    mu         sync.Mutex
    cond       sync.Cond
    ch         chan *ByteBuffer  // In-memory queue
    file       *mmap.File        // Fallback when channel full
}
```

### 6. Hot Path Optimizations

#### Unsafe Zero-Copy
**Location**: `lib/bytesutil/bytesutil.go`

```go
func ToUnsafeString(b []byte) string {
    return unsafe.String(unsafe.SliceData(b), len(b))
}

func ToUnsafeBytes(s string) []byte {
    return unsafe.Slice(unsafe.StringData(s), len(s))
}
```

**Warning**: Must ensure source data remains valid and unchanged.

#### Custom Encoding
**Location**: `lib/encoding/`

"Nearest Delta" encoding for time series - encode differences, not absolute values:

```go
// Encodes: store difference from previous value
// Also removes trailing zeros from floating point
```

#### Fast Time
**Location**: `lib/fasttime/fasttime_normal.go`

Update timestamp once per second - atomic load is faster than time.Now():

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

## Directory Structure

```
VictoriaMetrics/
├── lib/                      # Reusable libraries (66+ packages)
│   ├── bytesutil/            # Byte buffer utilities and pools
│   ├── slicesutil/           # Slice manipulation helpers
│   ├── stringsutil/          # String utilities
│   ├── encoding/             # Custom encoding for time series
│   ├── fasttime/             # Fast timestamp access
│   ├── timeutil/             # Time utilities
│   ├── atomicutil/           # Cache-line padded atomics
│   ├── syncwg/               # Safe WaitGroup wrapper
│   ├── timerpool/            # Timer pooling
│   ├── writeconcurrencylimiter/  # Concurrent request limiting
│   ├── ratelimiter/          # Per-second rate limiting
│   ├── lrucache/             # LRU cache with sharding
│   ├── blockcache/           # Block cache with sharding
│   ├── workingsetcache/      # Working set cache (dual-mode)
│   ├── leveledbytebufferpool/   # Multi-level buffer pool
│   ├── persistentqueue/      # Persistent and in-memory queue
│   ├── chunkedbuffer/        # Chunked buffer for large data
│   ├── consistenthash/       # Consistent hashing
│   ├── bloomfilter/          # Bloom filter
│   ├── flagutil/             # Flag utilities
│   ├── envflag/              # Environment variable flag support
│   └── logger/               # Structured logging with rate limiting
├── app/                      # Application binaries
│   ├── vmstorage/            # Storage engine
│   ├── vmselect/             # Query engine
│   └── vminsert/             # Ingestion endpoint
└── docs/                     # Documentation
```

## When to Use This Skill

Invoke this skill when you need help with:

1. **Memory optimization** - Reducing allocations, reusing buffers
2. **Concurrency patterns** - Sharding, atomics, worker pools
3. **Performance critical code** - Hot path optimization techniques
4. **Zero-allocation patterns** - Avoiding GC pressure
5. **Lock contention reduction** - Sharding strategies
6. **Rate limiting** - Preventing resource exhaustion
7. **Go best practices** - Production-tested patterns

## Quick Reference

| Pattern | Location | Use Case |
|---------|----------|----------|
| sync.Pool | `lib/bytesutil/` | Object reuse |
| Leveled pools | `lib/leveledbytebufferpool/` | Size-specific buffers |
| [:0] reset | Throughout | Slice reuse |
| Sharded locks | `lib/lrucache/` | Reduce contention |
| Atomics | `lib/atomicutil/`, `lib/bloomfilter/` | Simple counters |
| Cache-line padding | `lib/atomicutil/` | Prevent false sharing |
| Worker pools | `app/vmselect/` | Multi-core efficiency |
| Channel semaphores | `lib/mergeset/` | Concurrency limiting |
| Rate-limited logging | `lib/logger/` | Prevent log storms |
| Unsafe zero-copy | `lib/bytesutil/` | Hot path optimization |

## References

- GitHub: https://github.com/VictoriaMetrics/VictoriaMetrics
- Documentation: https://docs.victoriametrics.com/
- Article: [从入门到极致：VictoriaMetrics 教你写出最高效的 Go 代码](https://mp.weixin.qq.com/s/1svEokrz5C0FwBEA88YGqw)
