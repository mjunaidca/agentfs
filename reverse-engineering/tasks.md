# AgentFS Implementation Tasks (Reverse Engineered)

**Version:** 0.1.2 (Current Implementation)
**Date:** 2025-11-18
**Estimated Total Time:** 8-12 weeks for full implementation
**Team Size:** 1-2 developers

---

## Prerequisites

Before starting implementation:

- [ ] Rust 2021 edition or later installed
- [ ] Node.js 16+ and npm installed
- [ ] SQLite 3.x available for testing
- [ ] Linux machine for sandbox development (or VM)
- [ ] Git and GitHub account
- [ ] Read `spec.md` and `plan.md` thoroughly

---

## Phase 1: Project Foundation & Core SDK (Rust)

**Duration:** 2-3 weeks
**Goal:** Create Rust SDK with filesystem, KV, and tool tracking

### Task 1.1: Project Structure Setup

**Time:** 1 day

- [ ] **1.1.1**: Create Cargo workspace root
  ```bash
  cargo new --bin agentfs
  mkdir -p sdk/rust cli sandbox
  ```

- [ ] **1.1.2**: Create workspace `Cargo.toml`
  ```toml
  [workspace]
  members = ["sdk/rust", "cli", "sandbox"]
  resolver = "2"
  ```

- [ ] **1.1.3**: Initialize SDK package
  ```bash
  cd sdk/rust
  cargo init --lib
  ```

- [ ] **1.1.4**: Add dependencies to `sdk/rust/Cargo.toml`
  ```toml
  [dependencies]
  turso = "0.3.2"
  tokio = { version = "1", features = ["full"] }
  async-trait = "0.1"
  serde = { version = "1.0", features = ["derive"] }
  serde_json = "1.0"
  libc = "0.2"
  anyhow = "1.0"

  [dev-dependencies]
  tempfile = "3"
  ```

- [ ] **1.1.5**: Create module structure
  ```bash
  touch src/lib.rs src/filesystem.rs src/kvstore.rs src/toolcalls.rs
  ```

- [ ] **1.1.6**: Setup basic lib.rs exports
  ```rust
  pub mod filesystem;
  pub mod kvstore;
  pub mod toolcalls;
  ```

**Validation**: `cargo build` succeeds

---

### Task 1.2: Database Schema Implementation

**Time:** 2 days

- [ ] **1.2.1**: Create schema SQL constants in `lib.rs`
  ```rust
  const CREATE_KV_STORE: &str = "CREATE TABLE IF NOT EXISTS kv_store (...)";
  const CREATE_FS_INODE: &str = "CREATE TABLE IF NOT EXISTS fs_inode (...)";
  const CREATE_FS_DENTRY: &str = "CREATE TABLE IF NOT EXISTS fs_dentry (...)";
  const CREATE_FS_DATA: &str = "CREATE TABLE IF NOT EXISTS fs_data (...)";
  const CREATE_FS_SYMLINK: &str = "CREATE TABLE IF NOT EXISTS fs_symlink (...)";
  const CREATE_TOOL_CALLS: &str = "CREATE TABLE IF NOT EXISTS tool_calls (...)";
  ```
  Reference: `/home/user/agentfs/SPEC.md` for exact schema

- [ ] **1.2.2**: Create `AgentFS` struct with initialization
  ```rust
  pub struct AgentFS {
      conn: Arc<Connection>,
      pub kv: KvStore,
      pub fs: Filesystem,
      pub tools: ToolCalls,
  }

  impl AgentFS {
      pub async fn new(db_path: &str) -> Result<Self> {
          // 1. Connect to database
          // 2. Initialize each component
          // 3. Return AgentFS
      }
  }
  ```

- [ ] **1.2.3**: Test schema creation
  ```rust
  #[tokio::test]
  async fn test_schema_creation() {
      let agentfs = AgentFS::new(":memory:").await.unwrap();
      // Verify tables exist
  }
  ```

**Validation**: Schema creates successfully in SQLite

---

### Task 1.3: Key-Value Store Implementation

**Time:** 2 days

**File**: `sdk/rust/src/kvstore.rs`

- [ ] **1.3.1**: Define `KvStore` struct
  ```rust
  pub struct KvStore {
      conn: Arc<Connection>,
  }

  impl KvStore {
      pub async fn from_connection(conn: Arc<Connection>) -> Result<Self> {
          let kv = Self { conn };
          kv.initialize().await?;
          Ok(kv)
      }
  }
  ```

- [ ] **1.3.2**: Implement `initialize()` - Create table
  ```rust
  async fn initialize(&self) -> Result<()> {
      self.conn.execute(CREATE_KV_STORE, ()).await?;
      Ok(())
  }
  ```

- [ ] **1.3.3**: Implement `set()`
  ```rust
  pub async fn set<T: Serialize>(&self, key: &str, value: &T) -> Result<()> {
      let json = serde_json::to_string(value)?;
      self.conn.execute(
          "INSERT INTO kv_store (key, value, updated_at) VALUES (?, ?, unixepoch())
           ON CONFLICT(key) DO UPDATE SET value = excluded.value, updated_at = unixepoch()",
          (key, json)
      ).await?;
      Ok(())
  }
  ```

- [ ] **1.3.4**: Implement `get()`
  ```rust
  pub async fn get<T: DeserializeOwned>(&self, key: &str) -> Result<Option<T>> {
      let result = self.conn.query(
          "SELECT value FROM kv_store WHERE key = ?",
          (key,)
      ).await?;
      // Deserialize JSON
  }
  ```

- [ ] **1.3.5**: Implement `delete()`
  ```rust
  pub async fn delete(&self, key: &str) -> Result<()> {
      self.conn.execute("DELETE FROM kv_store WHERE key = ?", (key,)).await?;
      Ok(())
  }
  ```

- [ ] **1.3.6**: Implement `keys()`
  ```rust
  pub async fn keys(&self) -> Result<Vec<String>> {
      let rows = self.conn.query("SELECT key FROM kv_store ORDER BY key", ()).await?;
      // Extract keys
  }
  ```

- [ ] **1.3.7**: Write comprehensive tests
  ```rust
  #[tokio::test]
  async fn test_kv_set_get() { ... }

  #[tokio::test]
  async fn test_kv_upsert() { ... }

  #[tokio::test]
  async fn test_kv_delete() { ... }

  #[tokio::test]
  async fn test_kv_keys() { ... }
  ```

**Validation**: All KV tests pass

---

### Task 1.4: Tool Calls Implementation

**Time:** 2 days

**File**: `sdk/rust/src/toolcalls.rs`

