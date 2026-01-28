# lgh - High-Performance Go Programming Plugin

A Claude Code plugin for high-performance Go programming patterns and best practices, inspired by VictoriaMetrics.

## Overview

This plugin collects battle-tested patterns from VictoriaMetrics - one of the highest-performance Go applications. VictoriaMetrics achieves exceptional write speeds, low memory usage, and efficient CPU utilization through specific patterns and techniques.

## Skills

### go-high-performance-programming

Core skill providing patterns from VictoriaMetrics:

- **Memory Management** - sync.Pool, leveled buffers, slice reuse, resize strategies
- **Concurrency** - Sharded locks, atomic operations, cache-line padding, worker pools
- **Logging** - Rate-limited logging, throttlers
- **Configuration** - Environment variable flags
- **Hot Path Optimizations** - Unsafe zero-copy, custom encoding, fast time

Invoke with: `/lgh:go-high-performance-programming`

## Installation

The plugin is installed at:
```
~/.claude/plugins/marketplaces/lgh/
```

## Usage

```
/lgh:go-high-performance-programming
```

## Future Skills

Additional skills will be added to this plugin covering:
- More Go performance patterns
- Benchmarking techniques
- Profiling and optimization
- Additional high-performance Go libraries

## References

- VictoriaMetrics: https://github.com/VictoriaMetrics/VictoriaMetrics
- Documentation: https://docs.victoriametrics.com/

## License

Apache-2.0
