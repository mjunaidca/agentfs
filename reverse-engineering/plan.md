# AgentFS Implementation Plan (Reverse Engineered)

**Version:** 0.1.2 (Current Implementation)
**Date:** 2025-11-18

---

## Architecture Overview

### Architectural Style

**Layered Architecture** with clear separation of concerns:

```
┌─────────────────────────────────────────────────┐
│         Application Layer                       │
│  (AI Agents using Claude SDK, Mastra, OpenAI)  │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│         SDK Layer (Dual Language)               │
│    ┌──────────────┐      ┌──────────────┐      │
│    │   Rust SDK   │      │ TypeScript   │      │
│    │              │      │    SDK       │      │
│    │ - Filesystem │      │ - Filesystem │      │
│    │ - KvStore    │      │ - KvStore    │      │
│    │ - ToolCalls  │      │ - ToolCalls  │      │
│    └──────────────┘      └──────────────┘      │
└────────────────┬────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────┐
│      Database Layer (SQLite/Turso)              │
│                                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐        │
│  │tool_calls│ │fs_inode  │ │kv_store  │        │
│  │          │ │fs_dentry │ │          │        │
│  │          │ │fs_data   │ │          │        │
│  │          │ │fs_symlink│ │          │        │
│  └──────────┘ └──────────┘ └──────────┘        │
└─────────────────────────────────────────────────┘

           Parallel Components:

┌──────────────────┐        ┌───────────────────┐
│   CLI Component  │        │  Sandbox (Linux)  │
│                  │        │                    │
│  - init          │        │  Reverie Ptrace    │
│  - fs ls/cat     │        │  VFS Layer         │
│  - run (sandbox) │        │  Syscall Handlers  │
└──────────────────┘        └───────────────────┘
```

### Reasoning

**Why Layered Architecture?**
1. **Separation of Concerns**: SDK, CLI, and Sandbox are independent
2. **Dual-Language Support**: Rust and TypeScript SDKs share same database layer
3. **Testability**: Each layer can be tested independently
4. **Reusability**: SDK used by both CLI and applications
5. **Platform Flexibility**: Sandbox is optional (Linux-only)

**Alternative Considered**: Monolithic architecture
- ❌ Rejected: Would couple SDK with CLI, preventing library use

**Alternative Considered**: Microservices
- ❌ Rejected: Violates "zero configuration" principle, adds deployment complexity

---

## Layer Structure

### Layer 1: Database Layer (Foundation)

**Responsibility**: Persistent storage of agent state using SQLite

