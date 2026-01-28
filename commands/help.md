---
description: Shows help and available commands for the lgh plugin
---

# lgh - High-Performance Go Programming

This plugin provides battle-tested patterns from VictoriaMetrics for writing highly efficient Go code.

## Available Commands

### `/lgh:go-high-performance-programming`
Invokes the high-performance Go programming skill for guidance on:
- Memory optimization (sync.Pool, buffer reuse, zero-allocation patterns)
- Concurrency patterns (sharded locks, atomics, worker pools)
- Performance best practices
- Hot path optimization techniques

## Quick Start

Simply ask Claude about Go performance topics, and the skill will be automatically invoked. For example:

- "How do I reduce allocations in this Go code?"
- "Show me sharded lock patterns for Go"
- "Help me optimize this hot path"

## Resources

- Based on: https://github.com/VictoriaMetrics/VictoriaMetrics
- Plugin repo: https://github.com/vjoke/lgh
