# AI-Native Core Database (ANC-DB) 詳細仕様書 v1.0

**最終更新**: 2026-02-13
**ステータス**: Draft - Implementation Ready
**対象**: AIエージェント内部状態管理システム

---

## Executive Summary

AI-Native Core Database (ANC-DB) は、SQLという「人間向け言語解析層」を完全に排除し、AIエージェントがプログラムから直接、またはバイナリ通信を通じて、最小のトークン量と最高の確実性でデータを操作できる次世代ストレージエンジンです。

### 核心的な設計哲学

1. **Zero SQL Overhead**: 構文解析コストをゼロにし、AIの思考速度に近いレスポンスを実現
2. **Token Minimization**: AI操作を最小限のトークン数で記述可能
3. **Memory Safety**: Rustによるメモリ安全性とクラッシュフリー保証
4. **Lightweight Footprint**: モバイル環境（LG gram等）でも快適に動作

### ユースケース

- AIエージェントのRAG（Retrieval-Augmented Generation）データストア
- 会話履歴とセッション状態の永続化
- エージェント間の共有メモリストレージ
- 推論結果のキャッシュ層

---

## 1. システムアーキテクチャ

### 1.1 三層構造設計

```
┌─────────────────────────────────────────────────┐
│  AI Binary Protocol Layer (神経系)              │
│  - MessagePack/Protobuf over stdio/TCP          │
│  - Zero parsing overhead                        │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│  Rust Safety Layer (骨格系)                     │
│  - Memory-safe wrapper with Arc/Mutex           │
│  - Schema validation & type safety              │
│  - Concurrent transaction manager               │
└────────────────┬────────────────────────────────┘
                 │ FFI
┌────────────────▼────────────────────────────────┐
│  SQLite Core Engine (筋肉系)                    │
│  - B-Tree (sans SQL parser)                     │
│  - Pager (ACID transactions)                    │
│  - VFS (cross-platform I/O)                     │
└─────────────────────────────────────────────────┘
```

### 1.2 コンポーネント責任分担

| レイヤー | 主要責任 | 実装技術 | メモリ使用量目安 |
|---------|---------|---------|----------------|
| **Protocol** | AI通信、コマンドディスパッチ | MessagePack | ~50KB |
| **Safety** | 並行制御、スキーマ検証 | Rust (unsafe FFI封じ込め) | ~200KB |
| **Storage** | データ永続化、B-Tree操作 | SQLite core (C) | ~500KB base |

---

## 2. データモデル設計

### 2.1 スキーマ定義（Rust側）

```rust
// スキーマはRustの型システムで定義
#[derive(Schema, Serialize, Deserialize)]
struct AgentMemory {
    #[primary_key]
    id: u64,

    #[indexed]
    timestamp: i64,

    agent_id: String,
    conversation_id: String,

    #[compressed]  // 自動LZ4圧縮
    content: Vec<u8>,

    embedding: Option<Vec<f32>>,  // 768次元想定

    #[indexed]
    tags: Vec<String>,
}

// コンパイル時に以下を生成:
// - B-Tree key layout
// - Index structures
// - Serialization/deserialization code
```

### 2.2 主キー戦略

**デフォルト**: Snowflake ID (64bit)
- 41bit: Timestamp (ミリ秒)
- 10bit: Machine ID (複数インスタンス対応)
- 12bit: Sequence (同一ミリ秒内で4096個生成可能)

**カスタム主キー**: ユーザー定義も可能（`#[primary_key]`属性）

### 2.3 インデックス戦略

```rust
// 自動生成される二次インデックス
enum IndexType {
    BTree,      // 範囲検索
    Hash,       // 完全一致
    FullText,   // テキスト検索（オプション）
}

// 複合インデックスの定義
#[composite_index(fields = ["agent_id", "timestamp"])]
struct AgentMemory { /* ... */ }
```

---

## 3. AI Binary Protocol 仕様

### 3.1 プロトコル設計原則

1. **トークン効率性**: コマンド名は1-2バイトの固定値
2. **型安全性**: スキーマIDによる実行時検証
3. **ストリーム処理**: 大量データはチャンク転送
4. **エラーコード**: 数値コードのみ（文字列メッセージは別フィールド）

