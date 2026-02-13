# AI-Native Core Database (ANC-DB) Detailed Specification v1.0

**Last Updated**: 2026-02-13
**Status**: Draft - Implementation Ready
**Target**: AI Agent Internal State Management System

---

## Executive Summary

AI-Native Core Database (ANC-DB) is a next-generation storage engine that completely eliminates the SQL "human language parsing layer." It allows AI agents to operate data directly from programs or via binary communication with minimal token usage and maximum reliability.

### Core Design Philosophy

1. **Zero SQL Overhead**: Zero parsing cost, enabling response times close to AI thinking speed.
2. **Token Minimization**: AI operations can be described with minimal token counts.
3. **Memory Safety**: Memory safety and crash-free guarantees provided by Rust.
4. **Lightweight Footprint**: Smooth operation even in mobile environments (e.g., LG gram).

### Use Cases

- AI Agent RAG (Retrieval-Augmented Generation) data store
- Persistence of conversation history and session states
- Shared memory storage between agents
- Cache layer for reasoning results

---

## 1. System Architecture

### 1.1 Three-Layer Design

```
┌─────────────────────────────────────────────────┐
│  AI Binary Protocol Layer (Nervous System)      │
│  - MessagePack/Protobuf over stdio/TCP          │
│  - Zero parsing overhead                        │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│  Rust Safety Layer (Skeletal System)            │
│  - Memory-safe wrapper with Arc/Mutex           │
│  - Schema validation & type safety              │
│  - Concurrent transaction manager               │
└────────────────┬────────────────────────────────┘
                 │ FFI
┌────────────────▼────────────────────────────────┐
│  SQLite Core Engine (Muscular System)           │
│  - B-Tree (sans SQL parser)                     │
│  - Pager (ACID transactions)                     │
│  - VFS (cross-platform I/O)                     │
└─────────────────────────────────────────────────┘
```

### 1.2 Component Responsibilities

| Layer | Primary Responsibility | Implementation Technology | Estimated Memory Usage |
|---------|---------|---------|----------------|
| **Protocol** | AI Communication, Command Dispatch | MessagePack | ~50KB |
| **Safety** | Concurrency Control, Schema Validation | Rust (FFI safety) | ~200KB |
| **Storage** | Data Persistence, B-Tree Operations | SQLite core (C) | ~500KB base |

---

## 2. Data Model Design

### 2.1 Schema Definition (Rust)

```rust
// Schema is defined by the Rust type system
#[derive(Schema, Serialize, Deserialize)]
struct AgentMemory {
    #[primary_key]
    id: u64,

    #[indexed]
    timestamp: i64,

    agent_id: String,
    conversation_id: String,

    #[compressed]  // Automatic LZ4 compression
    content: Vec<u8>,

    embedding: Option<Vec<f32>>,  // 768 dimensions assumed

    #[indexed]
    tags: Vec<String>,
}

// Generated at compile time:
// - B-Tree key layout
// - Index structures
// - Serialization/deserialization code
```

### 2.2 Primary Key Strategy

**Default**: Snowflake ID (64bit)
- 41bit: Timestamp (milliseconds)
- 10bit: Machine ID (multi-instance support)
- 12bit: Sequence (4096 items can be generated in the same millisecond)

**Custom Primary Key**: User-defined possible via `#[primary_key]` attribute.

### 2.3 Index Strategy

```rust
// Auto-generated secondary indexes
enum IndexType {
    BTree,      // Range search
    Hash,       // Exact match
    FullText,   // Text search (optional)
}

// Composite index definition
#[composite_index(fields = ["agent_id", "timestamp"])]
struct AgentMemory { /* ... */ }
```

---

## 3. AI Binary Protocol Specification

### 3.1 Protocol Design Principles

1. **Token Efficiency**: Command names are 1-2 byte fixed values.
2. **Type Safety**: Runtime validation via Schema ID.
3. **Stream Processing**: Large data transferred in chunks.
4. **Error Codes**: Numerical codes only (string messages in a separate field).

### 3.2 Command Format (MessagePack)

```
[Command Byte][Schema ID][Payload]
```

#### 3.2.1 Basic Command List

