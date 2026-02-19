# Open Games - Specification v0.3

## Overview

Open Games is a **LiveKit networking plugin for Phaser** (and other game frameworks). We provide the multiplayer layer - you bring the game engine.

**Stop rebuilding game engines.** Focus on what's unique: making LiveKit the best multiplayer transport for web games.

## Philosophy

- **Plugin, not platform** - Enhance existing engines, don't replace them
- **LiveKit for networking** - Voice, video, and game state in one SDK
- **Get working fast** - Multiplayer in < 20 lines of code
- **Validate first** - Prove the approach before expanding

## What We Build

```
@opengames/livekit-phaser    # Phaser 3 plugin
@opengames/livekit-three     # Three.js helper (future)
@opengames/protocols         # Shared protocol definitions
@opengames/cli               # Dev tools

NOT building:
- Game engine (Phaser exists)
- Physics engine (Matter.js, Rapier exist)
- Rendering engine (PixiJS, Three.js exist)
- Asset pipeline (Vite, esbuild exist)
```

## Quick Start

```typescript
import Phaser from 'phaser';
import { LiveKitPlugin, createRoom } from '@opengames/livekit-phaser';

// Your game config
const config = {
  type: Phaser.AUTO,
  width: 800,
  height: 600,
  scene: GameScene
};

// Add multiplayer - that's it!
const game = new Phaser.Game(config);
game.plugins.install('LiveKitPlugin', LiveKitPlugin, true);

// In your scene
class GameScene extends Phaser.Scene {
  create() {
    // Connect to room
    this.livekit = this.plugins.get('LiveKitPlugin');
    this.livekit.connect('my-room', token);
    
    // Sync player position automatically
    this.livekit.syncPosition(this.player, { 
      interpolate: true,
      prediction: true 
    });
  }
}
```

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Your Game                         │
│   (Phaser scene, sprites, game logic)               │
└─────────────────────┬───────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────┐
│            @opengames/livekit-phaser                 │
│                                                      │
│   ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │
│   │ Position    │  │ State       │  │ Voice/     │ │
│   │ Sync        │  │ Sync        │  │ Video      │ │
│   │ (Interp)    │  │ (Reliable)  │  │ (Built-in) │ │
│   └──────┬──────┘  └──────┬──────┘  └─────┬──────┘ │
│          │                │                │        │
│   ┌──────▼────────────────▼────────────────▼──────┐│
│   │              LiveKit SDK                       ││
│   │         (DataChannels + Media)                ││
│   └───────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
                      │
              ┌───────▼───────┐
              │  LiveKit Cloud │
              │  or Self-Host  │
              └───────────────┘
```

## Package Structure

```
open-games/
├── packages/
│   ├── livekit-phaser/        # Phaser 3 plugin
│   │   ├── src/
│   │   │   ├── Plugin.ts      # Main plugin class
│   │   │   ├── PositionSync.ts
│   │   │   ├── StateSync.ts
│   │   │   └── index.ts
│   │   └── package.json
│   │
│   ├── protocols/             # Shared protocols
│   │   ├── proto/
│   │   │   ├── position.proto
│   │   │   └── state.proto
│   │   ├── src/
│   │   │   ├── encode.ts
│   │   │   └── decode.ts
│   │   └── package.json
│   │
│   └── cli/                   # Dev tools
│       ├── src/
│       │   ├── dev.ts         # Local dev server
│       │   └── token.ts       # Generate test tokens
│       └── package.json
│
├── examples/
│   ├── pong/                  # Minimal Pong
│   └── movement/              # Just two squares moving
│
└── docs/
    └── getting-started.md
```

## Core Features

### 1. Position Sync

Automatic position synchronization with interpolation and prediction:

```typescript
// packages/livekit-phaser/src/PositionSync.ts

export interface PositionSyncOptions {
  interpolate?: boolean;    // Smooth other players
  prediction?: boolean;     // Predict local movement
  sendRate?: number;        // Updates per second (default: 20)
  deadReckoning?: boolean;  // Continue movement during lag
}