- [ ] **1.4.1**: Define data structures
  ```rust
  #[derive(Debug, Clone, Serialize, Deserialize)]
  pub enum ToolCallStatus {
      Success,
      Error,
  }

  #[derive(Debug, Clone)]
  pub struct ToolCall {
      pub id: i64,
      pub name: String,
      pub parameters: Option<Value>,
      pub result: Option<Value>,
      pub error: Option<String>,
      pub started_at: i64,
      pub completed_at: i64,
      pub duration_ms: i64,
      pub status: ToolCallStatus,
  }

  #[derive(Debug, Clone)]
  pub struct ToolCallStats {
      pub name: String,
      pub total_calls: i64,
      pub successful: i64,
      pub failed: i64,
      pub avg_duration_ms: f64,
  }
  ```

- [ ] **1.4.2**: Implement `ToolCalls` struct
  ```rust
  pub struct ToolCalls {
      conn: Arc<Connection>,
  }
  ```

- [ ] **1.4.3**: Implement `record()` for completed calls
  ```rust
  pub async fn record(
      &self,
      name: &str,
      started_at: i64,
      completed_at: i64,
      parameters: Option<Value>,
      result: Option<Value>,
      error: Option<String>,
  ) -> Result<i64> {
      let duration_ms = (completed_at - started_at) * 1000;
      // INSERT INTO tool_calls ...
  }
  ```

- [ ] **1.4.4**: Implement `start()` for in-progress tracking (optional)
  ```rust
  pub async fn start(&self, name: &str, parameters: Option<Value>) -> Result<i64> {
      // Store started_at, return ID for later completion
  }

  pub async fn success(&self, id: i64, result: Option<Value>) -> Result<()> {
      // Update with completed_at and result
  }

  pub async fn error(&self, id: i64, error: &str) -> Result<()> {
      // Update with completed_at and error
  }
  ```

- [ ] **1.4.5**: Implement query methods
  ```rust
  pub async fn get(&self, id: i64) -> Result<Option<ToolCall>> { ... }

  pub async fn recent(&self, since: i64, limit: Option<usize>) -> Result<Vec<ToolCall>> { ... }

  pub async fn stats_for(&self, name: &str) -> Result<Option<ToolCallStats>> { ... }

  pub async fn stats(&self) -> Result<Vec<ToolCallStats>> { ... }
  ```

- [ ] **1.4.6**: Write tests
  ```rust
  #[tokio::test]
  async fn test_record_successful_call() { ... }

  #[tokio::test]
  async fn test_record_failed_call() { ... }

  #[tokio::test]
  async fn test_query_by_name() { ... }

  #[tokio::test]
  async fn test_stats() { ... }
  ```

**Validation**: All tool call tests pass

---

### Task 1.5: Filesystem Implementation (Part 1: Core)

**Time:** 5 days

**File**: `sdk/rust/src/filesystem.rs`

- [ ] **1.5.1**: Define constants and types
  ```rust
  const S_IFMT: u32 = 0o170000;
  const S_IFREG: u32 = 0o100000;
  const S_IFDIR: u32 = 0o040000;
  const S_IFLNK: u32 = 0o120000;
  const DEFAULT_FILE_MODE: u32 = S_IFREG | 0o644;
  const DEFAULT_DIR_MODE: u32 = S_IFDIR | 0o755;
  const ROOT_INO: i64 = 1;

  #[derive(Debug, Clone)]
  pub struct Stats {
      pub ino: i64,
      pub mode: u32,
      pub nlink: u32,
      pub uid: u32,
      pub gid: u32,
      pub size: i64,
      pub atime: i64,
      pub mtime: i64,
      pub ctime: i64,
  }

  impl Stats {
      pub fn is_file(&self) -> bool { (self.mode & S_IFMT) == S_IFREG }
      pub fn is_directory(&self) -> bool { (self.mode & S_IFMT) == S_IFDIR }
      pub fn is_symlink(&self) -> bool { (self.mode & S_IFMT) == S_IFLNK }
  }
  ```

- [ ] **1.5.2**: Implement `Filesystem` struct
  ```rust
  pub struct Filesystem {
      conn: Arc<Connection>,
  }

  impl Filesystem {
      pub async fn from_connection(conn: Arc<Connection>) -> Result<Self> {
          let fs = Self { conn };
          fs.initialize().await?;
          Ok(fs)
      }
  }
  ```

- [ ] **1.5.3**: Implement `initialize()` - Create tables and root
  ```rust
  async fn initialize(&self) -> Result<()> {
      // CREATE TABLE fs_inode
      // CREATE TABLE fs_dentry
      // CREATE TABLE fs_data
      // CREATE TABLE fs_symlink
      // CREATE INDEX idx_fs_dentry_parent
      // CREATE INDEX idx_fs_data_ino_offset

      self.ensure_root().await?;
      Ok(())
  }

  async fn ensure_root(&self) -> Result<()> {
      // Check if root (ino=1) exists
      // If not, INSERT INTO fs_inode (ino, mode, ...) VALUES (1, 16877, ...)
  }
  ```

- [ ] **1.5.4**: Implement path resolution
  ```rust
  async fn resolve_path(&self, path: &str) -> Result<Option<i64>> {
      if path == "/" {
          return Ok(Some(ROOT_INO));
      }

      let mut current_ino = ROOT_INO;
      for component in path.split('/').filter(|s| !s.is_empty()) {
          // SELECT ino FROM fs_dentry WHERE parent_ino = ? AND name = ?
          // If not found, return None
          // Otherwise, update current_ino
      }
      Ok(Some(current_ino))
  }
  ```

- [ ] **1.5.5**: Implement `mkdir()`
  ```rust
  pub async fn mkdir(&self, path: &str) -> Result<()> {
      // 1. Split path into parent + name
      // 2. Resolve parent to ino
      // 3. Check if already exists
      // 4. INSERT INTO fs_inode (mode, atime, mtime, ctime) VALUES (DEFAULT_DIR_MODE, ...)
      // 5. INSERT INTO fs_dentry (name, parent_ino, ino) VALUES (...)
  }
  ```

- [ ] **1.5.6**: Implement `write_file()`
  ```rust
  pub async fn write_file(&self, path: &str, data: &[u8]) -> Result<()> {
      // 1. Ensure parent directories exist (mkdir -p)
      // 2. Resolve path to see if file exists
      // 3. If exists, delete old data chunks
      // 4. If not exists, create inode + dentry
      // 5. INSERT INTO fs_data (ino, offset, size, data) VALUES (...)
      // 6. UPDATE fs_inode SET size=?, mtime=? WHERE ino=?
  }
  ```

- [ ] **1.5.7**: Implement `read_file()`
  ```rust
  pub async fn read_file(&self, path: &str) -> Result<Option<Vec<u8>>> {
      // 1. Resolve path to ino
      // 2. SELECT data FROM fs_data WHERE ino=? ORDER BY offset
      // 3. Concatenate chunks
      // 4. UPDATE fs_inode SET atime=? WHERE ino=?
  }
  ```