| Command | Byte | Description | Token Reduction Effect |
|---------|------|------|------------------|
| `DirectRead` | 0x01 | Direct retrieval by primary key | -95% vs SQL |
| `RangeScan` | 0x02 | Range search | -80% vs SQL |
| `AtomicWrite` | 0x03 | Single record write | -90% vs SQL |
| `BatchWrite` | 0x04 | Batch insertion (max 1000) | -98% vs SQL |
| `AtomicUpdate` | 0x05 | Update (Delta format) | -85% vs SQL |
| `Delete` | 0x06 | Deletion | -90% vs SQL |
| `BeginTx` | 0x10 | Start transaction | - |
| `CommitTx` | 0x11 | Commit | - |
| `RollbackTx` | 0x12 | Rollback | - |

#### 3.2.2 Command Detail Examples

**DirectRead Command**

```python
# AI side (Python client example)
import msgpack

command = msgpack.packb({
    "cmd": 0x01,
    "schema": "AgentMemory",
    "key": 1234567890123456789
})

# Token usage: practically zero (due to structured data)
# SQL comparison: "SELECT * FROM agent_memory WHERE id = 1234567890123456789"
#          → Saves about 15 tokens
```

**RangeScan Command**

```python
# Search in time range
command = msgpack.packb({
    "cmd": 0x02,
    "schema": "AgentMemory",
    "index": "timestamp",
    "range": {
        "start": 1704067200000,  # 2024-01-01 00:00:00 UTC
        "end": 1735689599999,    # 2024-12-31 23:59:59 UTC
    },
    "limit": 100,
    "order": "desc"
})
```

**BatchWrite Command (Most Important)**

```python
# AI saves multiple memories collectively
records = [
    {"agent_id": "gpt4", "content": b"..."},
    {"agent_id": "claude", "content": b"..."},
    # ... up to 1000 items
]

command = msgpack.packb({
    "cmd": 0x04,
    "schema": "AgentMemory",
    "records": records,
    "options": {
        "on_conflict": "replace",  # or "ignore", "fail"
    }
})

# SQL comparison:
# INSERT INTO ... VALUES (...), (...), ...
# → Massive token consumption + parsing time
# MessagePack: Direct binary transmission, parser-less
```

---

## 4. Rust Safety Layer Design

### 4.1 FFI Bridge

```rust
// Safe wrapping of SQLite core functions
mod ffi {
    use std::ffi::c_void;

    // Functions extracted only from sqlite3amalgamation.c
    extern "C" {
        fn sqlite3BtreeOpen(
            vfs: *const c_void,
            filename: *const u8,
            flags: u32,
            bt: *mut *mut Btree,
        ) -> i32;

        /* ... other FFI functions ... */
    }

    type Btree = c_void;
    type BtCursor = c_void;
}
```

---

## 5. SQLite Core Extraction Strategy

### 5.1 Targeted Modules

```c
// Group of minimum functions extracted from sqlite3.c (amalgamation)
// File size: ~8MB → ~500KB (94% reduction)

// Required Modules:
// 1. B-Tree (btree.c)
// 2. Pager (pager.c)
// 3. VFS (os_unix.c / os_win.c)
// 4. Utilities

// Targets for Removal (Unnecessary):
// - parse.y (Parser) - ~2MB
// - tokenize.c (Tokenizer)
// - prepare.c (Prepared Statements)
// - expr.c (Expression Evaluator)
// - select.c (SELECT statement processing)
// - where.c (WHERE clause optimization)
// - vdbe.c (Virtual Machine) - 1.5MB
```

---

## 6. Performance Optimization

### 6.1 Benchmark Goals

| Operation | Standard SQL | ANC-DB Goal | Improvement |
|-----|---------|-----------|--------|
| DirectRead | 50μs | 5μs | **10x** |
| RangeScan (100) | 500μs | 50μs | **10x** |
| BatchWrite (1000) | 50ms | 5ms | **10x** |
| Token Count (Typical INSERT) | 15 tokens | 0 tokens | **∞** |

---

## Appendix A: Quantitative Analysis of Token Reduction

### A.1 Comparison of Typical Operations

**Scenario**: AI saves 100 conversation logs.

**Standard SQL**:
- **Token Count**: ~1,500 tokens
- **Latency**: 50ms (including parsing time)

**ANC-DB**:
- **Token Count**: Practically zero (MessagePack binary)
- **Latency**: 5ms

**Reduction Effect**: **99.7% Reduction** + **10x Speed Improvement**
