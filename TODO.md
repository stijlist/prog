# prog Implementation Roadmap

## Phase 1: Core Infrastructure (MVP)

### Directory Structure
- [ ] Create `/opt/prog/` root directory
- [ ] Set up subdirectories: `versions/`, `current/`, `profiles/`, `repo/`, `cache/`, `db/`
- [ ] Set proper permissions for multi-user support

### Database Schema
- [ ] Design SQLite schema for packages table
- [ ] Design SQLite schema for dependencies table
- [ ] Create migration system for schema updates
- [ ] Implement database connection pooling
- [ ] Add transaction support

### CLI Framework
- [ ] Set up argument parsing (clap or similar)
- [ ] Implement command routing (search, add, remove, etc.)
- [ ] Add global flags (--verbose, --dry-run, etc.)
- [ ] Create help text and usage documentation
- [ ] Implement colorized output for terminal

### State Manager
- [ ] Implement package addition to database
- [ ] Implement package removal from database
- [ ] Implement dependency graph queries
- [ ] Add reference counting logic
- [ ] Create orphan detection algorithm

## Phase 2: Package Operations

### Package Definition Format
- [ ] Define zombiebuild package spec format
- [ ] Create parser for .zb files
- [ ] Implement package metadata extraction
- [ ] Add version parsing and comparison
- [ ] Create validation for package definitions

### Dependency Resolver
- [ ] Implement Minimum Version Selection (MVS) algorithm
- [ ] Create dependency graph builder
- [ ] Add cycle detection
- [ ] Implement topological sort for build order
- [ ] Handle version constraints

### Build System Integration
- [ ] Create zombiebuild wrapper/interface
- [ ] Implement hermetic build environment setup
- [ ] Add source download and caching
- [ ] Implement checksum verification
- [ ] Create build parallelization system

### Install/Remove Operations
- [ ] Implement package installation flow
- [ ] Create symlink management for current/
- [ ] Add rollback on installation failure
- [ ] Implement package removal flow
- [ ] Add orphan cleanup on removal

## Phase 3: Search & Discovery

### Search Index
- [ ] Design search index format
- [ ] Implement index builder from git repo
- [ ] Add incremental index updates
- [ ] Create relevance ranking algorithm

### Search Implementation
- [ ] Implement fuzzy name matching
- [ ] Add description search
- [ ] Create tag/category filtering
- [ ] Optimize for <100ms response time
- [ ] Add search result formatting

### Package Metadata
- [ ] Define metadata.json format
- [ ] Create metadata extraction from .zb files
- [ ] Implement metadata validation
- [ ] Add metadata update on repo sync

## Phase 4: Time Travel (Profiles)

### Profile Save/Restore
- [ ] Implement profile save (snapshot current state)
- [ ] Create profile restore (revert to snapshot)
- [ ] Add profile listing
- [ ] Implement profile deletion
- [ ] Create profile diff functionality

### Automatic Snapshots
- [ ] Add pre-install snapshot
- [ ] Add post-install snapshot
- [ ] Implement snapshot retention policy
- [ ] Create snapshot naming convention

### Atomic Switching
- [ ] Implement atomic symlink updates
- [ ] Add transaction support for profile restore
- [ ] Create rollback on failure
- [ ] Ensure all-or-nothing semantics

## Phase 5: Polish & Optimization

### Build Performance
- [ ] Implement parallel package builds
- [ ] Add build artifact caching
- [ ] Create incremental build support
- [ ] Optimize download caching

### User Experience
- [ ] Add progress bars for long operations
- [ ] Implement detailed error messages
- [ ] Create recovery suggestions for common errors
- [ ] Add interactive prompts for dangerous operations

### Reliability
- [ ] Add comprehensive error handling
- [ ] Implement automatic recovery from interrupted operations
- [ ] Create integrity checking (fsck-like command)
- [ ] Add logging system for debugging

### Documentation
- [ ] Write user guide
- [ ] Create package maintainer guide
- [ ] Document API/library usage
- [ ] Add troubleshooting guide

## Phase 6: Repository & Community

### Package Collection
- [ ] Port top 50 homebrew packages
- [ ] Port top 100 homebrew packages
- [ ] Reach 500 packages
- [ ] Reach 1000 packages

### Contribution Workflow
- [ ] Create package submission template
- [ ] Implement automated testing for new packages
- [ ] Add CI/CD for package validation
- [ ] Create package review process

### Advanced Features
- [ ] Implement binary cache (optional)
- [ ] Add signature verification for packages
- [ ] Create package signing workflow
- [ ] Implement mirror support

## Immediate Next Steps (Week 1)

1. Set up Rust project structure with Cargo
2. Implement basic SQLite database with schema
3. Create CLI framework with argument parsing
4. Implement state manager (add/remove packages to DB)
5. Create directory structure initialization
6. Write unit tests for core components

## Key Milestones

- **M1**: Can install a package without dependencies (hardcoded, no build)
- **M2**: Can build a simple package using zombiebuild
- **M3**: Can resolve and install dependencies
- **M4**: Can search packages quickly
- **M5**: Can save/restore profiles
- **M6**: Alpha release with 50 packages
- **M7**: Beta release with 500 packages
- **M8**: 1.0 release with 1000+ packages