**Validation**: Basic filesystem tests pass

---

### Task 1.6: Filesystem Implementation (Part 2: Advanced)

**Time:** 3 days

- [ ] **1.6.1**: Implement `readdir()`
  ```rust
  pub async fn readdir(&self, path: &str) -> Result<Option<Vec<String>>> {
      // 1. Resolve path to ino
      // 2. Verify it's a directory
      // 3. SELECT name FROM fs_dentry WHERE parent_ino=? ORDER BY name
  }
  ```

- [ ] **1.6.2**: Implement `stat()` and `lstat()`
  ```rust
  pub async fn stat(&self, path: &str) -> Result<Option<Stats>> {
      // 1. Resolve path (following symlinks)
      // 2. SELECT * FROM fs_inode WHERE ino=?
      // 3. COUNT links: SELECT COUNT(*) FROM fs_dentry WHERE ino=?
  }

  pub async fn lstat(&self, path: &str) -> Result<Option<Stats>> {
      // Same but don't follow symlinks
  }
  ```

- [ ] **1.6.3**: Implement `symlink()` and `readlink()`
  ```rust
  pub async fn symlink(&self, target: &str, linkpath: &str) -> Result<()> {
      // 1. Create inode with S_IFLNK mode
      // 2. Create dentry
      // 3. INSERT INTO fs_symlink (ino, target) VALUES (...)
  }

  pub async fn readlink(&self, path: &str) -> Result<Option<String>> {
      // 1. Resolve path without following symlinks
      // 2. SELECT target FROM fs_symlink WHERE ino=?
  }
  ```

- [ ] **1.6.4**: Implement `remove()`
  ```rust
  pub async fn remove(&self, path: &str) -> Result<()> {
      // 1. Resolve path to ino
      // 2. DELETE FROM fs_dentry WHERE parent_ino=? AND name=?
      // 3. Check link count
      // 4. If link count = 0:
      //    - DELETE FROM fs_data WHERE ino=?
      //    - DELETE FROM fs_symlink WHERE ino=?
      //    - DELETE FROM fs_inode WHERE ino=?
  }
  ```

- [ ] **1.6.5**: Write comprehensive filesystem tests
  ```rust
  #[tokio::test]
  async fn test_mkdir() { ... }

  #[tokio::test]
  async fn test_write_read_file() { ... }

  #[tokio::test]
  async fn test_readdir() { ... }

  #[tokio::test]
  async fn test_stat() { ... }

  #[tokio::test]
  async fn test_symlink() { ... }

  #[tokio::test]
  async fn test_remove() { ... }

  #[tokio::test]
  async fn test_hard_links() { ... }
  ```

**Validation**: All filesystem tests pass

---

### Task 1.7: Integration Tests for Rust SDK

**Time:** 2 days

- [ ] **1.7.1**: Create integration test file
  ```bash
  mkdir -p sdk/rust/tests
  touch sdk/rust/tests/integration.rs
  ```

- [ ] **1.7.2**: Test cross-component interactions
  ```rust
  #[tokio::test]
  async fn test_agentfs_full_workflow() {
      // 1. Create AgentFS
      // 2. Write file
      // 3. Set KV
      // 4. Record tool call
      // 5. Read back and verify
  }

  #[tokio::test]
  async fn test_agentfs_persistence() {
      // 1. Create database file
      // 2. Write data
      // 3. Close connection
      // 4. Reopen
      // 5. Verify data still there
  }
  ```

- [ ] **1.7.3**: Test edge cases
  ```rust
  #[tokio::test]
  async fn test_large_file() {
      // Write 10MB file, read back, verify
  }

  #[tokio::test]
  async fn test_unicode_paths() {
      // Test paths with Unicode characters
  }

  #[tokio::test]
  async fn test_concurrent_access() {
      // Multiple tokio tasks accessing same DB
  }
  ```

**Validation**: All integration tests pass

---

## Phase 2: TypeScript SDK

**Duration:** 2 weeks
**Goal:** Port Rust SDK to TypeScript with feature parity

### Task 2.1: TypeScript Project Setup

**Time:** 1 day

- [ ] **2.1.1**: Initialize npm package
  ```bash
  cd sdk/typescript
  npm init -y
  ```

- [ ] **2.1.2**: Configure `package.json`
  ```json
  {
    "name": "agentfs-sdk",
    "version": "0.1.2",
    "main": "dist/index.js",
    "types": "dist/index.d.ts",
    "scripts": {
      "build": "tsc",
      "test": "vitest",
      "prepublishOnly": "npm run build"
    },
    "dependencies": {
      "@tursodatabase/database": "^0.3.2"
    },
    "devDependencies": {
      "@types/node": "^20",
      "typescript": "^5.3",
      "vitest": "latest"
    }
  }
  ```

- [ ] **2.1.3**: Create `tsconfig.json`
  ```json
  {
    "compilerOptions": {
      "target": "ES2022",
      "module": "ESNext",
      "moduleResolution": "node",
      "declaration": true,
      "outDir": "./dist",
      "strict": true,
      "esModuleInterop": true
    },
    "include": ["src/**/*"]
  }
  ```

- [ ] **2.1.4**: Create source structure
  ```bash
  mkdir -p src tests
  touch src/index.ts src/filesystem.ts src/kvstore.ts src/toolcalls.ts
  ```

**Validation**: `npm run build` succeeds

---

### Task 2.2: TypeScript KvStore Implementation

**Time:** 1 day

**File**: `sdk/typescript/src/kvstore.ts`

- [ ] **2.2.1**: Implement KvStore class
  ```typescript
  import { Database } from '@tursodatabase/database';

  export class KvStore {
      constructor(private db: Database) {}

      async initialize(): Promise<void> {
          await this.db.exec(`CREATE TABLE IF NOT EXISTS kv_store (...)`);
      }

      async set(key: string, value: any): Promise<void> {
          const json = JSON.stringify(value);
          await this.db.exec({
              sql: `INSERT INTO kv_store (key, value, updated_at) VALUES (?, ?, unixepoch())
                    ON CONFLICT(key) DO UPDATE SET value=excluded.value, updated_at=unixepoch()`,
              args: [key, json]
          });
      }

      async get(key: string): Promise<any> {
          const result = await this.db.query({
              sql: 'SELECT value FROM kv_store WHERE key = ?',
              args: [key]
          });
          if (result.rows.length === 0) return undefined;
          return JSON.parse(result.rows[0].value as string);
      }

      async delete(key: string): Promise<void> { ... }

      async keys(): Promise<string[]> { ... }
  }
  ```