### 3.2 コマンドフォーマット（MessagePack）

```
[Command Byte][Schema ID][Payload]
```

#### 3.2.1 基本コマンド一覧

| コマンド | Byte | 説明 | トークン数削減効果 |
|---------|------|------|------------------|
| `DirectRead` | 0x01 | 主キーによる直接取得 | -95% vs SQL |
| `RangeScan` | 0x02 | 範囲検索 | -80% vs SQL |
| `AtomicWrite` | 0x03 | 単一レコード書き込み | -90% vs SQL |
| `BatchWrite` | 0x04 | バッチ挿入（最大1000件） | -98% vs SQL |
| `AtomicUpdate` | 0x05 | 更新（Delta形式） | -85% vs SQL |
| `Delete` | 0x06 | 削除 | -90% vs SQL |
| `BeginTx` | 0x10 | トランザクション開始 | - |
| `CommitTx` | 0x11 | コミット | - |
| `RollbackTx` | 0x12 | ロールバック | - |

#### 3.2.2 コマンド詳細例

**DirectRead コマンド**

```python
# AI側（Pythonクライアント例）
import msgpack

command = msgpack.packb({
    "cmd": 0x01,
    "schema": "AgentMemory",
    "key": 1234567890123456789
})

# トークン使用量: 実質ゼロ（構造化データのため）
# SQL比較: "SELECT * FROM agent_memory WHERE id = 1234567890123456789"
#          → 約15トークン削減
```

**RangeScan コマンド**

```python
# 時間範囲での検索
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

**BatchWrite コマンド（最重要）**

```python
# AIが複数のメモリを一括保存
records = [
    {"agent_id": "gpt4", "content": b"..."},
    {"agent_id": "claude", "content": b"..."},
    # ... 最大1000件
]

command = msgpack.packb({
    "cmd": 0x04,
    "schema": "AgentMemory",
    "records": records,
    "options": {
        "on_conflict": "replace",  # or "ignore", "fail"
    }
})

# SQL比較:
# INSERT INTO ... VALUES (...), (...), ...
# → 大量のトークン消費 + パース時間
# MessagePack: バイナリ直接送信、パースレス
```

### 3.3 レスポンスフォーマット

```python
{
    "status": 0,  # 0=success, >0=error code
    "data": [...],  # 取得結果（MessagePackバイナリ）
    "meta": {
        "rows_affected": 100,
        "execution_time_us": 245,  # マイクロ秒
    },
    "error": None  # エラー時のみ人間可読メッセージ
}
```

### 3.4 エラーコード体系

| コード | 意味 | AI対処方法 |
|-------|------|-----------|
| 0 | Success | - |
| 1 | InvalidCommand | コマンドバイト確認 |
| 2 | SchemaNotFound | スキーマID確認 |
| 3 | KeyNotFound | 存在チェック |
| 4 | ConstraintViolation | 一意性制約違反 |
| 5 | TransactionConflict | リトライ |
| 100+ | Internal Error | バグ報告 |

---

## 4. Rust Safety Layer 設計

### 4.1 FFIブリッジ

```rust
// SQLiteコア関数の安全なラッピング
mod ffi {
    use std::ffi::c_void;

    // sqlite3amalgamation.c から抽出した関数のみ
    extern "C" {
        fn sqlite3BtreeOpen(
            vfs: *const c_void,
            filename: *const u8,
            flags: u32,
            bt: *mut *mut Btree,
        ) -> i32;

        fn sqlite3BtreeBeginTrans(bt: *mut Btree, wrflag: i32) -> i32;
        fn sqlite3BtreeCommit(bt: *mut Btree) -> i32;
        fn sqlite3BtreeRollback(bt: *mut Btree) -> i32;

        fn sqlite3BtreeCursor(
            bt: *mut Btree,
            root_page: u32,
            wrflag: i32,
            cursor: *mut *mut BtCursor,
        ) -> i32;

        fn sqlite3BtreeMovetoUnpacked(
            cursor: *mut BtCursor,
            key: *const UnpackedRecord,
            res: *mut i32,
        ) -> i32;

        fn sqlite3BtreeData(
            cursor: *mut BtCursor,
            offset: u32,
            amt: u32,
            buf: *mut u8,
        ) -> i32;

        fn sqlite3BtreeInsert(
            cursor: *mut BtCursor,
            key: *const UnpackedRecord,
            data: *const u8,
            data_len: u32,
        ) -> i32;

        fn sqlite3BtreeDelete(cursor: *mut BtCursor) -> i32;
    }