export class PositionSync {
  private localPlayer: Phaser.GameObjects.Sprite;
  private remotePlayers: Map<string, RemotePlayer>;
  private unconfirmedInputs: Input[] = [];
  
  constructor(
    private plugin: LiveKitPlugin,
    private options: PositionSyncOptions = {}
  ) {}
  
  // Sync this sprite's position to other players
  sync(sprite: Phaser.GameObjects.Sprite, playerId: string) {
    this.localPlayer = sprite;
    
    // Listen for position updates from others
    this.plugin.onData('position', (data, from) => {
      this.updateRemotePlayer(from, data);
    });
  }
  
  update() {
    if (!this.localPlayer) return;
    
    // Send local position (unreliable, fast)
    if (this.shouldSend()) {
      this.plugin.sendUnreliable('position', {
        x: this.localPlayer.x,
        y: this.localPlayer.y,
        rotation: this.localPlayer.rotation,
        timestamp: Date.now()
      });
    }
    
    // Interpolate remote players
    if (this.options.interpolate) {
      this.interpolateRemotes();
    }
  }
  
  // Client-side prediction
  predict(input: Input) {
    // Apply locally immediately
    this.localPlayer.x += input.dx;
    this.localPlayer.y += input.dy;
    
    // Store for reconciliation
    this.unconfirmedInputs.push(input);
    
    // Send to server
    this.plugin.sendUnreliable('input', input);
  }
  
  // Server correction
  reconcile(authoritativePos: Position) {
    // Snap to server position
    this.localPlayer.x = authoritativePos.x;
    this.localPlayer.y = authoritativePos.y;
    
    // Re-apply unconfirmed inputs
    for (const input of this.unconfirmedInputs) {
      this.localPlayer.x += input.dx;
      this.localPlayer.y += input.dy;
    }
    
    // Clear confirmed
    this.unconfirmedInputs = this.unconfirmedInputs.filter(
      i => i.sequence > authoritativePos.lastSequence
    );
  }
}
```

### 2. State Sync

Reliable state for game events, scores, health:

```typescript
// packages/livekit-phaser/src/StateSync.ts

export class StateSync {
  constructor(private plugin: LiveKitPlugin) {}
  
  // Sync game state reliably
  set(key: string, value: any) {
    this.plugin.sendReliable('state', { key, value });
  }
  
  // Listen for state changes
  on(key: string, callback: (value: any) => void) {
    this.plugin.onData('state', (data) => {
      if (data.key === key) callback(data.value);
    });
  }
  
  // Request full state (for late joiners)
  requestFullState() {
    this.plugin.sendReliable('request-state', {});
  }
}
```

### 3. Room Management

```typescript
// packages/livekit-phaser/src/Plugin.ts

export class LiveKitPlugin extends Phaser.Plugins.BasePlugin {
  private room: Room | null = null;
  private dataChannels: Map<string, DataChannel> = new Map();
  
  async connect(roomName: string, token: string) {
    this.room = new Room();
    
    // Connect to LiveKit
    await this.room.connect('wss://your-livekit-server.com', token);
    
    // Create data channels
    await this.setupDataChannels();
    
    // Ready!
    this.emit('connected');
  }
  
  private async setupDataChannels() {
    // Unreliable for positions
    const positionChannel = await this.room.localParticipant.publishData(
      new Uint8Array(),
      DataPacket_Kind.LOSSY
    );
    this.dataChannels.set('position', positionChannel);
    
    // Reliable for state
    const stateChannel = await this.room.localParticipant.publishData(
      new Uint8Array(),
      DataPacket_Kind.RELIABLE
    );
    this.dataChannels.set('state', stateChannel);
  }
  
  sendUnreliable(topic: string, data: any) {
    const channel = this.dataChannels.get(topic);
    if (channel) {
      channel.send(this.encode(topic, data));
    }
  }
  