- [ ] **2.2.2**: Write tests
  ```typescript
  import { describe, it, expect } from 'vitest';

  describe('KvStore', () => {
      it('should set and get values', async () => { ... });
      it('should upsert values', async () => { ... });
      it('should delete keys', async () => { ... });
  });
  ```

**Validation**: KvStore tests pass

---

### Task 2.3: TypeScript ToolCalls Implementation

**Time:** 1 day

**File**: `sdk/typescript/src/toolcalls.ts`

- [ ] **2.3.1**: Define types
  ```typescript
  export interface ToolCall {
      id: number;
      name: string;
      parameters?: any;
      result?: any;
      error?: string;
      started_at: number;
      completed_at: number;
      duration_ms: number;
  }

  export interface ToolCallStats {
      name: string;
      total_calls: number;
      successful: number;
      failed: number;
      avg_duration_ms: number;
  }
  ```

- [ ] **2.3.2**: Implement ToolCalls class (mirror Rust API)
  ```typescript
  export class ToolCalls {
      constructor(private db: Database) {}

      async initialize(): Promise<void> { ... }

      async record(
          name: string,
          started_at: number,
          completed_at: number,
          parameters?: any,
          result?: any,
          error?: string
      ): Promise<number> { ... }

      async get(id: number): Promise<ToolCall | undefined> { ... }

      async getByName(name: string, limit?: number): Promise<ToolCall[]> { ... }

      async getRecent(since: number, limit?: number): Promise<ToolCall[]> { ... }

      async getStats(): Promise<ToolCallStats[]> { ... }
  }
  ```

- [ ] **2.3.3**: Write tests (translate from Rust tests)

**Validation**: ToolCalls tests pass

---

### Task 2.4: TypeScript Filesystem Implementation

**Time:** 5 days

**File**: `sdk/typescript/src/filesystem.ts`

- [ ] **2.4.1**: Define types and constants
  ```typescript
  const S_IFMT = 0o170000;
  const S_IFREG = 0o100000;
  const S_IFDIR = 0o040000;
  const S_IFLNK = 0o120000;
  const DEFAULT_FILE_MODE = S_IFREG | 0o644;
  const DEFAULT_DIR_MODE = S_IFDIR | 0o755;
  const ROOT_INO = 1;

  export interface Stats {
      ino: number;
      mode: number;
      nlink: number;
      uid: number;
      gid: number;
      size: number;
      atime: number;
      mtime: number;
      ctime: number;
      isFile(): boolean;
      isDirectory(): boolean;
      isSymbolicLink(): boolean;
  }
  ```

- [ ] **2.4.2**: Implement Filesystem class (mirror Rust SDK)
  - [ ] `initialize()`
  - [ ] `resolvePath()`
  - [ ] `mkdir()`
  - [ ] `writeFile()`
  - [ ] `readFile()`
  - [ ] `readdir()`
  - [ ] `stat()` and `lstat()`
  - [ ] `symlink()` and `readlink()`
  - [ ] `deleteFile()`

- [ ] **2.4.3**: Write comprehensive tests (translate from Rust)

**Validation**: All filesystem tests pass

---

### Task 2.5: TypeScript AgentFS Main Class

**Time:** 1 day

**File**: `sdk/typescript/src/index.ts`

- [ ] **2.5.1**: Implement AgentFS class
  ```typescript
  import { Database } from '@tursodatabase/database';
  import { KvStore } from './kvstore.js';
  import { Filesystem } from './filesystem.js';
  import { ToolCalls } from './toolcalls.js';

  export class AgentFS {
      public readonly kv: KvStore;
      public readonly fs: Filesystem;
      public readonly tools: ToolCalls;

      private constructor(private db: Database) {
          this.kv = new KvStore(db);
          this.fs = new Filesystem(db);
          this.tools = new ToolCalls(db);
      }

      static async open(dbPath: string = ':memory:'): Promise<AgentFS> {
          const db = new Database(dbPath);
          const agentfs = new AgentFS(db);

          await agentfs.kv.initialize();
          await agentfs.fs.initialize();
          await agentfs.tools.initialize();

          return agentfs;
      }

      async close(): Promise<void> {
          await this.db.close();
      }

      getDatabase(): Database {
          return this.db;
      }
  }

  export * from './filesystem.js';
  export * from './kvstore.js';
  export * from './toolcalls.js';
  ```

- [ ] **2.5.2**: Test full workflow
  ```typescript
  it('should create and use AgentFS instance', async () => {
      const agentfs = await AgentFS.open(':memory:');

      await agentfs.kv.set('test', { value: 42 });
      const val = await agentfs.kv.get('test');
      expect(val.value).toBe(42);

      await agentfs.fs.writeFile('/test.txt', 'hello');
      const content = await agentfs.fs.readFile('/test.txt');
      expect(content).toBe('hello');

      await agentfs.close();
  });
  ```

**Validation**: TypeScript SDK fully functional

---

### Task 2.6: Cross-Language Compatibility Tests

**Time:** 1 day

- [ ] **2.6.1**: Create test that uses both Rust and TypeScript SDKs
  ```typescript
  // Write with TypeScript SDK
  const agentfs = await AgentFS.open('test.db');
  await agentfs.fs.writeFile('/data.txt', 'from typescript');
  await agentfs.close();

  // Read with Rust SDK (via CLI or separate test)
  // Verify content matches
  ```

- [ ] **2.6.2**: Verify schema compatibility
  ```typescript
  // Both SDKs should produce identical database schemas
  // Compare PRAGMA table_info for all tables
  ```

**Validation**: Data written by one SDK readable by the other

---

## Phase 3: CLI Component

**Duration:** 1 week
**Goal:** Command-line tool for managing agent filesystems

### Task 3.1: CLI Project Setup

**Time:** 1 day

- [ ] **3.1.1**: Initialize CLI package
  ```bash
  cd cli
  cargo init --bin
  ```

- [ ] **3.1.2**: Configure dependencies
  ```toml
  [dependencies]
  agentfs-sdk = { path = "../sdk/rust" }
  tokio = { version = "1", features = ["full"] }
  clap = { version = "4", features = ["derive"] }
  anyhow = "1.0"
  turso = "0.3.2"

  [target.'cfg(target_os = "linux")'.dependencies]
  agentfs-sandbox = { path = "../sandbox" }
  reverie = { git = "https://github.com/facebookexperimental/reverie" }
  reverie-ptrace = { git = "https://github.com/facebookexperimental/reverie" }
  reverie-process = { git = "https://github.com/facebookexperimental/reverie" }
  ```

- [ ] **3.1.3**: Create command structure
  ```bash
  mkdir -p src/cmd
  touch src/main.rs src/cmd/mod.rs
  ```

