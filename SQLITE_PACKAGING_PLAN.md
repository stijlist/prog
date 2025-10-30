# SQLite Packaging Plan for prog

## Overview

This document outlines the plan for creating the first package for `prog`: SQLite. This will serve as a reference implementation and establish patterns for future packages.

## Background Research

### About zb (zombiebuild)

**Key Characteristics:**
- Lua-based build configuration (similar to Nix but more accessible)
- Hermetic, reproducible builds with sandboxing
- Content-addressed storage (uses Nix .drv format internally)
- Cross-platform support (Linux, macOS, Windows)

**Basic Syntax:**
```lua
return derivation {
  name = "package-name";
  builder = "/bin/sh";
  system = "x86_64-darwin";  -- or "aarch64-darwin" for Apple Silicon
  args = {"-c", "build commands here"};
  -- dependencies would be specified here
}
```

**Key Functions:**
- `derivation {}` - Define a build derivation
- `path "..."` - Reference a file/directory with dependency tracking
- Strings carry dependency information automatically

### About SQLite

**Version:** Latest stable is 3.44.x (as of late 2024)

**Build Approaches:**

1. **Amalgamation Build (Recommended for simple cases)**
   - Single `sqlite3.c` file + `sqlite3.h` header
   - No configure script needed
   - Simple compilation: `gcc -dynamiclib sqlite3.c -o libsqlite3.dylib`
   - Minimal dependencies

2. **Autoconf Tarball (Recommended for prog)**
   - Standard `./configure && make && make install` workflow
   - More flexible, proper installation structure
   - Better integration with system conventions
   - Produces both library and CLI tool

**Dependencies:**
- **Build-time**:
  - C compiler (clang on macOS)
  - make
  - POSIX shell (/bin/sh)
  - Standard build tools (awk, sed, etc.)
- **Runtime**:
  - libm (math library - usually part of system)
  - pthread (threading - usually part of system)
  - No external dependencies (SQLite is self-contained)

**Build Process (Autoconf):**
```bash
./configure --prefix=/opt/prog/versions/sqlite/3.44.0 \
            --enable-fts5 \
            --enable-rtree \
            --enable-json1
make
make install
```

**macOS Specifics:**
- Universal binary support: `-arch x86_64 -arch arm64`
- Uses clang by default
- Need Xcode Command Line Tools

## Package Definition Design

### Directory Structure

```
/home/user/prog/packages/
└── sqlite/
    ├── sqlite.lua          # Main zb build definition
    ├── metadata.json       # Package metadata for search
    └── patches/            # Optional patches (if needed)
```

### Package Metadata Format

**File:** `packages/sqlite/metadata.json`

```json
{
  "name": "sqlite",
  "version": "3.44.0",
  "description": "A C library that implements a SQL database engine",
  "homepage": "https://www.sqlite.org/",
  "license": "Public Domain",
  "tags": ["database", "sql", "embedded", "library"],
  "categories": ["database", "development"],
  "maintainer": "prog maintainers",
  "build_dependencies": [],
  "runtime_dependencies": [],
  "provides": ["sqlite3", "libsqlite3"],
  "macOS_min_version": "10.13"
}
```

### zb Build Definition

**File:** `packages/sqlite/sqlite.lua`

We need to design this carefully. Here's a conceptual structure:

```lua
-- SQLite package definition for zb
-- Version: 3.44.0

local version = "3.44.0"
local year = "2023"  -- Update based on release
local version_number = "3440000"  -- SQLite's version numbering

-- Source URLs
local source_url = string.format(
  "https://www.sqlite.org/%s/sqlite-autoconf-%s.tar.gz",
  year, version_number
)

-- Download and verify source
local source = fetch {
  url = source_url;
  sha256 = "actual_hash_here";  -- Must be verified
}

-- Build derivation
return derivation {
  name = "sqlite-" .. version;
  system = "aarch64-darwin";  -- or x86_64-darwin

  builder = "/bin/sh";

  args = {
    "-c",
    [[
      set -e

      # Extract source
      tar xzf $source
      cd sqlite-autoconf-*

      # Configure
      ./configure \
        --prefix=$out \
        --enable-fts5 \
        --enable-rtree \
        --enable-json1 \
        --enable-session \
        --enable-dynamic-extensions \
        --disable-static

      # Build
      make -j$(sysctl -n hw.ncpu)

      # Install
      make install

      # Verify installation
      test -f $out/bin/sqlite3
      test -f $out/lib/libsqlite3.dylib
    ]]
  };

  source = source;

  -- Environment for hermetic build
  env = {
    PATH = "/usr/bin:/bin";  -- Minimal PATH
    CC = "clang";
    CFLAGS = "-O2";
  };
}
```

## Implementation Plan

### Phase 1: Setup Package Repository Structure

**Tasks:**
- [ ] Create `packages/` directory in prog repo
- [ ] Create `packages/sqlite/` subdirectory
- [ ] Set up basic directory structure
- [ ] Create README for package contributors

**Deliverables:**
```
packages/
├── README.md              # Guide for package maintainers
└── sqlite/
    └── (placeholder)
```

### Phase 2: Research and Preparation

**Tasks:**
- [ ] Determine exact SQLite version to package (latest stable)
- [ ] Download and verify SQLite source tarball manually
- [ ] Calculate correct SHA256 hash
- [ ] Test build manually on macOS to understand requirements
- [ ] Document any macOS-specific quirks or issues

**Deliverables:**
- Build notes documenting manual build process
- Verified SHA256 hash
- List of actual configure flags needed

### Phase 3: Create Package Metadata

**Tasks:**
- [ ] Create `metadata.json` with complete information
- [ ] Document all features enabled in the build
- [ ] List any system dependencies
- [ ] Define search tags and categories

**Deliverables:**
- `packages/sqlite/metadata.json`

### Phase 4: Write zb Build Definition

**Tasks:**
- [ ] Study zb documentation for `fetch` and `derivation` details
- [ ] Write basic `sqlite.lua` with source fetch
- [ ] Add configure step with appropriate flags
- [ ] Add build step with parallelization
- [ ] Add install step
- [ ] Add verification checks
- [ ] Test the build definition

**Challenges:**
- zb syntax and API might differ from assumptions
- Need to handle both x86_64 and aarch64 (universal binary?)
- May need to adjust for zb's hermetic environment

**Deliverables:**
- `packages/sqlite/sqlite.lua` (working build definition)

### Phase 5: Testing

**Tasks:**
- [ ] Test build in zb environment
- [ ] Verify output structure matches expectations
- [ ] Test installed sqlite3 binary works
- [ ] Test libsqlite3 library can be linked against
- [ ] Verify hermetic build (reproducibility)

**Test Cases:**
1. Build completes successfully
2. Output includes `bin/sqlite3` executable
3. Output includes `lib/libsqlite3.dylib` library
4. Output includes header files in `include/`
5. sqlite3 binary runs and shows correct version
6. Rebuilding produces identical output (reproducibility)

### Phase 6: Documentation

**Tasks:**
- [ ] Document the package structure
- [ ] Create template for future packages
- [ ] Write guide for package maintainers
- [ ] Document lessons learned

**Deliverables:**
- `packages/README.md` - Package maintainer guide
- `TEMPLATE.lua` - Template for new packages
- Notes on zb-specific considerations

## Open Questions and Decisions

### 1. Universal Binary Support

**Question:** Should we build universal binaries (x86_64 + arm64)?

**Options:**
- A) Single-architecture builds (simpler, faster)
- B) Universal binaries (better compatibility, larger)
- C) Separate packages per architecture

**Recommendation:** Start with single-architecture (A), add universal support later

**Reason:** Simpler for initial implementation; users typically only need one architecture

### 2. Configure Flags

**Question:** Which features should be enabled by default?