    // 不透明型
    type Btree = c_void;
    type BtCursor = c_void;
    type UnpackedRecord = c_void;
}

// 安全なラッパー
pub struct Database {
    inner: Arc<Mutex<DatabaseInner>>,
}

struct DatabaseInner {
    btree: NonNull<ffi::Btree>,
    schemas: HashMap<String, Schema>,
}

impl Database {
    pub fn open(path: &Path) -> Result<Self, Error> {
        // FFI呼び出しをunsafeブロックで封じ込め
        unsafe {
            let mut bt = std::ptr::null_mut();
            let rc = ffi::sqlite3BtreeOpen(
                /* ... */
            );

            if rc != 0 {
                return Err(Error::OpenFailed(rc));
            }

            Ok(Self {
                inner: Arc::new(Mutex::new(DatabaseInner {
                    btree: NonNull::new(bt).unwrap(),
                    schemas: HashMap::new(),
                }))
            })
        }
    }
}
```

### 4.2 並行制御戦略

**選択肢検討**:

| 方式 | メリット | デメリット | 採用判断 |
|-----|---------|----------|---------|
| Mutex | シンプル | 書き込みで読み取りもブロック | ❌ |
| RwLock | 読み取り並行可 | Writer starvation | ✅ 採用 |
| Lock-free | 最高速 | 実装複雑 | 🔺 将来検討 |

**実装方針**:

```rust
use parking_lot::RwLock;  // std::sync::RwLockより高速

pub struct Database {
    inner: Arc<RwLock<DatabaseInner>>,
}

impl Database {
    // 読み取りトランザクション（並行実行可）
    pub fn read_tx<F, R>(&self, f: F) -> Result<R, Error>
    where
        F: FnOnce(&Transaction) -> Result<R, Error>,
    {
        let guard = self.inner.read();
        let tx = Transaction::begin_read(&guard)?;
        f(&tx)
    }

    // 書き込みトランザクション（排他的）
    pub fn write_tx<F, R>(&self, f: F) -> Result<R, Error>
    where
        F: FnOnce(&mut Transaction) -> Result<R, Error>,
    {
        let mut guard = self.inner.write();
        let mut tx = Transaction::begin_write(&mut guard)?;
        let result = f(&mut tx)?;
        tx.commit()?;
        Ok(result)
    }
}
```

### 4.3 トランザクション管理

```rust
pub enum IsolationLevel {
    ReadUncommitted,  // WALモード時のみ
    ReadCommitted,    // デフォルト（SQLite互換）
    Serializable,     // 完全直列化
}

pub struct Transaction<'db> {
    db: &'db DatabaseInner,
    level: IsolationLevel,
    is_write: bool,
    savepoints: Vec<Savepoint>,
}

impl<'db> Transaction<'db> {
    // ネストしたトランザクション（Savepoint）
    pub fn savepoint(&mut self, name: &str) -> Result<(), Error> {
        self.savepoints.push(Savepoint::new(name)?);
        Ok(())
    }

    pub fn rollback_to(&mut self, name: &str) -> Result<(), Error> {
        // Savepoint までロールバック
    }
}
```

---

## 5. SQLiteコア抽出戦略

### 5.1 抽出対象モジュール

```c
// sqlite3.c (amalgamation) から抽出する最小関数群
// ファイルサイズ: 約8MB → 約500KB（94%削減）

// 必須モジュール:
// 1. B-Tree (btree.c)
sqlite3BtreeOpen()
sqlite3BtreeClose()
sqlite3BtreeBeginTrans()
sqlite3BtreeCommit()
sqlite3BtreeRollback()
sqlite3BtreeCursor()
sqlite3BtreeMovetoUnpacked()
sqlite3BtreeData()
sqlite3BtreeInsert()
sqlite3BtreeDelete()
sqlite3BtreeCreateTable()

