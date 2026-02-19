# Open Games Build Plan

## Phase 1: Plugin Foundation (Week 1-2)

### Goal: Phaser plugin that connects to LiveKit

### Day 1-2: Project Setup
- [ ] Initialize Turborepo monorepo
- [ ] Create `packages/livekit-phaser/` structure
- [ ] Set up TypeScript + Vite build
- [ ] Add Phaser 3 as peer dependency

### Day 3-5: Core Plugin
- [ ] Create `LiveKitPlugin` class extending `Phaser.Plugins.BasePlugin`
- [ ] Basic `connect(roomName, token)` method
- [ ] Handle LiveKit Room connection
- [ ] Emit connection events

### Day 6-7: DataChannel Setup
- [ ] Create unreliable DataChannel for positions
- [ ] Create reliable DataChannel for state
- [ ] Basic `sendUnreliable()` and `sendReliable()` methods
- [ ] Test with LiveKit Cloud dev account

### Day 8-9: Rate Limiting + States
- [ ] Add rate limiting to `sendUnreliable()` (16ms min)
- [ ] Implement `ConnectionState` enum
- [ ] Emit `connection-state` events
- [ ] Add connection state to plugin

### Day 10: Testing
- [ ] Unit tests with Jest
- [ ] Manual test in example scene
- [ ] Verify connection/disconnect flow

### Deliverable
```typescript
// This works:
const game = new Phaser.Game(config);
game.plugins.install('LiveKitPlugin', LiveKitPlugin, true);

const livekit = game.plugins.get('LiveKitPlugin');
await livekit.connect('test-room', token);
// Connected!
```

---

## Phase 2: Movement Example (Week 3-4)

### Goal: Two squares that see each other move

### Day 11-13: Example Setup
- [ ] Create `examples/movement/` directory
- [ ] Simple HTML + Phaser scene
- [ ] Token generation helper (CLI or hardcoded for dev)
- [ ] Local LiveKit server or cloud

### Day 14-16: Local Movement
- [ ] Green square controlled by keyboard
- [ ] Arrow keys = movement
- [ ] Position updated locally

### Day 17-19: Remote Player Sync
- [ ] Red square for other players
- [ ] `onData('position')` listener
- [ ] Update remote square position
- [ ] Basic interpolation (lerp)

### Day 20-21: Send Position
- [ ] Send position on every move
- [ ] Rate limited to 20Hz
- [ ] Include timestamp
- [ ] Test with 2 browser tabs

### Deliverable
```bash
# Open two browser tabs
# Tab 1: Green square moves with arrow keys
# Tab 2: Sees red square move (Tab 1's position)
# Vice versa
```

---

## Phase 3: Position Sync (Week 5-6)

### Goal: Smooth movement with prediction

### Day 22-24: PositionSync Class
- [ ] Create `PositionSync` class
- [ ] `sync(sprite, playerId)` method
- [ ] Automatic position sending
- [ ] Interpolation for remotes

### Day 25-27: Interpolation
- [ ] Buffer remote positions
- [ ] Linear interpolation between positions
- [ ] Handle network jitter
- [ ] Configurable interpolation delay (50-100ms)

### Day 28-30: Object Pooling
- [ ] Create `RemotePlayerPool` class
- [ ] Pre-allocate sprites
- [ ] Recycle on player leave
- [ ] Test with multiple players joining/leaving

### Day 31-32: Polish
- [ ] Smooth movement at various latencies
- [ ] Handle 50ms, 100ms, 200ms latency
- [ ] Test packet loss scenarios

### Deliverable
```typescript
// One line to sync:
this.livekit.syncPosition(this.player, { interpolate: true });

// Movement is smooth even with 100ms latency
```

---

## Phase 4: Pong Example (Week 7-8)

### Goal: Playable Pong game

### Day 33-35: Pong Setup
- [ ] Create `examples/pong/`
- [ ] Two paddles, one ball
- [ ] Score display
- [ ] Basic collision

### Day 36-38: Multiplayer Paddles
- [ ] Player 1 controls left paddle
- [ ] Player 2 controls right paddle
- [ ] Paddle positions synced

### Day 39-40: Ball Authority
- [ ] Player 1 is "host" (first to join)
- [ ] Host controls ball position
- [ ] Broadcast ball to player 2
- [ ] Score sync via reliable channel

### Day 41-42: Polish
- [ ] Predictive ball movement
- [ ] Score persistence
- [ ] Win/lose detection
- [ ] Voice chat optional

### Deliverable
```bash
# Two players can play Pong
# Ball is smooth, paddles responsive
# Scores update correctly
```

---

## Phase 5: CLI + Docs (Week 9-10)

### Goal: Developer-friendly package

### Day 43-45: CLI Tool
- [ ] `@opengames/cli` package
- [ ] `npx @opengames/cli token --room test`
- [ ] `npx @opengames/cli dev` (local LiveKit)
- [ ] `npx @opengames/cli example movement`

### Day 46-48: Documentation
- [ ] Getting started guide
- [ ] API reference
- [ ] Example walkthroughs
- [ ] Troubleshooting guide

### Day 49-50: NPM Package
- [ ] Package.json setup
- [ ] TypeScript declarations
- [ ] Publish to npm (beta)
- [ ] Test install in fresh project

### Deliverable
```bash
npm install @opengames/livekit-phaser

# In your Phaser game:
import { LiveKitPlugin } from '@opengames/livekit-phaser';
// Multiplayer in 5 minutes
```

---

## Success Metrics

| Milestone | Success Criteria |
|-----------|------------------|
| Phase 1 | Plugin connects to LiveKit |
| Phase 2 | Two squares see each other move |
| Phase 3 | Smooth movement at 100ms latency |
| Phase 4 | Pong is playable |
| Phase 5 | Published to npm, docs complete |

## Dependencies

```json
{
  "peerDependencies": {
    "phaser": "^3.70.0",
    "livekit-client": "^2.0.0"
  },
  "devDependencies": {
    "typescript": "^5.3.0",
    "vite": "^5.0.0",
    "turbo": "^1.11.0"
  }
}
```

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| LiveKit DataChannel quirks | Test early with real network |
| Phaser plugin lifecycle | Follow Phaser plugin patterns |
| Latency issues | Interpolation + prediction |
| Scope creep | Stick to "two squares first" |

## Dev Environment

```bash
# LiveKit Cloud (free tier) or local:
docker run -d \
  --name livekit \
  -p 7880:7880 \
  -p 50000-50200:50000-50200/udp \
  livekit/livekit-server \
  --dev

# Generate test token:
npx @opengames/cli token --room test --player player1
```