  sendReliable(topic: string, data: any) {
    this.room.localParticipant.publishData(
      this.encode(topic, data),
      DataPacket_Kind.RELIABLE
    );
  }
  
  onData(topic: string, callback: (data: any, from: string) => void) {
    this.room.on('data-received', (payload, participant) => {
      const decoded = this.decode(payload);
      if (decoded.topic === topic) {
        callback(decoded.data, participant.identity);
      }
    });
  }
  
  // Voice/video built-in!
  async publishMicrophone() {
    await this.room.localParticipant.setMicrophoneEnabled(true);
  }
  
  async publishCamera() {
    await this.room.localParticipant.setCameraEnabled(true);
  }
}
```

## Protocol Definitions

```protobuf
// packages/protocols/proto/position.proto
syntax = "proto3";

package opengames;

message Position {
    uint32 sequence = 1;
    uint64 timestamp = 2;
    string player_id = 3;
    float x = 4;
    float y = 5;
    float rotation = 6;
    float velocity_x = 7;
    float velocity_y = 8;
}

// packages/protocols/proto/state.proto
message StateUpdate {
    string key = 1;
    bytes value = 2;
    uint64 timestamp = 3;
}

message Input {
    uint32 sequence = 1;
    uint64 timestamp = 2;
    float dx = 3;
    float dy = 4;
    repeated uint32 actions = 5;
}
```

## Examples

### Example 1: Movement Only (Simplest)

Just two squares that can see each other move:

```typescript
// examples/movement/src/main.ts
import Phaser from 'phaser';
import { LiveKitPlugin } from '@opengames/livekit-phaser';

const config = {
  type: Phaser.AUTO,
  width: 800,
  height: 600,
  scene: MovementScene
};

const game = new Phaser.Game(config);
game.plugins.install('LiveKitPlugin', LiveKitPlugin, true);

class MovementScene extends Phaser.Scene {
  create() {
    // My square
    this.me = this.add.rectangle(400, 300, 32, 32, 0x00ff00);
    this.cursors = this.input.keyboard.createCursorKeys();
    
    // Other players
    this.others = {};
    
    // Connect
    const token = getYourToken(); // From your backend
    const livekit = this.plugins.get('LiveKitPlugin');
    livekit.connect('room-1', token);
    
    // Sync my position
    livekit.onData('position', (data, from) => {
      if (!this.others[from]) {
        this.others[from] = this.add.rectangle(0, 0, 32, 32, 0xff0000);
      }
      this.others[from].setPosition(data.x, data.y);
    });
  }
  
  update() {
    const speed = 5;
    let moved = false;
    
    if (this.cursors.left.isDown) { this.me.x -= speed; moved = true; }
    if (this.cursors.right.isDown) { this.me.x += speed; moved = true; }
    if (this.cursors.up.isDown) { this.me.y -= speed; moved = true; }
    if (this.cursors.down.isDown) { this.me.y += speed; moved = true; }
    
    if (moved) {
      const livekit = this.plugins.get('LiveKitPlugin');
      livekit.sendUnreliable('position', {
        x: this.me.x,
        y: this.me.y
      });
    }
  }
}
```

### Example 2: Pong

Full Pong game with scoring:

```typescript
// examples/pong/src/main.ts
import Phaser from 'phaser';
import { LiveKitPlugin } from '@opengames/livekit-phaser';

