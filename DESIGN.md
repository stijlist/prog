# prog - A Modern macOS Package Manager

## Vision

`prog` is a next-generation package manager for macOS that combines the best features of existing tools:
- **Speed**: Fast like macports/pkgsrc
- **Time Travel**: Rollback capability like Nix
- **Modern Tooling**: Git-based package repository
- **Hermetic Builds**: Reproducible builds using zombiebuild (zb)
- **Smart Dependencies**: Minimum version selection with orphan cleanup

## Core Principles

1. **Git for Everything**: Package definitions and versions tracked in git
2. **Symlink-Based Versioning**: Atomic state transitions and rollback capability
3. **Hermetic Builds**: Reproducible, isolated build environments
4. **Fast Operations**: Optimized for instant search and efficient dependency resolution
5. **Minimal Overhead**: Reuse existing packages when possible (minimum version selection)

## Architecture

### Directory Structure

```
/opt/prog/
├── versions/              # All installed package versions
│   ├── nodejs/
│   │   ├── 18.17.0/      # Actual package files
│   │   └── 20.9.0/
│   └── python/
│       └── 3.11.5/
├── current/               # Symlinks to active versions
│   ├── nodejs -> ../versions/nodejs/20.9.0
│   └── python -> ../versions/python/3.11.5
├── profiles/              # Saved states for time travel
│   ├── default/          # Snapshot of current/ symlinks
│   ├── 2024-10-30/
│   └── pre-upgrade/
├── repo/                  # Git repository with package definitions
│   └── packages/
│       ├── nodejs.zb
│       ├── python.zb
│       └── ...
├── cache/                 # Build cache and downloads
│   ├── downloads/
│   └── build/
└── db/                    # Local state database
    └── state.db          # SQLite database
```

### Component Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     CLI Interface (prog)                     │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   Search     │   │  Dependency  │   │   Profile    │
│   Engine     │   │   Resolver   │   │   Manager    │
└──────────────┘   └──────────────┘   └──────────────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            ▼
                   ┌──────────────┐
                   │     State    │
                   │    Manager   │
                   └──────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  Git Repo    │   │  SQLite DB   │   │ Zombiebuild  │
│   Manager    │   │              │   │  Integration │
└──────────────┘   └──────────────┘   └──────────────┘
```

## Core Components

### 1. Package Repository (Git-Based)

**Location**: `/opt/prog/repo`

**Structure**:
```
packages/
├── nodejs.zb           # Zombiebuild definition
├── python.zb
└── metadata.json       # Package metadata for fast search
```

**Features**:
- Package definitions as zombiebuild files
- Version history tracked in git
- Fast cloning and updates via git pull
- Community contributions via git workflow (PRs)

**Package Definition Format** (example `nodejs.zb`):
```python
# zombiebuild definition for nodejs
package(
    name = "nodejs",
    version = "20.9.0",
    url = "https://nodejs.org/dist/v20.9.0/node-v20.9.0.tar.gz",
    sha256 = "...",
    dependencies = ["python", "icu4c", "openssl"],
    build_deps = ["gcc", "make"],
)

def build():
    configure("--prefix=/opt/prog/versions/nodejs/20.9.0")
    make("-j$(nproc)")
    make("install")
```

### 2. Search Engine

**Goal**: Sub-second search across all packages

**Implementation**:
- Build search index at `/opt/prog/db/search.idx` from repo
- Index structure:
  - Package name (primary key)
  - Description
  - Tags/categories
  - Dependencies
- Use simple grep-like search for speed (no heavy indexing)
- Rebuild index on repo update

**Command**: `prog search <query>`
- Search package names (fuzzy match)
- Search descriptions
- Return results sorted by relevance

### 3. Dependency Resolver

**Algorithm**: Minimum Version Selection (MVS)

**Features**:
- Direct dependencies: User-requested packages
- Transitive dependencies: Automatically resolved
- Version constraints: Minimum version requirements
- Conflict resolution: Use highest required version

**Data Structure** (in SQLite):
```sql
CREATE TABLE packages (
    name TEXT PRIMARY KEY,
    version TEXT NOT NULL,
    installed_at TIMESTAMP,
    is_direct BOOLEAN,  -- True if user-installed, False if dependency
    ref_count INTEGER   -- Number of packages depending on this
);