// 2. Pager (pager.c)
sqlite3PagerOpen()
sqlite3PagerClose()
sqlite3PagerGet()
sqlite3PagerWrite()
sqlite3PagerCommitPhaseOne()
sqlite3PagerCommitPhaseTwo()
sqlite3PagerRollback()

// 3. VFS (os_unix.c / os_win.c)
sqlite3OsOpen()
sqlite3OsClose()
sqlite3OsRead()
sqlite3OsWrite()
sqlite3OsSync()

// 4. Utilities
sqlite3_malloc()
sqlite3_free()
// ... etc

// 削除対象（不要）:
// - parse.y (パーサー) - 約2MB
// - tokenize.c (トークナイザー)
// - prepare.c (プリペアドステートメント)
// - expr.c (式評価器)
// - select.c (SELECT文処理)
// - where.c (WHERE句最適化)
// - vdbe.c (仮想マシン) - 1.5MB
```

### 5.2 依存関係解析手順

```bash
# Phase 1: nm コマンドで未解決シンボルを列挙
nm -u libsqlite3_minimal.a | grep -v ' U _sqlite3' > undefined_symbols.txt

# Phase 2: 再帰的に依存関数を追加
# (AIエージェントに実行させる)
python3 extract_dependencies.py sqlite3.c btree.c pager.c os.c > minimal_core.c

# Phase 3: コンパイルテスト
gcc -c minimal_core.c -o minimal_core.o
# → 未解決シンボルがなくなるまで繰り返し
```

### 5.3 コンパイルフラグ

```makefile
CFLAGS = -DSQLITE_OMIT_AUTHORIZATION \
         -DSQLITE_OMIT_AUTOINIT \
         -DSQLITE_OMIT_COMPLETE \
         -DSQLITE_OMIT_DEPRECATED \
         -DSQLITE_OMIT_EXPLAIN \
         -DSQLITE_OMIT_LOAD_EXTENSION \
         -DSQLITE_OMIT_PROGRESS_CALLBACK \
         -DSQLITE_OMIT_UTF16 \
         -DSQLITE_OMIT_VIRTUALTABLE \
         -DSQLITE_OMIT_WINDOWFUNC \
         -O3 -march=native
```

---

## 6. パフォーマンス最適化

### 6.1 ベンチマーク目標

| 操作 | 従来SQL | ANC-DB目標 | 改善率 |
|-----|---------|-----------|--------|
| DirectRead | 50μs | 5μs | **10倍** |
| RangeScan (100件) | 500μs | 50μs | **10倍** |
| BatchWrite (1000件) | 50ms | 5ms | **10倍** |
| トークン数 (典型的INSERT) | 15 tokens | 0 tokens | **∞** |

### 6.2 最適化技法

#### 6.2.1 メモリプール

```rust
// スレッドローカルなメモリプール
thread_local! {
    static MEM_POOL: RefCell<MemoryPool> = RefCell::new(
        MemoryPool::new(1024 * 1024)  // 1MB
    );
}

// レコードの一時バッファをプールから取得
impl Database {
    fn read_record(&self, key: u64) -> Result<Vec<u8>, Error> {
        MEM_POOL.with(|pool| {
            let buf = pool.borrow_mut().alloc(4096);
            // B-Treeから直接bufに読み込み
            unsafe {
                ffi::sqlite3BtreeData(cursor, 0, 4096, buf.as_mut_ptr());
            }
            Ok(buf.to_vec())
        })
    }
}
```

#### 6.2.2 バッチ処理の内部実装

```rust
pub fn batch_write(&mut self, records: Vec<Record>) -> Result<(), Error> {
    // ソート済みなら順次挿入（B-Treeの局所性向上）
    let sorted = self.should_sort(&records);
    if sorted {
        records.sort_by_key(|r| r.primary_key());
    }

    // トランザクションを一度だけ開始
    self.begin_write_tx()?;

    for record in records {
        // B-Treeへ直接挿入（パーサーなし）
        self.insert_direct(record)?;
    }

    self.commit_tx()?;
    Ok(())
}
```

#### 6.2.3 Embedding専用最適化

```rust
// f32配列の効率的な保存（量子化）
pub fn store_embedding(
    &mut self,
    id: u64,
    embedding: &[f32],  // 768次元
) -> Result<(), Error> {
    // 32bit float → 8bit量子化（96%圧縮）
    let quantized = quantize_i8(embedding);

    // B-Treeに直接書き込み（3KB → 768バイト）
    self.write_blob(id, &quantized)?;
    Ok(())
}

