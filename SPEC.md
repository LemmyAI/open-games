# Open Games - Specification v0.2

## Overview

Open Games is an open-source game platform where games are written in TypeScript/JavaScript and communicate entirely over LiveKit DataChannels. No WebSocket, no custom networking - just LiveKit for everything.

## Philosophy

- **One protocol** - All communication over LiveKit DataChannel
- **Browser-first** - Games run in browser or Electron
- **TypeScript** - Type-safe game development
- **LiveKit-native** - Voice, video, and data all through one SDK

## Repository Structure

```
open-games/
├── games/                  # Open game implementations
│   ├── pong/
│   ├── chess/
│   └── party-games/
├── sdk/                    # Game Development SDK
│   ├── core/               # Core game loop, state management
│   ├── livekit/            # LiveKit integration, protocols
│   ├── input/              # Input handling (keyboard, gamepad)
│   ├── renderer/           # 2D/3D rendering (WebGL, WebGPU)
│   └── audio/              # Audio engine
├── protocols/              # Protocol definitions
│   ├── game.proto          # Game state/input protocol
│   ├── chat.proto          # Chat protocol
│   ├── match.proto         # Matchmaking protocol
│   └── voice.proto         # Voice events
├── devtools/               # Developer tools
│   ├── asset-packer/       # Asset bundling
│   ├── game-compiler/      # Build system
│   ├── debugger/           # In-game debug overlay
│   └── templates/          # Game starter templates
├── docs/                   # Documentation
└── examples/               # Example games
```

## SDK Components

### @opengames/sdk-core
Core game engine functionality:
- Game loop (fixed timestep, variable render)
- State management
- Entity-component system
- Scene management

### @opengames/sdk-livekit
LiveKit integration:
- Room connection
- DataChannel management
- Protocol encoding/decoding
- Participant handling

```typescript
import { Game, DataChannel } from '@opengames/sdk';

class MyGame extends Game {
  async onJoin(roomId: string, token: string) {
    // Connect to LiveKit room
    await this.connect(roomId, token);
    
    // Create reliable channel for events
    this.eventChannel = this.createDataChannel('events', {
      ordered: true,
      reliable: true
    });
    
    // Create unreliable channel for fast updates
    this.stateChannel = this.createDataChannel('state', {
      ordered: false,
      reliable: false
    });
  }
  
  onPlayerInput(input: PlayerInput) {
    // Send unreliably for speed
    this.stateChannel.send(encodePlayerInput(input));
  }
}
```

### @opengames/sdk-input
Input handling:
- Keyboard state
- Gamepad support
- Touch controls
- Input mapping

### @opengames/sdk-renderer
Rendering:
- Sprite batch rendering
- WebGL2 backend
- WebGPU (when available)
- Particle systems
- Shader management

### @opengames/sdk-audio
Audio engine:
- Sound effects
- Music streaming
- 3D positional audio
- Voice integration (via LiveKit)

## Protocol Definitions

### Game Protocol
```protobuf
syntax = "proto3";

package opengames;

// Player input (sent frequently, unreliable)
message PlayerInput {
  uint32 sequence = 1;
  uint32 timestamp = 2;
  repeated InputAction actions = 3;
}

message InputAction {
  uint32 action_id = 1;    // JUMP, SHOOT, MOVE, etc.
  float value = 2;          // 0.0 - 1.0 or -1.0 - 1.0
  float x = 3;              // Direction/vector
  float y = 4;
}

// Game state (broadcast by authority)
message GameState {
  uint32 tick = 1;
  uint32 timestamp = 2;
  repeated EntityState entities = 3;
}

message EntityState {
  uint32 entity_id = 1;
  float x = 2;
  float y = 3;
  float rotation = 4;
  uint32 state_flags = 5;   // Animation, status effects
}
```