**Validation**: CLI builds successfully

---

### Task 3.2: Implement CLI Commands

**Time:** 3 days

**File**: `cli/src/main.rs`

- [ ] **3.2.1**: Define CLI structure with clap
  ```rust
  use clap::{Parser, Subcommand};

  #[derive(Parser)]
  #[command(name = "agentfs")]
  #[command(about = "The filesystem for agents")]
  struct Args {
      #[command(subcommand)]
      command: Commands,
  }

  #[derive(Subcommand)]
  enum Commands {
      Init {
          filename: Option<String>,
          #[arg(long)]
          force: bool,
      },
      Fs {
          #[command(subcommand)]
          command: FsCommands,
      },
      #[cfg(target_os = "linux")]
      Run {
          #[arg(long, value_name = "MOUNT_SPEC")]
          mount: Vec<String>,
          #[arg(long)]
          strace: bool,
          command: String,
          args: Vec<String>,
      },
  }

  #[derive(Subcommand)]
  enum FsCommands {
      Ls {
          #[arg(default_value = "agent.db")]
          filesystem: String,
          #[arg(default_value = "/")]
          path: String,
      },
      Cat {
          #[arg(default_value = "agent.db")]
          filesystem: String,
          path: String,
      },
  }
  ```

- [ ] **3.2.2**: Implement `init` command
  ```rust
  async fn init_database(filename: Option<String>, force: bool) -> Result<()> {
      let filename = filename.unwrap_or_else(|| "agent.db".to_string());

      if !force && std::path::Path::new(&filename).exists() {
          anyhow::bail!("File already exists. Use --force to overwrite.");
      }

      let _agentfs = AgentFS::new(&filename).await?;
      println!("Created agent filesystem: {}", filename);
      Ok(())
  }
  ```

- [ ] **3.2.3**: Implement `fs ls` command
  ```rust
  async fn ls_filesystem(filesystem: String, path: String) -> Result<()> {
      let db = Builder::new_local(&filesystem).build().await?;
      let conn = db.connect()?;

      // Resolve path to inode
      let ino = resolve_path(&conn, &path).await?;

      // Query directory entries
      let rows = conn.query(
          "SELECT d.name, i.mode FROM fs_dentry d
           JOIN fs_inode i ON d.ino = i.ino
           WHERE d.parent_ino = ?
           ORDER BY d.name",
          (ino,)
      ).await?;

      for row in rows {
          let name = row.get::<String>(0)?;
          let mode = row.get::<u32>(1)?;
          let type_char = if (mode & S_IFDIR) != 0 { 'd' } else { 'f' };
          println!("{} {}", type_char, name);
      }
      Ok(())
  }
  ```

- [ ] **3.2.4**: Implement `fs cat` command
  ```rust
  async fn cat_filesystem(filesystem: String, path: String) -> Result<()> {
      let agentfs = AgentFS::new(&filesystem).await?;

      let content = agentfs.fs.read_file(&path).await?
          .ok_or_else(|| anyhow::anyhow!("File not found: {}", path))?;

      std::io::stdout().write_all(&content)?;
      Ok(())
  }
  ```

- [ ] **3.2.5**: Wire up main function
  ```rust
  #[tokio::main]
  async fn main() -> Result<()> {
      let args = Args::parse();

      match args.command {
          Commands::Init { filename, force } => {
              init_database(filename, force).await?;
          }
          Commands::Fs { command } => match command {
              FsCommands::Ls { filesystem, path } => {
                  ls_filesystem(filesystem, path).await?;
              }
              FsCommands::Cat { filesystem, path } => {
                  cat_filesystem(filesystem, path).await?;
              }
          },
          #[cfg(target_os = "linux")]
          Commands::Run { mount, strace, command, args } => {
              // Defer to sandbox (Phase 4)
              todo!("Sandbox implementation in Phase 4")
          }
      }

      Ok(())
  }
  ```

**Validation**: CLI commands work manually

---

### Task 3.3: CLI Integration Tests

**Time:** 1 day

- [ ] **3.3.1**: Create test scripts
  ```bash
  mkdir -p cli/tests
  touch cli/tests/test_cli.sh
  ```

- [ ] **3.3.2**: Test `init` command
  ```bash
  #!/bin/bash
  set -e

  # Test init
  cargo run -- init test.db
  [ -f test.db ] || exit 1

  # Test force flag
  cargo run -- init --force test.db
  ```

- [ ] **3.3.3**: Test `fs` commands
  ```bash
  # Use SDK to write test data
  # Then verify CLI can read it
  cargo run -- fs ls test.db /
  cargo run -- fs cat test.db /hello.txt
  ```

**Validation**: All CLI tests pass

---

## Phase 4: Sandbox Component (Linux Only)

**Duration:** 3 weeks
**Goal:** Syscall interception for virtual filesystem at `/agent`

**Note**: This phase is complex and Linux-specific. Consider deferring if targeting cross-platform first.

### Task 4.1: Sandbox Project Setup

**Time:** 1 day

- [ ] **4.1.1**: Initialize sandbox package
  ```bash
  cd sandbox
  cargo init --lib
  ```

- [ ] **4.1.2**: Configure dependencies
  ```toml
  [dependencies]
  agentfs-sdk = { path = "../sdk/rust" }
  tokio = { version = "1", features = ["full"] }
  anyhow = "1.0"
  async-trait = "0.1"

  [target.'cfg(target_os = "linux")'.dependencies]
  reverie = { git = "https://github.com/facebookexperimental/reverie" }
  reverie-ptrace = { git = "https://github.com/facebookexperimental/reverie" }
  reverie-process = { git = "https://github.com/facebookexperimental/reverie" }
  reverie-util = { git = "https://github.com/facebookexperimental/reverie" }
  libc = "0.2"
  ```

- [ ] **4.1.3**: Create module structure
  ```bash
  mkdir -p src/sandbox src/vfs src/syscall
  touch src/lib.rs
  touch src/sandbox/mod.rs
  touch src/vfs/{mod.rs,mount.rs,bind.rs,sqlite.rs,file.rs,fdtable.rs}
  touch src/syscall/{mod.rs,file.rs,stat.rs,process.rs}
  ```

**Validation**: Sandbox builds on Linux

---

### Task 4.2: VFS Layer Implementation

**Time:** 5 days