class PongScene extends Phaser.Scene {
  create() {
    // Game objects
    this.paddle1 = this.add.rectangle(50, 300, 16, 100, 0xffffff);
    this.paddle2 = this.add.rectangle(750, 300, 16, 100, 0xffffff);
    this.ball = this.add.rectangle(400, 300, 16, 16, 0xffffff);
    
    this.score1 = this.add.text(200, 50, '0', { fontSize: '48px' });
    this.score2 = this.add.text(600, 50, '0', { fontSize: '48px' });
    
    // Connect
    const livekit = this.plugins.get('LiveKitPlugin');
    livekit.connect('pong-' + this.gameId, this.token);
    
    // I'm player 1 or 2?
    livekit.on('player-assigned', (num) => {
      this.myPaddle = num === 1 ? this.paddle1 : this.paddle2;
      this.myY = num === 1 ? 'paddle1Y' : 'paddle2Y';
    });
    
    // Sync paddle positions
    livekit.onData('position', (data) => {
      if (data.paddle === 1) this.paddle1.y = data.y;
      if (data.paddle === 2) this.paddle2.y = data.y;
    });
    
    // Sync ball (player 1 is authoritative)
    livekit.onData('ball', (data) => {
      this.ball.x = data.x;
      this.ball.y = data.y;
      this.ball.vx = data.vx;
      this.ball.vy = data.vy;
    });
    
    // Sync score (reliable)
    livekit.onData('state', (data) => {
      if (data.key === 'score') {
        this.score1.setText(data.value.p1);
        this.score2.setText(data.value.p2);
      }
    });
  }
  
  update() {
    // Move my paddle
    if (this.cursors.up.isDown) this.myPaddle.y -= 5;
    if (this.cursors.down.isDown) this.myPaddle.y += 5;
    
    // Send position
    livekit.sendUnreliable('position', {
      paddle: this.myPaddle === this.paddle1 ? 1 : 2,
      y: this.myPaddle.y
    });
  }
}
```

## CLI Tools

```bash
# Start local dev with LiveKit
npx @opengames/cli dev

# Generate test token for testing
npx @opengames/cli token --room my-room --player player1

# Run example
npx @opengames/cli example movement
```

## Success Metrics (v0.3)

- [ ] Two players can see each other move in movement example
- [ ] Latency feels acceptable (< 100ms perceived lag)
- [ ] Pong is playable
- [ ] Voice chat works in examples
- [ ] Documentation gets someone started in < 5 minutes

## Tech Stack

| Component | Technology |
|-----------|------------|
| **Game Engine** | Phaser 3 (user provides) |
| **Networking** | LiveKit SDK |
| **Protocols** | Protocol Buffers (optional, JSON first) |
| **Build** | Vite |
| **Package** | Turborepo |

## Version Goals

### v0.3 (Current) - Validate the Concept
- [ ] LiveKit plugin for Phaser
- [ ] Position sync with interpolation
- [ ] Movement example (two squares)
- [ ] Pong example
- [ ] Basic CLI

### v0.4 - Make it Usable
- [ ] Client prediction
- [ ] State sync API
- [ ] Better docs
- [ ] Token generation helper

### v1.0 - Production Ready
- [ ] Three.js support
- [ ] Performance optimizations
- [ ] Full protocol buffer support
- [ ] Server authority helper

## What We're NOT Building

| Feature | Why Not | Use Instead |
|---------|---------|-------------|
| Game engine | Phaser, PixiJS exist | Phaser 3 |
| Physics | Matter.js, Rapier exist | Matter.js |
| Asset pipeline | Vite, esbuild exist | Vite |
| UI framework | React, Vue exist | Your choice |
| Animation system | Phaser has one | Phaser's |
| 3D renderer | Three.js exists | Three.js |

## Key Changes from v0.2

| Aspect | v0.2 | v0.3 |
|--------|------|------|
| **Scope** | Full game platform | Phaser plugin |
| **Engine** | Custom SDK | Use Phaser |
| **Focus** | Build everything | Just networking |
| **First goal** | Full SDK | Two moving squares |
| **Physics** | Custom | Use existing |
| **Rendering** | Custom WebGL | Use Phaser |

## License

MIT License

## Version History

- **v0.3** (2026): Focus on Phaser plugin, validate LiveKit approach
- **v0.2** (2026): Full game platform (archived - too ambitious)
- **v0.1** (Archived): Initial Rust/Node specs
