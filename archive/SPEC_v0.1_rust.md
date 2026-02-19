# Open Games Platform - Specification v0.1

## Overview

Open Games is an open-source gaming platform that provides:
- A curated collection of open-source games
- Developer tools and SDKs for game development
- Community-driven game discovery and collaboration

## Core Components

### 1. Game Library
- Open-source game catalog with metadata
- Game discovery and search functionality
- Rating and review system
- Category/tag-based organization

### 2. Developer Tools
- **Game SDK**: Core libraries for game development
  - Input handling
  - Audio system
  - Asset management
  - Save/load system
- **Level Editor**: Visual editor for creating game content
- **Asset Pipeline**: Tools for importing and optimizing game assets
  - Sprite/texture processing
  - Audio conversion
  - Font generation
- **Debug Overlay**: In-game debugging tools
  - Performance metrics
  - Entity inspector
  - Console commands

### 3. Build System
- Cross-platform build support (Windows, Linux, macOS)
- Automated CI/CD pipelines
- Version management

## Technical Stack

- **Language**: Rust (core), TypeScript (web tools)
- **Graphics**: wgpu (WebGPU)
- **Audio**: rodio / cpal
- **Networking**: tokio, quinn (QUIC)
- **Storage**: SQLite for local data

## Architecture

```
open-games/
├── games/           # Open-source game collection
├── sdk/             # Core SDK libraries
├── tools/           # Developer tools
│   ├── editor/      # Level editor
│   ├── asset-pipe/  # Asset pipeline
│   └── debugger/    # Debug overlay
├── docs/            # Documentation
└── examples/        # Example games and templates
```

## Version Goals (v0.1)

- [ ] Basic SDK structure
- [ ] Simple game template
- [ ] Asset pipeline MVP
- [ ] Documentation skeleton

## License

MIT License - Open for community contributions