- [ ] **4.2.1**: Define `Vfs` trait (`vfs/mod.rs`)
  ```rust
  #[async_trait]
  pub trait Vfs: Send + Sync {
      async fn open(&self, path: &str, flags: i32, mode: u32) -> VfsResult<VirtualFile>;
      async fn stat(&self, path: &str) -> VfsResult<VfsStat>;
      async fn lstat(&self, path: &str) -> VfsResult<VfsStat>;
      async fn readdir(&self, path: &str) -> VfsResult<Vec<DirEntry>>;
      async fn mkdir(&self, path: &str, mode: u32) -> VfsResult<()>;
      async fn unlink(&self, path: &str) -> VfsResult<()>;
      async fn symlink(&self, target: &str, linkpath: &str) -> VfsResult<()>;
      async fn readlink(&self, path: &str) -> VfsResult<String>;
  }

  pub struct VirtualFile {
      content: Vec<u8>,
      position: usize,
  }

  impl VirtualFile {
      pub fn read(&mut self, buf: &mut [u8]) -> usize { ... }
      pub fn write(&mut self, buf: &[u8]) -> usize { ... }
  }
  ```

- [ ] **4.2.2**: Implement `BindVfs` (`vfs/bind.rs`)
  ```rust
  pub struct BindVfs {
      host_path: PathBuf,
  }

  #[async_trait]
  impl Vfs for BindVfs {
      async fn open(&self, path: &str, flags: i32, mode: u32) -> VfsResult<VirtualFile> {
          let real_path = self.host_path.join(path.trim_start_matches('/'));
          let content = tokio::fs::read(&real_path).await?;
          Ok(VirtualFile::new(content))
      }

      async fn stat(&self, path: &str) -> VfsResult<VfsStat> {
          let real_path = self.host_path.join(path.trim_start_matches('/'));
          let metadata = tokio::fs::metadata(&real_path).await?;
          Ok(VfsStat::from_metadata(metadata))
      }

      // ... other methods
  }
  ```

- [ ] **4.2.3**: Implement `SqliteVfs` (`vfs/sqlite.rs`)
  ```rust
  pub struct SqliteVfs {
      agentfs: AgentFS,
  }

  #[async_trait]
  impl Vfs for SqliteVfs {
      async fn open(&self, path: &str, flags: i32, mode: u32) -> VfsResult<VirtualFile> {
          let content = self.agentfs.fs.read_file(path).await?
              .ok_or(VfsError::NotFound)?;
          Ok(VirtualFile::new(content))
      }

      async fn stat(&self, path: &str) -> VfsResult<VfsStat> {
          let stats = self.agentfs.fs.stat(path).await?
              .ok_or(VfsError::NotFound)?;
          Ok(VfsStat::from_agentfs_stats(stats))
      }

      // ... other methods
  }
  ```

- [ ] **4.2.4**: Implement `MountTable` (`vfs/mount.rs`)
  ```rust
  pub struct MountConfig {
      pub mount_type: MountType,
      pub source: String,
      pub target: PathBuf,
  }

  pub enum MountType {
      Bind,
      Sqlite,
  }

  pub struct MountTable {
      mounts: Vec<(PathBuf, Arc<dyn Vfs>)>,
  }

  impl MountTable {
      pub fn new(configs: Vec<MountConfig>) -> Result<Self> {
          let mut mounts = Vec::new();
          for config in configs {
              let vfs: Arc<dyn Vfs> = match config.mount_type {
                  MountType::Bind => Arc::new(BindVfs::new(&config.source)?),
                  MountType::Sqlite => Arc::new(SqliteVfs::new(&config.source).await?),
              };
              mounts.push((config.target, vfs));
          }
          Ok(Self { mounts })
      }

      pub fn find(&self, path: &Path) -> Option<(&Path, &Arc<dyn Vfs>, &Path)> {
          // Longest prefix match
          // Returns (mount_point, vfs, relative_path)
      }
  }
  ```

- [ ] **4.2.5**: Implement `FdTable` (`vfs/fdtable.rs`)
  ```rust
  pub struct FdTable {
      fds: HashMap<i32, VirtualFile>,
      next_fd: i32,
  }

  impl FdTable {
      pub fn new() -> Self {
          Self {
              fds: HashMap::new(),
              next_fd: 100, // Start at 100 to avoid conflicts with stdin/stdout/stderr
          }
      }

      pub fn insert(&mut self, file: VirtualFile) -> i32 {
          let fd = self.next_fd;
          self.fds.insert(fd, file);
          self.next_fd += 1;
          fd
      }

      pub fn get_mut(&mut self, fd: i32) -> Option<&mut VirtualFile> {
          self.fds.get_mut(&fd)
      }

      pub fn remove(&mut self, fd: i32) -> Option<VirtualFile> {
          self.fds.remove(&fd)
      }
  }
  ```

**Validation**: VFS trait and implementations compile

---

### Task 4.3: Syscall Handlers Implementation

**Time:** 5 days

- [ ] **4.3.1**: Implement syscall dispatcher (`syscall/mod.rs`)
  ```rust
  use libc::*;

  pub fn dispatch_syscall(
      syscall: &Syscall,
      mount_table: &MountTable,
      fd_table: &mut FdTable,
  ) -> Result<i64> {
      match syscall.number() as i64 {
          SYS_openat => handle_openat(syscall, mount_table, fd_table),
          SYS_read => handle_read(syscall, fd_table),
          SYS_write => handle_write(syscall, fd_table),
          SYS_close => handle_close(syscall, fd_table),
          SYS_stat | SYS_lstat | SYS_fstat => handle_stat(syscall, mount_table, fd_table),
          SYS_getdents64 => handle_getdents64(syscall, mount_table, fd_table),
          // ... more syscalls
          _ => Err(anyhow!("Unsupported syscall: {}", syscall.number()))
      }
  }
  ```

- [ ] **4.3.2**: Implement file operations (`syscall/file.rs`)
  ```rust
  pub fn handle_openat(
      syscall: &Syscall,
      mount_table: &MountTable,
      fd_table: &mut FdTable,
  ) -> Result<i64> {
      let dirfd = syscall.arg0() as i32;
      let pathname = syscall.arg1() as *const c_char;
      let flags = syscall.arg2() as i32;
      let mode = syscall.arg3() as u32;

      // Resolve pathname (handle dirfd, AT_FDCWD, etc.)
      let path = resolve_pathname(dirfd, pathname)?;

      // Find mount point
      let (mount_point, vfs, rel_path) = mount_table.find(&path)
          .ok_or_else(|| anyhow!("Path not mounted: {:?}", path))?;

      // Open file via VFS
      let file = vfs.open(rel_path.to_str().unwrap(), flags, mode).await?;

      // Insert into FD table
      let fd = fd_table.insert(file);
      Ok(fd as i64)
  }

  pub fn handle_read(syscall: &Syscall, fd_table: &mut FdTable) -> Result<i64> {
      let fd = syscall.arg0() as i32;
      let buf = syscall.arg1() as *mut u8;
      let count = syscall.arg2() as usize;

      let file = fd_table.get_mut(fd)
          .ok_or_else(|| anyhow!("Invalid fd: {}", fd))?;

      let mut local_buf = vec![0u8; count];
      let bytes_read = file.read(&mut local_buf);

      // Copy to guest memory (use Reverie's memory API)
      unsafe { std::ptr::copy_nonoverlapping(local_buf.as_ptr(), buf, bytes_read); }

      Ok(bytes_read as i64)
  }

  // Similar implementations for write, close
  ```