CREATE TABLE dependencies (
    package_name TEXT,
    depends_on TEXT,
    min_version TEXT,
    FOREIGN KEY (package_name) REFERENCES packages(name)
);
```

**Resolution Process**:
1. Parse requested package and its dependencies
2. Check installed packages for version satisfaction
3. Build dependency graph (detect cycles)
4. Select minimum versions that satisfy all constraints
5. Install missing packages in topological order

### 4. Profile Manager (Time Travel)

**Goal**: Instant rollback to previous states

**Implementation**:
- Profiles are snapshots of `/opt/prog/current/` symlinks
- Stored as lightweight metadata in `/opt/prog/profiles/`
- Atomic switching via symlink updates

**Profile Structure**:
```json
{
  "name": "2024-10-30-before-node-upgrade",
  "timestamp": "2024-10-30T10:30:00Z",
  "packages": {
    "nodejs": "18.17.0",
    "python": "3.11.5",
    "git": "2.42.0"
  }
}
```

**Commands**:
- `prog profile save <name>` - Save current state
- `prog profile list` - List saved profiles
- `prog profile restore <name>` - Restore previous state
- `prog profile diff <name1> <name2>` - Show differences

**Restore Process**:
1. Read target profile
2. Update all symlinks in `/opt/prog/current/`
3. Update SQLite database
4. Atomic operation (all or nothing)

### 5. State Manager

**Database**: SQLite at `/opt/prog/db/state.db`

**Responsibilities**:
- Track installed packages and versions
- Maintain dependency graph
- Reference counting for safe removal
- Transaction support for atomic operations

**Operations**:
- `add_package(name, version, is_direct)`
- `remove_package(name)`
- `get_dependencies(name)`
- `get_reverse_dependencies(name)`
- `is_orphan(name)` - True if ref_count == 0 and not direct

### 6. Build System Integration

**Tool**: Zombiebuild (zb)

**Features**:
- Hermetic builds (isolated environment)
- Reproducible (same inputs → same outputs)
- Parallel builds
- Build caching

**Build Process**:
1. Fetch source (cached in `/opt/prog/cache/downloads/`)
2. Verify checksum
3. Create hermetic build environment
4. Run zombiebuild script
5. Install to `/opt/prog/versions/<package>/<version>/`
6. Update symlinks in `/opt/prog/current/`
7. Update database

**Build Environment**:
- Isolated PATH (only prog-managed tools)
- Controlled environment variables
- No access to system libraries (except macOS SDK)
- Deterministic timestamps and file ordering

## Command Specifications

### `prog search <query>`

**Purpose**: Find packages matching query

**Algorithm**:
1. Load search index from `/opt/prog/db/search.idx`
2. Match query against package names and descriptions
3. Rank by relevance (exact match > prefix match > substring match)
4. Display top 20 results

**Output**:
```
nodejs          JavaScript runtime built on V8 engine
nodejs-lts      Long-term support version of Node.js
```

**Performance**: < 100ms for any query

### `prog add <package>`

**Purpose**: Install package and dependencies

**Algorithm**:
1. Resolve dependencies (build full dependency graph)
2. Check which packages are already installed at sufficient versions
3. Build list of packages to install
4. Download sources to cache
5. Build packages in topological order (parallelizable)
6. Install to `/opt/prog/versions/`
7. Update `/opt/prog/current/` symlinks
8. Update database (increment ref_counts)
9. Save automatic profile snapshot

**Options**:
- `--no-deps` - Don't install dependencies
- `--dry-run` - Show what would be installed
- `--version <ver>` - Install specific version

**Output**:
```
Resolving dependencies...
Will install:
  - openssl 3.1.0
  - icu4c 73.2
  - nodejs 20.9.0

Building openssl 3.1.0... ✓
Building icu4c 73.2... ✓
Building nodejs 20.9.0... ✓

Installed nodejs 20.9.0
Profile saved: auto-2024-10-30-10-30-00
```

### `prog remove <package>`

**Purpose**: Remove package and orphaned dependencies

**Algorithm**:
1. Check if package is installed
2. Get reverse dependencies (packages that depend on this)
3. If reverse deps exist, error (or --force to remove anyway)
4. Decrement ref_counts for all dependencies
5. Remove package from `/opt/prog/current/`
6. Remove from database (mark as direct=false if it's still a dep)
7. Find orphans (ref_count == 0 and not direct)
8. Recursively remove orphans
9. Save automatic profile snapshot

**Options**:
- `--keep-deps` - Don't remove orphaned dependencies
- `--force` - Remove even if other packages depend on it

**Output**:
```
Removing nodejs 20.9.0...
Checking for orphaned dependencies...
Will also remove:
  - icu4c 73.2 (no longer needed)