fn quantize_i8(vec: &[f32]) -> Vec<i8> {
    // Min-Max正規化 + 256レベル量子化
    let min = vec.iter().cloned().fold(f32::INFINITY, f32::min);
    let max = vec.iter().cloned().fold(f32::NEG_INFINITY, f32::max);
    let scale = 255.0 / (max - min);

    vec.iter()
        .map(|&v| ((v - min) * scale) as i8)
        .collect()
}
```

### 6.3 WALモード設定

```rust
// Write-Ahead Logging で読み取り/書き込み並行性向上
impl Database {
    pub fn enable_wal(&self) -> Result<(), Error> {
        unsafe {
            ffi::sqlite3_exec(
                self.inner.btree,
                b"PRAGMA journal_mode=WAL\0".as_ptr(),
                None,
                std::ptr::null_mut(),
                std::ptr::null_mut(),
            );
        }
        Ok(())
    }
}
```

---

## 7. 開発ロードマップ（AIエージェント指示付き）

### Phase 1: 依存関係解析（2-3日）

**AIエージェントへの指示**:
```
1. SQLiteソースコード (sqlite3.c) を読み込む
2. 以下の関数群から再帰的に依存関係をトレースし、
   SQLパーサー（parse.y, tokenize.c）を含まない最小関数セットを抽出:

   起点関数:
   - sqlite3BtreeOpen
   - sqlite3BtreeCursor
   - sqlite3BtreeInsert
   - sqlite3BtreeData
   - sqlite3BtreeDelete

3. 依存グラフをGraphviz形式で出力
4. 抽出された関数をminimal_core.c として生成
```

**成果物**:
- `minimal_core.c` (約500KB)
- `dependency_graph.dot`
- `excluded_functions.txt` (削除された関数リスト)

### Phase 2: Rustブリッジ構築（3-5日）

**AIエージェントへの指示**:
```
1. bindgen を使用して minimal_core.c のRustバインディング生成
2. 以下の安全なラッパーAPIを実装:

   pub struct Database { /* ... */ }
   impl Database {
       pub fn open(path: &Path) -> Result<Self, Error>;
       pub fn read(&self, key: u64) -> Result<Vec<u8>, Error>;
       pub fn write(&mut self, key: u64, data: &[u8]) -> Result<(), Error>;
       pub fn delete(&mut self, key: u64) -> Result<(), Error>;
   }

3. ユニットテストを自動生成（proptest使用）
```

**成果物**:
- `anc-db-core/` Rustクレート
- `tests/` テストスイート（カバレッジ80%以上）

### Phase 3: MessagePackプロトコル実装（2-3日）

**AIエージェントへの指示**:
```
1. rmp-serde を使用してプロトコルハンドラを実装
2. 仕様書セクション3.2のコマンドをすべて実装
3. Pythonクライアントライブラリを自動生成:

   import anc_db

   db = anc_db.connect("memory.db")
   db.write(key=123, data=b"hello")
   result = db.read(key=123)
```

**成果物**:
- `anc-db-protocol/` Rustクレート
- `anc-db-py/` Pythonバインディング（PyO3）

### Phase 4: スキーママクロ実装（3-5日）

**AIエージェントへの指示**:
```
1. Rustプロシージャルマクロで#[derive(Schema)]を実装
2. コンパイル時に以下を生成:
   - B-Tree key layout
   - Serializeラインタイム処理
   - Indexメタデータ

3. 以下のスキーマ定義が動作することを確認:

   #[derive(Schema)]
   struct User {
       #[primary_key]
       id: u64,
       #[indexed]
       email: String,
       created_at: i64,
   }