- [ ] **4.3.3**: Implement stat operations (`syscall/stat.rs`)
  ```rust
  pub fn handle_stat(
      syscall: &Syscall,
      mount_table: &MountTable,
      fd_table: &FdTable,
  ) -> Result<i64> {
      // Handle SYS_stat, SYS_lstat, SYS_fstat
      // Query VFS for file metadata
      // Populate struct stat and copy to guest memory
  }

  pub fn handle_getdents64(
      syscall: &Syscall,
      mount_table: &MountTable,
      fd_table: &mut FdTable,
  ) -> Result<i64> {
      // Read directory entries
      // Populate linux_dirent64 structures
      // Copy to guest memory
  }
  ```

**Validation**: Syscall handlers compile

---

### Task 4.4: Reverie Sandbox Tool Implementation

**Time:** 3 days

- [ ] **4.4.1**: Implement global state (`sandbox/mod.rs`)
  ```rust
  use std::sync::{OnceLock, Mutex};
  use std::collections::HashMap;

  static MOUNT_TABLE: OnceLock<MountTable> = OnceLock::new();
  static FD_TABLES: OnceLock<Mutex<HashMap<Pid, FdTable>>> = OnceLock::new();
  static STRACE_ENABLED: AtomicBool = AtomicBool::new(false);

  pub fn init_mount_table(table: MountTable) {
      MOUNT_TABLE.set(table).expect("Mount table already initialized");
  }

  pub fn init_fd_tables() {
      FD_TABLES.set(Mutex::new(HashMap::new())).expect("FD tables already initialized");
  }

  pub fn init_strace(enabled: bool) {
      STRACE_ENABLED.store(enabled, Ordering::SeqCst);
  }

  fn get_mount_table() -> &'static MountTable {
      MOUNT_TABLE.get().expect("Mount table not initialized")
  }

  fn get_fd_table(pid: Pid) -> FdTable {
      let mut tables = FD_TABLES.get().unwrap().lock().unwrap();
      tables.entry(pid).or_insert_with(FdTable::new).clone()
  }
  ```

- [ ] **4.4.2**: Implement Reverie Tool
  ```rust
  use reverie::{Tool, Guest, Syscall};

  #[derive(Debug, Default, Clone)]
  pub struct Sandbox;

  #[reverie::tool]
  impl Tool for Sandbox {
      type GlobalState = ();
      type ThreadState = ();

      async fn handle_syscall_event<T: Guest<Self>>(
          &self,
          guest: &mut T,
          syscall: Syscall,
      ) -> Result<i64, reverie::Error> {
          let mount_table = get_mount_table();
          let mut fd_table = get_fd_table(guest.tid().as_raw());

          if STRACE_ENABLED.load(Ordering::SeqCst) {
              eprintln!("[strace] {:?}", syscall);
          }

          match dispatch_syscall(&syscall, mount_table, &mut fd_table) {
              Ok(ret) => Ok(ret),
              Err(e) => {
                  eprintln!("Syscall error: {}", e);
                  // Return appropriate error code
                  Ok(-1)
              }
          }
      }
  }
  ```

- [ ] **4.4.3**: Wire into CLI `run` command (`cli/src/cmd/run_linux.rs`)
  ```rust
  use agentfs_sandbox::{Sandbox, init_mount_table, init_fd_tables, init_strace};
  use reverie_ptrace::TracerBuilder;

  pub async fn run_sandbox(
      mounts: Vec<String>,
      strace: bool,
      command: String,
      args: Vec<String>,
  ) -> Result<()> {
      // Parse mount specs
      let mount_configs = parse_mount_specs(&mounts)?;

      // Add default agent.db mount
      let mut configs = mount_configs;
      configs.push(MountConfig {
          mount_type: MountType::Sqlite,
          source: "agent.db".to_string(),
          target: PathBuf::from("/agent"),
      });

      // Initialize globals
      let mount_table = MountTable::new(configs).await?;
      init_mount_table(mount_table);
      init_fd_tables();
      init_strace(strace);

      // Run command with Reverie
      let tracer = TracerBuilder::new()
          .spawn(Sandbox::default(), command, args)
          .await?;

      let (exit_code, _) = tracer.wait().await?;
      std::process::exit(exit_code.unwrap_or(1));
  }
  ```

**Validation**: Sandbox runs simple commands (echo, ls)

---

### Task 4.5: Sandbox Testing

**Time:** 3 days

- [ ] **4.5.1**: Test basic commands
  ```bash
  # Test echo
  agentfs run echo "hello world"

  # Test file operations
  agentfs run bash -c 'echo "test" > /agent/file.txt && cat /agent/file.txt'

  # Verify persistence
  agentfs fs cat /file.txt
  ```

- [ ] **4.5.2**: Test with real programs
  ```bash
  # Python
  agentfs run python3 -c 'open("/agent/output.txt", "w").write("from python")'

  # Node.js
  agentfs run node -e 'require("fs").writeFileSync("/agent/node.txt", "from node")'
  ```

- [ ] **4.5.3**: Test bind mounts
  ```bash
  agentfs run --mount type=bind,src=/tmp,dst=/data ls /data
  ```

- [ ] **4.5.4**: Test strace mode
  ```bash
  agentfs run --strace cat /agent/file.txt
  # Should see syscall traces
  ```

**Validation**: Sandbox works with complex programs

---

## Phase 5: Examples & Documentation

**Duration:** 1 week
**Goal:** Create working examples and comprehensive docs

### Task 5.1: Create Example Applications

**Time:** 3 days

- [ ] **5.1.1**: Create OpenAI Agents example
  ```bash
  mkdir -p examples/openai-agents/research-assistant
  cd examples/openai-agents/research-assistant
  npm init -y
  ```
  - [ ] Add AgentFS integration in `src/utils/agentfs.ts`
  - [ ] Create PDF fetching tool with caching
  - [ ] Create semantic search tool
  - [ ] Wire into OpenAI Agents SDK

- [ ] **5.1.2**: Create Claude Agent SDK example
  - [ ] Similar structure to OpenAI example
  - [ ] Use `@anthropic-ai/claude-agent-sdk`
  - [ ] Demonstrate MCP server integration