**Technology**:
- [Turso](https://github.com/tursodatabase/turso) v0.3.2 (libSQL fork of SQLite)
- Supports local files and remote Turso databases
- Full ACID transactions

**Components**:

#### 1.1 Tool Calls Schema (`tool_calls` table)
```sql
CREATE TABLE tool_calls (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  parameters TEXT,          -- JSON
  result TEXT,              -- JSON
  error TEXT,
  started_at INTEGER NOT NULL,
  completed_at INTEGER NOT NULL,
  duration_ms INTEGER NOT NULL
)
```
**Purpose**: Insert-only audit log of all tool invocations

#### 1.2 Filesystem Schema (4 tables)
```sql
-- Inode metadata
CREATE TABLE fs_inode (
  ino INTEGER PRIMARY KEY AUTOINCREMENT,
  mode INTEGER NOT NULL,    -- File type + permissions
  uid INTEGER, gid INTEGER,
  size INTEGER,
  atime INTEGER, mtime INTEGER, ctime INTEGER
)

-- Directory entries (namespace)
CREATE TABLE fs_dentry (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  parent_ino INTEGER NOT NULL,
  ino INTEGER NOT NULL,
  UNIQUE(parent_ino, name)
)

-- File data (content)
CREATE TABLE fs_data (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  ino INTEGER NOT NULL,
  offset INTEGER NOT NULL,
  size INTEGER NOT NULL,
  data BLOB NOT NULL
)

-- Symbolic links
CREATE TABLE fs_symlink (
  ino INTEGER PRIMARY KEY,
  target TEXT NOT NULL
)
```
**Purpose**: Unix-like inode filesystem with hard link support

#### 1.3 Key-Value Schema (`kv_store` table)
```sql
CREATE TABLE kv_store (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL,      -- JSON
  created_at INTEGER,
  updated_at INTEGER
)
```
**Purpose**: Simple state management with timestamps

**Dependencies**: None (foundation layer)

---

### Layer 2: SDK Layer (Core Abstraction)

**Responsibility**: Provide type-safe, async APIs for filesystem, KV, and tool tracking

**Technology**: Rust 2021 + TypeScript 5.3

#### 2.1 Rust SDK (`sdk/rust`)

**Main Module**: `lib.rs`
```rust
pub struct AgentFS {
    conn: Arc<Connection>,
    pub kv: KvStore,
    pub fs: Filesystem,
    pub tools: ToolCalls,
}
```

**Sub-modules**:

**`filesystem.rs`** (632 lines)
- Path resolution with symlink following
- Inode allocation and management
- Operations: `mkdir`, `write_file`, `read_file`, `readdir`, `stat`, `lstat`, `symlink`, `readlink`, `remove`
- Automatic parent directory creation
- Hard link support via dentry sharing

**`kvstore.rs`** (150 lines)
- JSON serialization/deserialization
- Upsert with ON CONFLICT
- Operations: `set`, `get`, `delete`, `keys`

**`toolcalls.rs`** (250 lines)
- Timestamp tracking
- Duration calculation
- Operations: `start`, `success`, `error`, `record`, `get`, `recent`, `stats_for`, `stats`

**Key Design Patterns**:
- **Builder Pattern**: `AgentFS::new()` constructs with async initialization
- **Repository Pattern**: Each module (fs/kv/tools) encapsulates database access
- **Arc for Sharing**: Connection wrapped in `Arc<Connection>` for clone-ability
- **Error Handling**: `anyhow::Result` for ergonomic error propagation

#### 2.2 TypeScript SDK (`sdk/typescript`)

**Main Module**: `index.ts`
```typescript
export class AgentFS {
    private db: Database;
    public readonly kv: KvStore;
    public readonly fs: Filesystem;
    public readonly tools: ToolCalls;

    static async open(dbPath: string): Promise<AgentFS>
}
```

**Feature Parity with Rust**:
- Identical API surface
- Same async/await semantics
- JSON serialization built-in (TypeScript)
- Promise-based error handling

**Dependencies**:
- `@tursodatabase/database` v0.3.2

---

### Layer 3: CLI Component (User Interface)

**Responsibility**: Command-line tool for database management and sandbox execution

**Technology**: Rust with clap v4 (derive macros)

**Entry Point**: `cli/src/main.rs`

**Command Structure**:
```rust
#[derive(Parser)]
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
```

**Implementation**:

**`init` command** (`cmd/mod.rs:init_database`)
- Creates new AgentFS instance
- Initializes all tables
- Returns success message

**`fs ls` command** (`cmd/mod.rs:ls_filesystem`)
- Direct SQL query:
  ```sql
  SELECT d.name, i.mode
  FROM fs_dentry d
  JOIN fs_inode i ON d.ino = i.ino
  WHERE d.parent_ino = ?
  ```
- Formats output: `f <name>` or `d <name>`

**`fs cat` command** (`cmd/mod.rs:cat_filesystem`)
- Resolves path to inode
- Fetches chunks: `SELECT data FROM fs_data WHERE ino = ? ORDER BY offset`
- Concatenates and outputs

**`run` command** (Linux only) (`cmd/run_linux.rs`)
- Parses mount specifications
- Delegates to sandbox component
- Handles strace output

**Dependencies**:
- `agentfs-sdk` (local Rust SDK)
- `clap` for CLI parsing
- `agentfs-sandbox` (conditional, Linux only)

---

### Layer 4: Sandbox Component (Linux Execution)

**Responsibility**: Intercept syscalls to provide virtual filesystem at `/agent`

**Technology**:
- Reverie ptrace framework (Facebook)
- Linux-specific (ptrace requires Linux kernel)

**Architecture**:

```
User Program (bash, python, etc.)
         │
         │ syscall (open, read, write, etc.)
         ▼
┌─────────────────────────────────┐
│   Reverie Ptrace Framework      │
│   (Intercepts all syscalls)     │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│   Sandbox Tool (sandbox/mod.rs) │
│   - Global mount table          │
│   - Per-process FD tables       │
│   - Strace logging              │
└────────────┬────────────────────┘
             │
     ┌───────┴────────┐
     ▼                ▼
┌──────────┐    ┌──────────┐
│   VFS    │    │ Syscall  │
│  Layer   │    │ Handlers │
└──────────┘    └──────────┘
```

**Key Modules**:

#### 4.1 Sandbox Tool (`sandbox/src/sandbox/mod.rs`)

**Global State** (OnceLock and Mutex):
```rust
static MOUNT_TABLE: OnceLock<MountTable> = OnceLock::new();
static FD_TABLES: OnceLock<Mutex<HashMap<Pid, FdTable>>> = OnceLock::new();
static STRACE_ENABLED: AtomicBool = AtomicBool::new(false);
```

**Reverie Tool Implementation**:
```rust
#[derive(Debug, Default, Clone)]
pub struct Sandbox;

#[reverie::tool]
impl Tool for Sandbox {
    async fn handle_syscall_event(&self, guest: Guest<Self>, syscall: Syscall) -> Result<i64> {
        let mount_table = get_mount_table();
        let fd_table = get_fd_table(guest.pid());

        match syscall.number() {
            SYS_open | SYS_openat => handle_open(...),
            SYS_read => handle_read(...),
            SYS_write => handle_write(...),
            // ... 30+ syscall handlers
        }
    }
}
```

#### 4.2 VFS Layer (`sandbox/src/vfs/`)

**`Vfs` Trait** (`mod.rs`):
```rust
#[async_trait]
pub trait Vfs: Send + Sync {
    async fn open(&self, path: &str, flags: i32) -> VfsResult<VirtualFile>;
    async fn stat(&self, path: &str) -> VfsResult<Stat>;
    async fn readdir(&self, path: &str) -> VfsResult<Vec<String>>;
    async fn mkdir(&self, path: &str, mode: u32) -> VfsResult<()>;
    // ... more operations
}
```

**Implementations**:

**`BindVfs`** (`bind.rs`) - Passthrough to host filesystem
```rust
pub struct BindVfs {
    host_path: PathBuf,
}

impl Vfs for BindVfs {
    async fn open(&self, path: &str, flags: i32) -> VfsResult<VirtualFile> {
        let real_path = self.host_path.join(path.trim_start_matches('/'));
        // Use std::fs operations
    }
}
```

**`SqliteVfs`** (`sqlite.rs`) - AgentFS database backend
```rust
pub struct SqliteVfs {
    agentfs: AgentFS,
}

impl Vfs for SqliteVfs {
    async fn open(&self, path: &str, flags: i32) -> VfsResult<VirtualFile> {
        let content = self.agentfs.fs.read_file(path).await?;
        Ok(VirtualFile::new(content))
    }
}
```

**Mount Table** (`mount.rs`):
```rust
pub struct MountTable {
    mounts: Vec<(PathBuf, Arc<dyn Vfs>)>,
}

impl MountTable {
    pub fn find(&self, path: &Path) -> Option<(&Path, &Arc<dyn Vfs>)> {
        // Longest prefix match
    }
}
```

#### 4.3 Syscall Handlers (`sandbox/src/syscall/`)

**Dispatch** (`mod.rs`):
```rust
pub fn dispatch(syscall: &Syscall, mount_table: &MountTable, fd_table: &mut FdTable) -> Result<i64> {
    match syscall.number() {
        SYS_openat => handle_openat(...),
        SYS_read => handle_read(...),
        SYS_write => handle_write(...),
        SYS_stat => handle_stat(...),
        SYS_fstat => handle_fstat(...),
        SYS_getdents64 => handle_getdents64(...),
        // ... 30+ handlers
        _ => Err(anyhow!("Unsupported syscall: {}", syscall.number()))
    }
}
```

**File Descriptor Table** (`vfs/fdtable.rs`):
```rust
pub struct FdTable {
    fds: HashMap<i32, VirtualFile>,
    next_fd: i32,
}

impl FdTable {
    pub fn insert(&mut self, file: VirtualFile) -> i32 {
        let fd = self.next_fd;
        self.fds.insert(fd, file);
        self.next_fd += 1;
        fd
    }
}
```

**Dependencies**:
- `reverie`, `reverie-ptrace`, `reverie-process`, `reverie-util`
- `agentfs-sdk`
- `libc` for syscall constants

---

## Design Patterns Applied

### Pattern 1: Repository Pattern

**Location**: SDK layer (filesystem.rs, kvstore.rs, toolcalls.rs)

**Purpose**: Encapsulate database access behind domain-specific interfaces

**Implementation**:
```rust
// filesystem.rs
pub struct Filesystem {
    conn: Arc<Connection>,
}

impl Filesystem {
    pub async fn read_file(&self, path: &str) -> Result<Option<Vec<u8>>> {
        // 1. Resolve path to inode
        // 2. Query fs_data table
        // 3. Concatenate chunks
        // 4. Return bytes
    }
}
```

**Benefits**:
- Database schema hidden from users
- Easy to add caching layer later
- Testable with mocks

### Pattern 2: Virtual Filesystem (VFS)

**Location**: Sandbox component (`vfs/mod.rs`)

**Purpose**: Abstract different storage backends behind uniform interface

**Implementation**:
```rust
trait Vfs {
    async fn open(&self, path: &str, flags: i32) -> VfsResult<VirtualFile>;
    async fn stat(&self, path: &str) -> VfsResult<Stat>;
}

// Multiple implementations:
struct BindVfs { ... }      // Host filesystem
struct SqliteVfs { ... }    // AgentFS database
```

**Benefits**:
- Support multiple mount points
- Easy to add new backends (S3, HTTP, etc.)
- Consistent interface for syscall handlers

### Pattern 3: Singleton with Lazy Initialization

**Location**: Sandbox global state (`sandbox/mod.rs`)

**Purpose**: One mount table and FD tables shared across all syscalls

**Implementation**:
```rust
static MOUNT_TABLE: OnceLock<MountTable> = OnceLock::new();

pub fn init_mount_table(table: MountTable) {
    MOUNT_TABLE.set(table).expect("Mount table already initialized");
}

pub fn get_mount_table() -> &'static MountTable {
    MOUNT_TABLE.get().expect("Mount table not initialized")
}
```

**Benefits**:
- Zero-cost after initialization
- Thread-safe access
- Prevents accidental re-initialization

### Pattern 4: Strategy Pattern (Dual SDK)

**Location**: Rust SDK vs TypeScript SDK

**Purpose**: Same interface, different implementation languages

**Implementation**:
```rust
// Rust
pub struct AgentFS { ... }
impl AgentFS {
    pub async fn new(db_path: &str) -> Result<Self> { ... }
}

// TypeScript
export class AgentFS { ... }
static async open(dbPath: string): Promise<AgentFS> { ... }
```

**Benefits**:
- Users choose language based on ecosystem
- Consistent API across languages
- Shared database schema ensures compatibility

### Pattern 5: Inode-Based Filesystem

**Location**: Filesystem schema and implementation

**Purpose**: Separate namespace (paths) from data (file content)

**Implementation**:
```sql
-- Metadata
fs_inode: ino → (mode, size, timestamps)

-- Namespace
fs_dentry: (parent_ino, name) → ino

-- Data
fs_data: (ino, offset) → data
```

**Benefits**:
- Hard links: Multiple dentries → same inode
- Efficient renames: Update dentry, not data
- POSIX compatibility

### Pattern 6: Command Pattern (CLI)

**Location**: CLI commands (`cli/src/main.rs`)

**Purpose**: Encapsulate each operation as a command

**Implementation**:
```rust
enum Commands {
    Init { ... },
    Fs { command: FsCommands },
    Run { ... },
}

match args.command {
    Commands::Init { filename, force } => init_database(...),
    Commands::Fs { command } => match command {
        FsCommands::Ls { path } => ls_filesystem(...),
        FsCommands::Cat { path } => cat_filesystem(...),
    },
    Commands::Run { ... } => handle_run_command(...),
}
```

**Benefits**:
- Easy to add new commands
- Clap handles parsing and help text
- Clear separation of command logic

---

## Data Flow

### Data Flow 1: File Write Operation

```
Application Code
  └─> agent.fs.writeFile('/report.pdf', buffer)
      └─> SDK (Filesystem)
          ├─> resolve_path('/report.pdf')
          │   └─> SELECT ino FROM fs_dentry WHERE parent_ino=1 AND name='report.pdf'
          │       (If not exists, create inode and dentry)
          │
          ├─> INSERT INTO fs_inode (mode, size, atime, mtime, ctime) VALUES (...)
          │   └─> Get new ino (e.g., 42)
          │
          ├─> INSERT INTO fs_dentry (name, parent_ino, ino) VALUES ('report.pdf', 1, 42)
          │
          ├─> INSERT INTO fs_data (ino, offset, size, data) VALUES (42, 0, ?, ?)
          │   └─> Store buffer as BLOB
          │
          └─> UPDATE fs_inode SET size=?, mtime=? WHERE ino=42
              └─> Transaction committed
```

### Data Flow 2: Tool Call Recording

```
Application Code
  └─> agent.tools.record('web_search', startTime, endTime, params, result)
      └─> SDK (ToolCalls)
          ├─> Calculate duration_ms = (endTime - startTime) * 1000
          │
          ├─> Serialize params to JSON
          ├─> Serialize result to JSON
          │
          └─> INSERT INTO tool_calls (
                  name, parameters, result, error,
                  started_at, completed_at, duration_ms
              ) VALUES ('web_search', ..., NULL, ?, ?, ?)
              └─> Returns auto-generated ID
```

### Data Flow 3: Sandboxed Syscall Interception

```
User Program
  └─> open("/agent/hello.txt", O_RDONLY)
      └─> Linux Kernel
          └─> Ptrace intercept (Reverie)
              └─> Sandbox::handle_syscall_event()
                  ├─> Get mount table
                  │   └─> Find "/agent" → SqliteVfs
                  │
                  ├─> SqliteVfs::open("/hello.txt", O_RDONLY)
                  │   └─> agentfs.fs.read_file("/hello.txt")
                  │       └─> SELECT data FROM fs_data WHERE ino=? ORDER BY offset
                  │           └─> Returns Vec<u8>
                  │
                  ├─> Create VirtualFile with content
                  ├─> Insert into FdTable → returns virtual FD (e.g., 3)
                  │
                  └─> Resume syscall with FD=3
                      └─> User program receives FD 3
                          └─> read(3, buffer, size) → Intercept again
                              └─> FdTable lookup → VirtualFile → Copy bytes
```

---

## Technology Stack

### Core Technologies

| Component | Technology | Version | Rationale |
|-----------|-----------|---------|-----------|
| Database | Turso (libSQL) | 0.3.2 | SQLite compatibility + extended features, local and remote support |
| Rust Runtime | Tokio | 1.x | Industry standard async runtime, full feature set |
| CLI Framework | Clap | 4.x | Derive macros for ergonomic CLI, excellent error messages |
| TypeScript Compiler | tsc | 5.3+ | Latest language features, strong type inference |
| Test Framework (TS) | Vitest | Latest | Fast, compatible with Jest, ESM support |
| Ptrace Framework | Reverie | Git (FB) | Production-tested syscall interception (used by Meta) |
| JSON Serialization | serde_json | 1.0 | De facto standard for Rust |
| Error Handling | anyhow | 1.0 | Ergonomic error propagation with context |

### Build & Distribution

| Tool | Purpose | Configuration |
|------|---------|---------------|
| Cargo | Rust build system | Workspace with 3 members |
| cargo-dist | Binary distribution | Cross-platform installers (v0.30.2) |
| npm | TypeScript package | Published to npmjs.com |
| GitHub Actions | CI/CD | Automated testing and releases |

**Supported Platforms**:
- Linux: x86_64 (full support with sandbox)
- macOS: ARM64 (Apple Silicon), x86_64 (Intel)
- Windows: x86_64 (SDK only, no sandbox)

---

## Regeneration Strategy

### Phase 1: Specification-First Rebuild

**Step 1**: Start with database schema (`SPEC.md`)
- Implement schema creation in Rust
- Verify with SQLite browser
- Write tests for each table

**Step 2**: Build Rust SDK (`sdk/rust`)
- Implement `KvStore` first (simplest)
- Implement `ToolCalls` second (insert-heavy)
- Implement `Filesystem` last (most complex)
- Test each module in isolation

**Step 3**: Build TypeScript SDK (`sdk/typescript`)
- Port Rust SDK API exactly
- Reuse test cases (translate Rust → TypeScript)
- Verify both SDKs produce same database state

**Step 4**: Build CLI (`cli`)
- Implement `init` command (uses SDK directly)
- Implement `fs` commands (direct SQL for performance)
- Defer `run` command until sandbox complete

**Step 5**: Build Sandbox (`sandbox`) - Linux only
- Implement VFS trait and mount table
- Implement BindVfs first (simpler)
- Implement SqliteVfs using SDK
- Implement syscall handlers incrementally
- Test with progressively complex programs (echo, cat, bash)

**Step 6**: Build Examples
- Start with simplest framework (OpenAI Agents)
- Add Claude Agent SDK example
- Add Mastra example
- Verify all examples share same integration pattern

### Phase 2: Test-First Development

**Unit Tests**:
- Each SDK module has comprehensive tests
- Use in-memory databases (`:memory:`)
- Cover happy path + edge cases + error paths

**Integration Tests**:
- CLI commands tested end-to-end
- Sandbox tested with real programs
- Cross-language compatibility (Rust SDK ↔ TypeScript SDK)

**Example Tests**:
- Each example runs successfully
- Verify agent.db contains expected data
- Performance benchmarks (tool call overhead, file write speed)

### Phase 3: Incremental Deployment

**For New Projects**:
- Use latest AgentFS directly
- Follow examples for integration

**For Existing Projects**:
- Add AgentFS alongside existing storage
- Migrate one feature at a time (e.g., tool tracking first)
- Gradual cutover with feature flags
- Validate equivalence before deprecating old system

---

## Improvement Opportunities

### Technical Improvements

#### 1. Replace Direct SQL with Prepared Statements
**Current**: Some CLI commands use raw SQL strings
**Proposed**: Use prepared statements throughout
**Benefits**:
- Prevent SQL injection (low risk, but good practice)
- Better performance (statement caching)
- Type safety with turso's query builder

**Location**: `cli/src/cmd/mod.rs:ls_filesystem`, `cli/src/cmd/mod.rs:cat_filesystem`

#### 2. Add Transaction Wrapper for Multi-Step Operations
**Current**: File writes are multi-step but not always in explicit transactions
**Proposed**:
```rust
impl Filesystem {
    async fn write_file_transactional(&self, path: &str, data: &[u8]) -> Result<()> {
        let tx = self.conn.transaction().await?;
        // ... multi-step operation
        tx.commit().await?;
    }
}
```
**Benefits**: Atomic operations, automatic rollback on error

#### 3. Add Caching Layer for Hot Paths
**Current**: Every operation queries database
**Proposed**: Add LRU cache for inode metadata, path resolution
**Benefits**: 10-100x faster for repeated reads
**Tradeoff**: Complexity, cache invalidation logic

#### 4. Compression for Large Files
**Current**: Files stored as raw BLOBs
**Proposed**: Optional compression (zstd or lz4) for fs_data
**Benefits**: 50-90% storage savings for text/PDFs
**Tradeoff**: CPU overhead, implementation complexity

---

### Architectural Improvements

#### 1. Add Migration Framework
**Proposed**: Alembic-style migrations for schema evolution
```
migrations/
  001_initial.sql
  002_add_compression.sql
  003_add_quotas.sql
```
**Benefits**: Safe schema upgrades, version tracking

#### 2. Plugin System for VFS Backends
**Proposed**: Dynamic loading of VFS implementations
```rust
trait VfsPlugin {
    fn name(&self) -> &str;
    fn create(&self, config: &Config) -> Box<dyn Vfs>;
}
```
**Benefits**:
- Community-contributed backends (S3, HTTP, etc.)
- No recompilation for new backends

#### 3. Event Sourcing for Filesystem Operations
**Proposed**: Record all filesystem events, not just current state
```sql
CREATE TABLE fs_events (
    id INTEGER PRIMARY KEY,
    event_type TEXT, -- 'create', 'write', 'delete'
    path TEXT,
    timestamp INTEGER,
    data BLOB
)
```
**Benefits**:
- Complete audit trail
- Time-travel debugging
- Replay for testing
**Tradeoff**: Storage overhead, complexity

---

### Operational Improvements

#### 1. CI/CD Pipeline with Automated Testing
**Current**: Manual testing
**Proposed**: GitHub Actions workflow
```yaml
- Run unit tests (Rust + TypeScript)
- Run integration tests
- Build binaries for all platforms
- Run examples against built CLI
- Publish to crates.io and npm
```

#### 2. Infrastructure as Code (Turso Deployment)
**Proposed**: Terraform/Pulumi for Turso databases
```hcl
resource "turso_database" "agent" {
  name   = "my-agent"
  group  = "production"
}
```
**Benefits**: Reproducible deployments, version control

#### 3. Monitoring Dashboards
**Proposed**: Grafana dashboards for:
- Tool call success rates
- Average durations by tool
- Database size growth
- Filesystem operation latency
**Benefits**: Production observability

#### 4. Automated Deployment (GitOps)
**Proposed**: ArgoCD or Flux for Kubernetes deployments
**Benefits**: Declarative, auditable, rollback support

---

## Key Architectural Decisions Justification

### ADR-1: Why SQLite Instead of PostgreSQL?

**Context**: Need persistent storage for agent state

**Decision**: Use SQLite (via Turso)

**Rationale**:
- **Portability**: Single file, no server required
- **Simplicity**: Zero configuration
- **ACID**: Same guarantees as PostgreSQL for single-writer
- **Embeddability**: Works in CLI, SDK, and sandbox
- **Turso**: Adds remote replication when needed

**Consequences**:
- ✅ Extreme simplicity for users
- ✅ Perfect for local development and testing
- ✅ Easy snapshotting (file copy)
- ❌ Not ideal for multi-agent concurrent writes
- ❌ No built-in replication (mitigated by Turso)

**Alternatives Considered**:
- PostgreSQL: ❌ Requires server, too heavy for local dev
- Redis: ❌ No durable filesystem semantics
- Plain files: ❌ No transactions, no SQL queries

### ADR-2: Why Inode-Based Filesystem?

**Context**: Need to store files with metadata

**Decision**: Use inode design (separate metadata from namespace)

**Rationale**:
- **Hard Links**: Multiple paths to same file (inode)
- **Efficient Renames**: Update dentry, not data
- **POSIX Compatibility**: Familiar to developers
- **Separation of Concerns**: Metadata, namespace, and data independent

**Consequences**:
- ✅ Full POSIX semantics
- ✅ Hard links work naturally
- ✅ Efficient for small files (no data duplication)
- ❌ More complex than path-based storage
- ❌ Path resolution requires JOIN queries

**Alternatives Considered**:
- Path-based: ❌ No hard links, inefficient renames
- Key-value: ❌ No directory semantics
- Flat storage: ❌ No hierarchy

### ADR-3: Why Reverie for Sandboxing?

**Context**: Need syscall interception on Linux

**Decision**: Use Facebook's Reverie ptrace framework

**Rationale**:
- **Production-Tested**: Used at Meta scale
- **Comprehensive**: Handles all syscall edge cases
- **Safe**: Rust-based, type-safe syscall handling
- **Flexible**: Easy to add new syscall handlers

**Consequences**:
- ✅ Robust syscall interception
- ✅ Minimal performance overhead (ptrace optimized)
- ✅ Rust safety guarantees
- ❌ Linux-only (ptrace limitation)
- ❌ Git dependency (not on crates.io)

**Alternatives Considered**:
- FUSE: ❌ Only intercepts filesystem, not all syscalls
- LD_PRELOAD: ❌ Bypassable, not secure
- Docker: ❌ Too heavy, not true isolation
- Custom ptrace: ❌ Complex, error-prone

### ADR-4: Why Dual-Language SDKs?

**Context**: Support multiple developer ecosystems

**Decision**: Maintain both Rust and TypeScript SDKs

**Rationale**:
- **Ecosystem Coverage**: Rust (systems) + TypeScript (web/Node.js)
- **AI Framework Diversity**: Some frameworks are TypeScript-only
- **Shared Database**: Both SDKs compatible via same schema
- **Type Safety**: Both languages provide strong typing

**Consequences**:
- ✅ Broad developer adoption
- ✅ Works with any AI framework
- ✅ Consistent API across languages
- ❌ Maintenance overhead (2x codebases)
- ❌ Risk of divergence (mitigated by tests)

**Alternatives Considered**:
- Rust only: ❌ Excludes TypeScript ecosystem
- TypeScript only: ❌ No systems-level performance
- C bindings: ❌ Unergonomic, error-prone

---

## Validation Checklist

This plan SUCCEEDS when:

✅ **Comprehensiveness**: All components (SDK, CLI, Sandbox) architecturally described
✅ **Clarity**: Diagrams and code examples make design understandable
✅ **Justification**: Architectural decisions explained with alternatives
✅ **Actionability**: Developers can follow regeneration strategy to rebuild
✅ **Improvement Roadmap**: Concrete suggestions for future enhancements

This plan NEEDS WORK when:

❌ Missing rationale for key decisions
❌ Unclear how components interact
❌ No guidance for rebuilding from scratch
❌ Improvement suggestions are vague

---

**Next Steps**:
1. Read `tasks.md` for step-by-step implementation guide
2. Read `intelligence-object.md` for reusable patterns and skills
3. Reference `spec.md` for complete requirements
4. Examine source code with this architectural understanding