Removed nodejs 20.9.0
Removed icu4c 73.2
Profile saved: auto-2024-10-30-10-35-00
```

### Additional Commands

**Repository Management**:
- `prog update` - Update package repository (git pull)
- `prog upgrade` - Upgrade all installed packages
- `prog upgrade <package>` - Upgrade specific package

**Information**:
- `prog list` - List installed packages
- `prog info <package>` - Show package details
- `prog deps <package>` - Show dependency tree
- `prog rdeps <package>` - Show reverse dependencies

**Profile Management**:
- `prog profile save <name>` - Save current state
- `prog profile list` - List profiles
- `prog profile restore <name>` - Restore profile
- `prog profile delete <name>` - Delete profile

## Implementation Phases

### Phase 1: Core Infrastructure (MVP)
- [ ] Directory structure setup
- [ ] Git repository initialization
- [ ] SQLite database schema
- [ ] Basic CLI framework
- [ ] State manager implementation

### Phase 2: Package Operations
- [ ] Package definition format (zombiebuild integration)
- [ ] Dependency resolver (MVS algorithm)
- [ ] Build system integration
- [ ] Install/remove operations
- [ ] Symlink management

### Phase 3: Search & Discovery
- [ ] Search index builder
- [ ] Fast search implementation
- [ ] Package metadata format
- [ ] `prog search` command

### Phase 4: Time Travel
- [ ] Profile save/restore
- [ ] Automatic snapshots
- [ ] Profile diff
- [ ] Atomic switching

### Phase 5: Polish & Optimization
- [ ] Parallel builds
- [ ] Build caching
- [ ] Progress indicators
- [ ] Error handling & recovery
- [ ] Documentation

### Phase 6: Repository & Community
- [ ] Initial package collection (port from homebrew)
- [ ] Package submission workflow
- [ ] Automated testing
- [ ] Binary cache (optional)

## Technical Decisions

### Why Git?
- Industry standard, well-understood
- Excellent performance for package metadata
- Built-in distributed collaboration
- Efficient updates (only fetch changes)

### Why SQLite?
- Embedded, no server needed
- ACID transactions (atomic operations)
- Fast queries for dependency resolution
- Single-file database (easy backup)

### Why Zombiebuild?
- Hermetic builds (reproducibility)
- Modern, performant build system
- Parallel execution
- Good macOS support

### Why Minimum Version Selection?
- Predictable, stable dependency resolution
- Avoids "dependency hell"
- Used successfully by Go modules
- Simple to understand and debug

### Why Symlinks?
- Atomic updates (single operation)
- Multiple versions coexist peacefully
- Fast rollback (just update symlinks)
- No file copying needed

## Open Questions

1. **Binary Cache**: Should we support pre-built binaries?
   - Pro: Much faster installation
   - Con: Trust and security concerns
   - Possible solution: Optional, with signature verification

2. **macOS Versions**: How to handle different macOS versions?
   - Each package definition specifies min macOS version
   - Build on oldest supported version for compatibility

3. **System Libraries**: Can we depend on macOS system libraries?
   - Start conservative (bundle everything)
   - Optimize later (allow system deps for common libs)

4. **Package Naming**: How to handle multiple versions of same package?
   - Use `@` syntax: `prog add nodejs@18`
   - Default to latest stable

5. **Garbage Collection**: When to remove old package versions?
   - Manual: `prog clean --old-versions`
   - Automatic: Keep last N versions
   - Profile-aware: Keep versions referenced by profiles

## Success Metrics

- **Search Speed**: < 100ms for any query
- **Install Speed**: Comparable to macports (faster than homebrew)
- **Reliability**: Zero broken states (atomic operations)
- **Rollback Speed**: < 1 second to restore previous profile
- **Package Count**: 1000+ packages within first year

## Next Steps

1. Set up basic directory structure
2. Implement SQLite database schema
3. Create minimal CLI framework
4. Implement simple add/remove without builds
5. Integrate zombiebuild for real package builds
6. Port 10-20 essential packages as proof of concept
7. Implement search functionality
8. Add profile management
9. Optimize and polish
10. Launch beta with core packages
