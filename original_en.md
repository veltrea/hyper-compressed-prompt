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

---

## 3. AI Binary Protocol Specification

### 3.1 Design Principles

1. **Token Efficiency**: Command names are 1-2 byte fixed values.
2. **Type Safety**: Runtime validation via Schema ID.

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

---

## 6. Performance Optimization

### 6.1 Benchmark Targets

| Operation | Standard SQL | ANC-DB Target | Improvement |
|-----|---------|-----------|--------|
| DirectRead | 50μs | 5μs | **10x** |
| BatchWrite (1k) | 50ms | 5ms | **10x** |

---

## CHECKSUM
- **SPEC_TOKENS**: 15,000 -> 850 (94.3% reduction)
- **STATUS**: READY_FOR_IMPL (Synthesis Successful)

**Document End**