### Chat Protocol
```protobuf
syntax = "proto3";

package opengames;

message ChatMessage {
  string from_participant = 1;
  string text = 2;
  uint64 timestamp = 3;
  ChatType type = 4;
}

enum ChatType {
  CHAT_ROOM = 0;
  CHAT_PRIVATE = 1;
  CHAT_SYSTEM = 2;
}
```

### Match Protocol
```protobuf
syntax = "proto3";

package opengames;

message MatchEvent {
  oneof event {
    MatchJoin join = 1;
    MatchLeave leave = 2;
    MatchStart start = 3;
    MatchEnd end = 4;
    PlayerList players = 5;
  }
}

message MatchJoin {
  string participant_id = 1;
  string display_name = 2;
  PlayerStats stats = 3;
}

message PlayerList {
  repeated PlayerInfo players = 1;
}
```

## Game Format

Games are packaged as modules:

```
my-game/
├── game.json           # Game manifest
├── index.ts            # Game entry point
├── assets/
│   ├── sprites/
│   ├── audio/
│   └── fonts/
└── scenes/
    ├── menu.ts
    ├── game.ts
    └── results.ts
```

### game.json
```json
{
  "id": "com.example.mygame",
  "name": "My Awesome Game",
  "version": "1.0.0",
  "minPlayers": 2,
  "maxPlayers": 8,
  "gameMode": "competitive|cooperative|party",
  "requiresAuthority": true,
  "tickRate": 20,
  "protocols": ["game", "chat"]
}
```

## DataChannel Types

| Channel | Mode | Use Case |
|---------|------|----------|
| `state` | Unreliable, unordered | Player positions, movement |
| `events` | Reliable, ordered | Game events, chat |
| `input` | Unreliable, ordered | Player input |
| `voice` | Built-in LiveKit | Voice chat |
| `video` | Built-in LiveKit | Video streams |

## Integration with GameServer

1. **Get LiveKit token** from GameServer API (HTTP)
2. **Connect to LiveKit room** using token
3. **All game communication** happens over DataChannels
4. **Server is a participant** - can be authoritative or observer

```
┌─────────────┐
│  GameServer │ ←── HTTP: auth, stats, leaderboards
│    API      │
└──────┬──────┘
       │ issues token
       ▼
┌─────────────┐
│   LiveKit   │ ←── All game traffic
│   Cloud     │
└──────┬──────┘
       │
  ┌────┼────┐
  ▼    ▼    ▼
[P1] [P2] [Server]
```

## DevTools

### Asset Packer
- Texture atlas generation
- Audio encoding (OGG, MP3, WAV)
- Font bitmap generation
- JSON manifest output

### Game Compiler
- TypeScript → JavaScript bundling
- Tree shaking for smaller bundles
- Source maps for debugging
- Hot reload for development

### Debugger
- Live state inspector
- Network traffic monitor (DataChannel)
- Performance overlay
- Entity inspector

### Templates
```bash
# Create new game from template
npx create-open-game --template=2d-multiplayer my-game

# Available templates
- 2d-multiplayer    # 2D competitive game
- 3d-multiplayer    # 3D game with WebGL
- party-game        # Turn-based party game
- puzzle-game       # Cooperative puzzle
```

## Tech Stack

- **Language**: TypeScript
- **Runtime**: Browser / Bun / Electron
- **Rendering**: WebGL2 / WebGPU
- **Audio**: Web Audio API
- **Networking**: LiveKit SDK
- **Protocol**: Protocol Buffers / MessagePack

## Version Goals

### v0.2 (Current)
- [ ] SDK core with game loop
- [ ] LiveKit integration
- [ ] Protocol definitions
- [ ] Basic 2D rendering
- [ ] One example game (Pong)

### v0.3
- [ ] Asset pipeline
- [ ] Input system
- [ ] Audio system
- [ ] DevTools CLI

### v1.0
- [ ] Full SDK
- [ ] 3+ example games
- [ ] Documentation
- [ ] Templates

## License

MIT License - Open for community contributions

## Version History

- **v0.2** (2025): Unified LiveKit-only architecture
- **v0.1** (Archived): Initial Rust/Node specs