**Recommended Flags:**
```bash
--enable-fts5              # Full-text search
--enable-rtree             # R-Tree spatial indexing
--enable-json1             # JSON support
--enable-session           # Session extension
--enable-dynamic-extensions # Loadable extensions
--disable-static           # Only shared library
```

**Rationale:** Enable commonly-used features; match what users expect from modern SQLite

### 3. Version Pinning

**Question:** Should we package multiple versions of SQLite?

**Recommendation:** Start with latest stable only

**Future:** Support multiple versions with version specifiers (`sqlite@3.44`, `sqlite@3.43`)

### 4. Fetch Mechanism in zb

**Question:** How does zb's `fetch` function work exactly?

**Unknowns:**
- Exact syntax for fetch (might be different from assumption)
- How to specify SHA256 hash
- Whether zb handles HTTP downloads or needs wrapper
- Caching behavior

**Action:** Need to consult zb documentation or examples

### 5. Hermetic Environment

**Question:** What's available in zb's hermetic build environment?

**Needs Research:**
- Which system tools are available? (make, sed, awk, etc.)
- Can we use /usr/bin or is it isolated?
- How to specify build-time dependencies (e.g., if we need tcl)
- PATH and environment variable handling

**Action:** Test with minimal example first

### 6. Output Structure

**Question:** How does zb handle the `$out` variable and installation?

**Assumptions:**
- `$out` points to output directory
- Configure with `--prefix=$out` should work
- zb collects everything under `$out`

**Needs Verification:** Test with simple package first

## SQLite-Specific Considerations

### Optional Features

SQLite has many compile-time options. We should decide on a reasonable default:

**Enabled by default:**
- FTS5 (full-text search)
- JSON1 (JSON functions)
- R-Tree (spatial indexing)
- Session extension
- Dynamic extensions

**Not enabled by default:**
- ICU (international components - adds dependency)
- Soundex (rarely used)
- Geopoly (specialized)

### macOS Integration

**System SQLite:** macOS ships with SQLite, so ours should coexist:
- Install to `/opt/prog/` (not `/usr/local/`)
- Users add `/opt/prog/current/bin` to PATH
- Our version takes precedence when in PATH

### Testing Strategy

After building, we should verify:

```bash
# Version check
./bin/sqlite3 --version

# Basic functionality
./bin/sqlite3 :memory: "SELECT 1+1;"

# Extension check
./bin/sqlite3 :memory: "SELECT json_valid('{}');"

# FTS5 check
./bin/sqlite3 :memory: "CREATE VIRTUAL TABLE test USING fts5(content);"
```

## Next Steps (Immediate)

1. **Set up package directory structure** in the prog repo
2. **Manually build SQLite** on macOS to document the exact process
3. **Research zb syntax** - need actual examples or documentation access
4. **Create metadata.json** with all package information
5. **Write first draft** of `sqlite.lua` (even if syntax needs adjustment)
6. **Test the build** and iterate

## Success Criteria

This packaging effort is successful when:

1. ✅ `packages/sqlite/sqlite.lua` exists and is syntactically correct
2. ✅ zb can build SQLite from the definition
3. ✅ Build is reproducible (same inputs → same outputs)
4. ✅ Installed SQLite works correctly (passes test suite)
5. ✅ Package metadata is complete and accurate
6. ✅ Documentation explains the package structure
7. ✅ Template is created for future packages

## Timeline Estimate

- **Phase 1** (Setup): 1 hour
- **Phase 2** (Research): 2-3 hours
- **Phase 3** (Metadata): 1 hour
- **Phase 4** (Build definition): 3-4 hours
- **Phase 5** (Testing): 2-3 hours
- **Phase 6** (Documentation): 2 hours

**Total:** ~11-14 hours of focused work

## References

- SQLite Download: https://www.sqlite.org/download.html
- SQLite Compile Guide: https://www.sqlite.org/howtocompile.html
- zb GitHub: https://github.com/256lights/zb
- zb Documentation: https://zb.256lights.llc/
- Homebrew SQLite Formula: For reference on common build flags
