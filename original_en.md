# AI-Native Core Database (ANC-DB) Detailed Specification v1.0

> [!NOTE]
> **Technical Process Note**:
> As the lead developer is Japanese, the detailed technical specification was first drafted in Japanese (`original.md`).
> The token compression process involves a **direct conversion from the Japanese verbose specification to the low-level symbolic format** (the assembler-like tokens shown in `compress.md`), bypassing a natural language bridge like English.
> Synthesizing symbolic code directly from the native conceptual framework minimizes information loss and maximizes development speed through "Direct Synthesis."

**Last Updated**: 2026-02-13
**Status**: Draft - Implementation Ready
**Target**: AI Agent Internal State Management System

---

## Executive Summary

AI-Native Core Database (ANC-DB) is a next-generation storage engine that completely eliminates the human-centric "SQL language analysis layer." It allows AI agents to manipulate data directly or via binary communication with minimal token usage and maximum reliability.

### Core Design Philosophy

1. **Zero SQL Overhead**: Eliminate parsing costs to achieve response speeds close to AI thought processing.
2. **Token Minimization**: Describe AI operations with the minimum number of tokens.
3. **Memory Safety**: Guaranteed memory safety and crash-free operation via Rust.
4. **Lightweight Footprint**: Comfortable operation even in mobile environments (e.g., LG gram).

### Use Cases

- RAG (Retrieval-Augmented Generation) data stores for AI agents.
- Persistence of conversation history and session states.
- Shared memory storage between agents.
- Caching layer for inference results.

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
│  - Pager (ACID transactions)                    │
│  - VFS (cross-platform I/O)                     │
└─────────────────────────────────────────────────┘
```

### 1.2 Component Responsibilities

| Layer | Primary Responsibility | Implementation Technology | Est. Memory Usage |
|---------|---------|---------|----------------|
| **Protocol** | AI Communication, Command Dispatch | MessagePack | ~50KB |
| **Safety** | Concurrency Control, Schema Validation | Rust (FFI containment) | ~200KB |
| **Storage** | Data Persistence, B-Tree Ops | SQLite core (C) | ~500KB base |

---

## 2. Data Model Design

### 2.1 Schema Definition (Rust-side)

```rust
// Schemas are defined in the Rust type system
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

// Generates the following at compile time:
// - B-Tree key layout
// - Index structures
// - Serialization/deserialization code
```

### 2.2 Primary Key Strategy

**Default**: Snowflake ID (64bit)
- 41bit: Timestamp (ms)
- 10bit: Machine ID (multi-instance support)
- 12bit: Sequence (4096 per ms)

**Custom PK**: User-defined possible via `#[primary_key]` attribute.

### 2.3 Indexing Strategy

```rust
// Auto-generated secondary indexes
enum IndexType {
    BTree,      // Range search
    Hash,       // Exact match
    FullText,   // Text search (Optional)
}

// Composite index definition
#[composite_index(fields = ["agent_id", "timestamp"])]
struct AgentMemory { /* ... */ }
```

---

## 3. AI Binary Protocol Specification

### 3.1 Design Principles

1. **Token Efficiency**: Command names are 1-2 byte fixed values.
2. **Type Safety**: Runtime validation via Schema ID.
3. **Stream Processing**: Large data via chunk transfer.
4. **Error Codes**: Numeric codes only (human messages in separate fields).

### 3.2 Command Format (MessagePack)

```
[Command Byte][Schema ID][Payload]
```

#### 3.2.1 Basic Commands

| Command | Byte | Description | Token Reduction |
|---------|------|------|------------------|
| `DirectRead` | 0x01 | Direct fetch by PK | -95% vs SQL |
| `RangeScan` | 0x02 | Range search | -80% vs SQL |
| `AtomicWrite` | 0x03 | Single record write | -90% vs SQL |
| `BatchWrite` | 0x04 | Batch insert (max 1000) | -98% vs SQL |
| `AtomicUpdate` | 0x05 | Update (Delta format) | -85% vs SQL |
| `Delete` | 0x06 | Deletion | -90% vs SQL |
| `BeginTx` | 0x10 | Transaction Start | - |
| `CommitTx` | 0x11 | Commit | - |
| `RollbackTx` | 0x12 | Rollback | - |

#### 3.2.2 Command Examples

**DirectRead Command**

```python
# AI-side (Python example)
import msgpack

command = msgpack.packb({
    "cmd": 0x01,
    "schema": "AgentMemory",
    "key": 1234567890123456789
})

# Token usage: Virtually zero (structured data)
# SQL comparison: "SELECT * FROM agent_memory WHERE id = 1234567890123456789"
# → ~15 tokens saved
```

---

## 4. Rust Safety Layer Design

### 4.1 FFI Bridge

(Omitted technical Rust/C details for brevity in this report, focusing on the concepts)

### 4.2 Concurrency Strategy

- **Choice**: RwLock (via `parking_lot` crate).
- **Policy**: Parallel Reads, Exclusive Writes.

---

## 5. SQLite Core Extraction Strategy

### 5.1 Modules

- **Target**: Reduce 8MB `sqlite3.c` to ~500KB (94% reduction).
- **Keep**: B-Tree, Pager, VFS, Utils.
- **Drop**: SQL Parser, Tokenizer, Prepared Statements, Expr Evaluator, Select/Where Optimizer, VDBE.

---

## 6. Performance Optimization

### 6.1 Benchmark Targets

| Operation | Standard SQL | ANC-DB Target | Improvement |
|-----|---------|-----------|--------|
| DirectRead | 50μs | 5μs | **10x** |
| BatchWrite (1k) | 50ms | 5ms | **10x** |

---

## 7. Roadmap

1. **Phase 1**: Dependency Analysis (2-3 days).
2. **Phase 2**: Rust Bridge Construction (3-5 days).
3. **Phase 3**: MessagePack Protocol Implementation (2-3 days).
4. **Phase 4**: Schema Macro Implementation (3-5 days).
5. **Phase 5**: Integration Testing & Benchmarking (2-3 days).

---

## CHECKSUM
- **SPEC_TOKENS**: 15,000 -> 850 (94.3% reduction)
- **STATUS**: READY_FOR_IMPL (Synthesis Successful)

**Document End**