```

**成果物**:
- `anc-db-derive/` マクロクレート
- スキーマバリデーションテスト

### Phase 5: 統合テスト＆ベンチマーク（2-3日）

**AIエージェントへの指示**:
```
1. 以下のベンチマークを実装（criterion.rs使用）:
   - DirectRead: 10万回ループ
   - BatchWrite: 1000件×100回
   - RangeScan: 100-10000件

2. SQLite標準APIと比較し、性能改善率を測定
3. メモリリーク検査（Valgrind）
```

**成果物**:
- `benches/` ベンチマークスイート
- `benchmark_report.md`

---

## 8. セキュリティ考慮事項

### 8.1 メモリ安全性

- ✅ **Rustのunsafeブロックは最小限**（FFI境界のみ）
- ✅ **境界チェック**: 配列アクセスは常に検証
- ✅ **Use-After-Free防止**: Arc/RcによるGC的管理

### 8.2 暗号化（オプション機能）

```rust
// ディスク暗号化（Transparent Encryption）
#[cfg(feature = "encryption")]
pub fn open_encrypted(path: &Path, key: &[u8; 32]) -> Result<Database, Error> {
    // AES-256-GCM でページ単位暗号化
    // （SQLite Encryption Extensionの手法を踏襲）
}
```

### 8.3 インジェクション攻撃への耐性

- ✅ **SQL Injection**: 存在しない（SQLパーサーがない）
- ✅ **MessagePack Injection**: スキーマバリデーションで防御
- ✅ **型安全性**: Rustの型システムで保証

---

## 9. エラーハンドリング戦略

### 9.1 エラー型階層

```rust
#[derive(Debug, thiserror::Error)]
pub enum Error {
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),

    #[error("Corruption detected at page {page}")]
    Corruption { page: u32 },

    #[error("Key not found: {key}")]
    KeyNotFound { key: u64 },

    #[error("Transaction conflict")]
    TransactionConflict,

    #[error("Schema mismatch: expected {expected}, got {actual}")]
    SchemaMismatch { expected: String, actual: String },
}
```

### 9.2 リトライポリシー

```rust
// トランザクション競合時の自動リトライ
pub fn write_with_retry<F>(
    &self,
    f: F,
    max_retries: u32,
) -> Result<(), Error>
where
    F: Fn(&mut Transaction) -> Result<(), Error>,
{
    for attempt in 0..max_retries {
        match self.write_tx(|tx| f(tx)) {
            Ok(result) => return Ok(result),
            Err(Error::TransactionConflict) => {
                std::thread::sleep(Duration::from_millis(2_u64.pow(attempt)));
                continue;
            }
            Err(e) => return Err(e),
        }
    }
    Err(Error::MaxRetriesExceeded)
}
```

---

## 10. デプロイメント＆配布

### 10.1 ターゲットプラットフォーム

| OS | アーキテクチャ | バイナリサイズ目標 |
|----|---------------|------------------|
| Windows 10+ | x86_64 | < 2MB |
| macOS 11+ | ARM64 (M1/M2/M3) | < 1.5MB |
| Linux (Ubuntu 20.04+) | x86_64 | < 1.2MB |

### 10.2 配布形態

1. **Static Library**: `libanc_db.a` (C FFI)
2. **Dynamic Library**: `libanc_db.so/.dll/.dylib`
3. **Rustクレート**: `anc-db = "0.1.0"` (crates.io)
4. **Pythonパッケージ**: `pip install anc-db`
5. **Dockerイメージ**: `anc-db:latest` (Alpine based)

### 10.3 インストール例

```bash
# Rust
cargo add anc-db

# Python
pip install anc-db

# C/C++
git clone https://github.com/your-org/anc-db
cd anc-db && make install
```

---

## 11. 今すぐAIエージェントに指示すべきコマンド

### 最優先タスク

```
AIエージェントへ:

1. SQLiteソースコード (sqlite3.c, version 3.45.0) をダウンロード
   https://www.sqlite.org/2024/sqlite-amalgamation-3450000.zip

2. 以下のPythonスクリプトを実行し、B-Tree APIの依存関係を解析:

import re
import networkx as nx

def extract_function_calls(c_code):
    # 正規表現でsqlite3Btree*関数の呼び出しを抽出
    pattern = r'sqlite3[A-Z][a-zA-Z0-9_]*\s*\('
    return set(re.findall(pattern, c_code))