- [ ] **5.1.3**: Create Mastra example
  - [ ] Use `@mastra/core`
  - [ ] Show workflow-based architecture
  - [ ] Demonstrate evaluation/scoring

**Validation**: All examples run successfully

---

### Task 5.2: Documentation

**Time:** 2 days

- [ ] **5.2.1**: Create `README.md`
  - [ ] What is AgentFS?
  - [ ] Quick start guide
  - [ ] Why AgentFS? (motivation)
  - [ ] Installation instructions
  - [ ] Links to detailed docs

- [ ] **5.2.2**: Create `MANUAL.md`
  - [ ] CLI reference (`init`, `fs`, `run`)
  - [ ] SDK API reference (TypeScript)
  - [ ] SDK API reference (Rust)
  - [ ] Examples section
  - [ ] Advanced usage (snapshots, querying, Turso)

- [ ] **5.2.3**: Create `SPEC.md` (if not exists)
  - [ ] SQLite schema specification
  - [ ] Operations for each component
  - [ ] Consistency rules
  - [ ] Extension points

- [ ] **5.2.4**: Create `CONTRIBUTING.md`
  - [ ] Development setup
  - [ ] Running tests
  - [ ] Code style guidelines
  - [ ] Pull request process

**Validation**: Documentation is clear and comprehensive

---

## Phase 6: Testing, Distribution & CI/CD

**Duration:** 1 week
**Goal:** Production-ready testing and distribution

### Task 6.1: Comprehensive Testing

**Time:** 2 days

- [ ] **6.1.1**: Unit test coverage
  - [ ] Rust SDK: 80%+ coverage
  - [ ] TypeScript SDK: 80%+ coverage
  - [ ] CLI: Core commands covered
  - [ ] Sandbox: VFS and syscall handlers covered

- [ ] **6.1.2**: Integration tests
  - [ ] Cross-language compatibility (Rust ↔ TypeScript)
  - [ ] CLI end-to-end workflows
  - [ ] Sandbox with real programs

- [ ] **6.1.3**: Performance benchmarks
  - [ ] File write/read speed
  - [ ] Tool call overhead
  - [ ] Sandbox syscall overhead
  - [ ] Database growth patterns

**Validation**: Tests pass, performance acceptable

---

### Task 6.2: Distribution Setup

**Time:** 2 days

- [ ] **6.2.1**: Configure cargo-dist
  ```toml
  # dist-workspace.toml
  [workspace]
  members = ["cargo:cli"]

  [workspace.metadata.dist]
  cargo-dist-version = "0.30.2"

  [[workspace.metadata.dist.targets]]
  linux-x86_64 = "x86_64-unknown-linux-gnu"
  macos-aarch64 = "aarch64-apple-darwin"
  macos-x86_64 = "x86_64-apple-darwin"
  windows-x86_64 = "x86_64-pc-windows-msvc"

  [[workspace.metadata.dist.installers]]
  shell = true
  powershell = true
  ```

- [ ] **6.2.2**: Prepare npm package
  ```json
  // sdk/typescript/package.json
  {
    "files": ["dist/**/*"],
    "scripts": {
      "prepublishOnly": "npm run build && npm test"
    }
  }
  ```

- [ ] **6.2.3**: Test local builds
  ```bash
  # Rust
  cargo dist build

  # TypeScript
  cd sdk/typescript && npm pack
  ```

**Validation**: Builds succeed for all platforms

---

### Task 6.3: CI/CD Pipeline

**Time:** 2 days

- [ ] **6.3.1**: Create GitHub Actions workflow
  ```yaml
  # .github/workflows/ci.yml
  name: CI

  on: [push, pull_request]

  jobs:
    test-rust:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v4
        - uses: actions-rs/toolchain@v1
        - run: cargo test --all

    test-typescript:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v4
        - uses: actions/setup-node@v4
        - run: cd sdk/typescript && npm ci && npm test

    test-examples:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v4
        - run: cargo build --release
        - run: ./test-examples.sh

    release:
      if: startsWith(github.ref, 'refs/tags/')
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v4
        - run: cargo dist build --release
        - run: cd sdk/typescript && npm publish
  ```

- [ ] **6.3.2**: Setup automated releases
  - [ ] Tag-based releases
  - [ ] Changelog generation
  - [ ] crates.io publishing
  - [ ] npm publishing

**Validation**: CI passes on push, releases on tag

---

## Phase 7: Production Hardening (Optional)

**Duration:** 1-2 weeks
**Goal**: Address technical debt and production concerns

### Task 7.1: Transaction Safety

- [ ] **7.1.1**: Wrap multi-step operations in explicit transactions
- [ ] **7.1.2**: Add rollback on error
- [ ] **7.1.3**: Test transaction isolation

### Task 7.2: Quota Management

- [ ] **7.2.1**: Add quota tracking tables
- [ ] **7.2.2**: Enforce limits in filesystem operations
- [ ] **7.2.3**: Expose quota APIs

### Task 7.3: Compression Support

- [ ] **7.3.1**: Add optional zstd compression for `fs_data`
- [ ] **7.3.2**: Transparent decompression on read
- [ ] **7.3.3**: Configuration flag

### Task 7.4: Migration Framework

- [ ] **7.4.1**: Create migration runner
- [ ] **7.4.2**: Version tracking in database
- [ ] **7.4.3**: Migration scripts for schema changes

---

## Timeline Summary

| Phase | Duration | Key Deliverables |
|-------|----------|------------------|
| 1. Rust SDK | 2-3 weeks | Filesystem, KV, ToolCalls |
| 2. TypeScript SDK | 2 weeks | Feature-parity with Rust |
| 3. CLI | 1 week | init, fs, run commands |
| 4. Sandbox | 3 weeks | VFS, syscall interception |
| 5. Examples & Docs | 1 week | 3 examples, comprehensive docs |
| 6. Testing & CI/CD | 1 week | Tests, distribution, automation |
| 7. Hardening (Optional) | 1-2 weeks | Transactions, quotas, compression |
| **Total** | **10-13 weeks** | Production-ready AgentFS |

---

## Success Criteria

The implementation SUCCEEDS when:

✅ All functional requirements from spec.md are implemented
✅ Both Rust and TypeScript SDKs have feature parity
✅ CLI works on all supported platforms
✅ Sandbox runs real programs successfully (Linux)
✅ Examples demonstrate integration with 3 AI frameworks
✅ Test coverage > 80% for SDK components
✅ Documentation is comprehensive and clear
✅ CI/CD pipeline automates testing and releases
✅ Binaries distributed for all target platforms

---

**Next Steps**:
1. Review `spec.md` and `plan.md` before starting
2. Setup development environment
3. Begin with Phase 1: Rust SDK
4. Test continuously as you build
5. Reference `intelligence-object.md` for architectural patterns
