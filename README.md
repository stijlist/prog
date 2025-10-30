# prog - A Modern Package Manager for macOS

**Status**: Planning Phase

## What is prog?

`prog` is a next-generation package manager for macOS that learns from the strengths and weaknesses of existing tools like Homebrew, MacPorts, Nix, and pkgsrc.

## Key Features

- **⚡ Fast**: Optimized for speed, with sub-second search and efficient dependency resolution
- **🔄 Time Travel**: Rollback to previous configurations instantly using profiles
- **🔒 Hermetic Builds**: Reproducible builds using zombiebuild (zb)
- **📦 Smart Dependencies**: Minimum version selection with automatic orphan cleanup
- **🌳 Git-Based**: Modern version control for package definitions

## Quick Start (Coming Soon)

```bash
# Search for packages
prog search nodejs

# Install a package and its dependencies
prog add nodejs

# Remove a package and orphaned dependencies
prog remove nodejs

# Save your current configuration
prog profile save working-state

# Restore a previous configuration
prog profile restore working-state
```

## Design Philosophy

### Problems with Existing Package Managers

| Manager   | Strengths              | Weaknesses                          |
|-----------|------------------------|-------------------------------------|
| Homebrew  | Most packages          | Slow, implementation quality issues |
| MacPorts  | Stable, fast           | No time travel, outdated VCS        |
| Nix       | Time travel, hermetic  | Complex, steep learning curve       |
| pkgsrc    | Stable, cross-platform | No time travel, outdated VCS        |

### Our Approach

1. **Git for Repository Versioning**: Modern, distributed, well-understood
2. **Symlinks for Local Versioning**: Atomic state transitions like Nix
3. **Zombiebuild for Builds**: Hermetic, reproducible build system
4. **Minimum Version Selection**: Predictable, stable dependency resolution
5. **Reference Counting**: Safe removal of orphaned dependencies

## Architecture Overview

```
/opt/prog/
├── versions/     # All installed package versions
├── current/      # Symlinks to active versions
├── profiles/     # Saved states for time travel
├── repo/         # Git repo with package definitions
├── cache/        # Build cache and downloads
└── db/           # SQLite database for state
```

## Documentation

- [Design Document](DESIGN.md) - Detailed architecture and design decisions
- [TODO](TODO.md) - Implementation roadmap and task breakdown

## Development Status

Currently in the planning phase. See [TODO.md](TODO.md) for the implementation roadmap.

**Phase 1** (MVP): Core infrastructure and database
**Phase 2**: Package operations and builds
**Phase 3**: Search functionality
**Phase 4**: Profile management (time travel)
**Phase 5**: Polish and optimization
**Phase 6**: Package repository and community

## Technology Stack

- **Language**: Rust (planned)
- **Database**: SQLite
- **Build System**: Zombiebuild (zb)
- **VCS**: Git
- **Package Repo**: Git-based

## Contributing

Not yet accepting contributions as we're still in the design phase. Stay tuned!

## License

TBD

## Comparison with Other Package Managers

### vs Homebrew
- ✅ Faster installation and search
- ✅ Time travel capability
- ✅ Hermetic builds
- ✅ Better dependency management

### vs Nix
- ✅ Simpler mental model
- ✅ macOS-focused (not cross-platform complexity)
- ✅ Familiar git-based workflow
- ⚖️ Similar time travel capability

### vs MacPorts
- ✅ Time travel capability
- ✅ Modern git-based repository
- ✅ Hermetic builds
- ⚖️ Similar stability and speed

## Roadmap

- **Q4 2024**: Complete design, start implementation
- **Q1 2025**: MVP with core functionality
- **Q2 2025**: Alpha release with 50+ packages
- **Q3 2025**: Beta release with 500+ packages
- **Q4 2025**: 1.0 release

## Contact

TBD