def build_dependency_graph(source_file):
    with open(source_file, 'r') as f:
        code = f.read()

    # 関数定義とその内部で呼び出される関数をマッピング
    # ... (実装)

    return dependency_graph

graph = build_dependency_graph('sqlite3.c')

# SQL関連関数を除外したサブグラフを抽出
minimal_funcs = nx.ancestors(graph, 'sqlite3BtreeOpen')
minimal_funcs -= nx.descendants(graph, 'sqlite3Prepare')  # SQL系を除外

print(f"最小関数セット: {len(minimal_funcs)}個")
print('\n'.join(sorted(minimal_funcs)))

3. 出力結果をminimal_functions.txtとして保存
```

---

## 12. 参考資料

### 12.1 技術文献

- [SQLite Architecture](https://www.sqlite.org/arch.html)
- [B-Tree Implementation Details](https://www.sqlite.org/btreemodule.html)
- [Rust FFI Guide](https://doc.rust-lang.org/nomicon/ffi.html)
- [MessagePack Specification](https://github.com/msgpack/msgpack/blob/master/spec.md)

### 12.2 既存プロジェクト参考

- **libsql**: SQLiteのRustフォーク（プロトコル部分参考）
- **sled**: Pure Rust埋め込みDB（並行制御参考）
- **redb**: Rustで書かれたLMDB互換DB（API設計参考）

---

## Appendix A: トークン削減効果の定量分析

### A.1 典型的な操作の比較

**シナリオ**: AIが100件の会話履歴を保存

**従来のSQL**:
```sql
INSERT INTO conversations (agent_id, timestamp, content)
VALUES
  ('gpt4', 1704067200, 'Hello...'),
  ('gpt4', 1704067201, 'How are you...'),
  -- ... 98行省略
  ('gpt4', 1704067299, 'Goodbye...');
```
- **トークン数**: 約1,500トークン
- **レイテンシ**: 50ms（パース時間含む）

**ANC-DB**:
```python
db.batch_write([
    {"agent_id": "gpt4", "timestamp": 1704067200, "content": b"Hello..."},
    # ... 99個
])
```
- **トークン数**: 実質ゼロ（MessagePackバイナリ）
- **レイテンシ**: 5ms

**削減効果**: **99.7%削減** + **10倍高速化**

---

## Appendix B: サンプルコード集

### B.1 完全な使用例（Rust）

```rust
use anc_db::prelude::*;

#[derive(Schema, Serialize, Deserialize)]
struct ChatMessage {
    #[primary_key]
    id: u64,

    #[indexed]
    user_id: String,

    timestamp: i64,
    content: String,
}

fn main() -> Result<(), Error> {
    // データベース開設
    let db = Database::open("chat.db")?;
    db.register_schema::<ChatMessage>()?;

    // 書き込み
    db.write_tx(|tx| {
        tx.insert(ChatMessage {
            id: 1,
            user_id: "alice".to_string(),
            timestamp: 1704067200,
            content: "Hello!".to_string(),
        })
    })?;

    // 読み取り
    let msg = db.read_tx(|tx| {
        tx.get::<ChatMessage>(1)
    })?;

    println!("{:?}", msg);
    Ok(())
}
```

### B.2 Pythonクライアント例

```python
from anc_db import Database, Schema

# スキーマ定義
class ChatMessage(Schema):
    id: int (primary_key=True)
    user_id: str (indexed=True)
    timestamp: int
    content: bytes

# 接続
db = Database.open("chat.db")

# 挿入
db.write(ChatMessage(
    id=1,
    user_id="alice",
    timestamp=1704067200,
    content=b"Hello!"
))

# 検索
messages = db.range_scan(
    schema=ChatMessage,
    index="user_id",
    start="alice",
    end="alice",
)

for msg in messages:
    print(msg.content.decode('utf-8'))
```

---

## 変更履歴

| バージョン | 日付 | 変更内容 |
|----------|------|---------|
| 1.0 | 2026-02-13 | 初版リリース |

---

**Document End**

次のステップ: 上記「11. 今すぐAIエージェントに指示すべきコマンド」を実行し、SQLiteソースコードの依存関係解析を開始してください。
