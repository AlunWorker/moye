# Moye ECS Framework - API Documentation

## Overview

Moye is a powerful Entity-Component-System (ECS) game framework designed for Cocos Creator. It provides a complete set of tools for building scalable and maintainable games with support for networking, UI management, event systems, and more.

## Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [Core Components](#core-components)
- [API Reference](#api-reference)
  - [Entity System](#entity-system)
  - [Event System](#event-system)
  - [UI Framework](#ui-framework)
  - [Network System](#network-system)
  - [Timer Management](#timer-management)
  - [Logger](#logger)
  - [Message System](#message-system)
- [Examples](#examples)
- [Configuration](#configuration)

## Installation

```bash
npm install moye-cocos
```

Or include it directly in your project:

```typescript
import * from 'moye-cocos';
```

## Quick Start

### Basic Setup

```typescript
import { Game, Entity, Scene, Logger } from 'moye-cocos';

// Initialize the framework
class MyGame {
    async start() {
        // Add core singletons
        Game.addSingleton(Logger);
        
        // Create a scene
        const scene = Scene.create();
        
        // Create entities
        const entity = Entity.create(scene);
        
        Logger.get().log("Moye framework initialized!");
    }
}

const game = new MyGame();
game.start();
```

### Creating Components

```typescript
import { Entity } from 'moye-cocos';

class HealthComponent extends Entity {
    health: number = 100;
    maxHealth: number = 100;
    
    takeDamage(amount: number) {
        this.health = Math.max(0, this.health - amount);
    }
    
    heal(amount: number) {
        this.health = Math.min(this.maxHealth, this.health + amount);
    }
}

// Use the component
const player = Entity.create(scene);
const healthComp = player.addComponent(HealthComponent);
healthComp.takeDamage(25);
```

## Core Components

The Moye framework is built around several key systems:

1. **Entity System** - Core ECS implementation
2. **Event System** - Decoupled communication between components
3. **UI Framework** - Cocos Creator UI management
4. **Network System** - HTTP and WebSocket networking
5. **Timer Management** - Scheduled task execution
6. **Logger** - Configurable logging system
7. **Message System** - Network message handling

## API Reference

### Entity System

The core of the ECS framework, providing entity and component management.

#### Entity Class

The base class for all entities and components in the framework.

```typescript
abstract class Entity {
    parent: Entity;
    domain: Scene;
    instanceId: bigint;
    id: bigint;
    readonly isDisposed: boolean;
    readonly children: Map<bigint, Entity>;
    readonly components: Map<Type<Entity>, Entity>;
}
```

**Key Methods:**

- `addComponent<T extends Entity>(type: new() => T): T` - Add a component to the entity
- `getComponent<T extends Entity>(type: new() => T): T` - Get a component from the entity
- `removeComponent<T extends Entity>(type: new() => T): void` - Remove a component
- `dispose(): void` - Dispose the entity and clean up resources

**Example:**

```typescript
class PlayerEntity extends Entity {
    awake() {
        // Initialize player
        this.addComponent(HealthComponent);
        this.addComponent(MovementComponent);
    }
}

const player = Entity.create(scene, PlayerEntity);
const health = player.getComponent(HealthComponent);
```

#### Scene Management

Scenes are the root containers for entities.

```typescript
class Scene extends Entity {
    sceneType: string;
    isDisposed: boolean;
}
```

**Example:**

```typescript
import { Scene, SceneFactory } from 'moye-cocos';

// Create a scene
const gameScene = Scene.create();
gameScene.sceneType = "Game";

// Or use the scene factory
const scene = SceneFactory.create("MainMenu");
```

### Event System

Provides decoupled communication between components using an event-driven architecture.

#### EventSystem Class

```typescript
class EventSystem extends Singleton {
    publishAsync<T extends AEvent>(scene: IScene, eventType: T): Promise<void>;
    publish<T extends AEvent>(scene: IScene, eventType: T): void;
}
```

**Creating Events:**

```typescript
import { AEvent } from 'moye-cocos';

class PlayerDiedEvent extends AEvent {
    playerId: string;
    cause: string;
    
    static create(playerId: string, cause: string): PlayerDiedEvent {
        const event = new PlayerDiedEvent();
        event.playerId = playerId;
        event.cause = cause;
        return event;
    }
}
```

**Event Handlers:**

```typescript
import { AEventHandler, EventHandler } from 'moye-cocos';

@EventHandler("Game") // Scene type filter
class PlayerDiedHandler extends AEventHandler<PlayerDiedEvent> {
    async handleAsync(scene: IScene, event: PlayerDiedEvent): Promise<void> {
        console.log(`Player ${event.playerId} died: ${event.cause}`);
        // Handle respawn logic
    }
}
```

**Publishing Events:**

```typescript
import { EventSystem } from 'moye-cocos';

// Synchronous
EventSystem.get().publish(scene, PlayerDiedEvent.create("player1", "fall"));

// Asynchronous
await EventSystem.get().publishAsync(scene, PlayerDiedEvent.create("player1", "fall"));
```

### UI Framework

Moye provides a comprehensive UI management system built on top of Cocos Creator.

#### AMoyeView Base Class

```typescript
abstract class AMoyeView extends Entity {
    viewName: string;
    layer: ViewLayer;
    node: Node;
    
    protected onLoad?(): void;
    protected onShow?(): void;
    protected onHide?(): void;
    protected destroy?(): void;
    protected awake?(): void;
    
    dispose(): void;
    bringToFront(): void;
}
```

**Creating UI Views:**

```typescript
import { AMoyeView, ViewDecorator, ViewLayer } from 'moye-cocos';

@ViewDecorator({
    prefabPath: "ui/MainMenu",
    layer: ViewLayer.UI
})
class MainMenuView extends AMoyeView {
    protected onLoad() {
        // Initialize UI components
        const startBtn = this.node.getChildByName("StartButton");
        startBtn.on('click', this.onStartClick, this);
    }
    
    protected onShow() {
        // Called when view becomes visible
        this.playOpenAnimation();
    }
    
    protected onHide() {
        // Called when view becomes hidden
    }
    
    private onStartClick() {
        // Handle start button click
    }
}
```

**View Management:**

```typescript
import { MoyeViewMgr } from 'moye-cocos';

const viewMgr = MoyeViewMgr.get();

// Show view
await viewMgr.show(MainMenuView);

// Hide view
viewMgr.hide("MainMenuView");

// Check if view is showing
if (viewMgr.isShowing("MainMenuView")) {
    console.log("Main menu is visible");
}
```

### Network System

Handles HTTP requests and WebSocket connections.

#### NetServices

```typescript
class NetServices extends Singleton {
    httpRequest<T>(url: string, options?: RequestOptions): Promise<T>;
    httpPost<T>(url: string, data: any, options?: RequestOptions): Promise<T>;
    httpGet<T>(url: string, options?: RequestOptions): Promise<T>;
}
```

**HTTP Examples:**

```typescript
import { NetServices } from 'moye-cocos';

const netService = NetServices.get();

// GET request
const userData = await netService.httpGet<UserData>('/api/user/profile');

// POST request
const result = await netService.httpPost('/api/user/login', {
    username: 'player1',
    password: 'password123'
});
```

#### WebSocket Support

```typescript
import { NetworkWebsocket, SessionCom } from 'moye-cocos';

// Create WebSocket connection
const session = SessionCom.create();
await session.connect('ws://localhost:8080');

// Send message
session.send(LoginMessage.create('username', 'password'));

// Handle incoming messages
@MsgHandler(LoginResponseMessage)
class LoginResponseHandler extends AMHandler<LoginResponseMessage> {
    async handle(session: Session, message: LoginResponseMessage): Promise<void> {
        if (message.success) {
            console.log('Login successful');
        }
    }
}
```

### Timer Management

Provides scheduling and timer functionality.

#### TimerMgr Class

```typescript
class TimerMgr extends Singleton {
    newRepeatedTimer(interval: number, callback: Action, immediately?: boolean): number;
    newOnceTimer(timeout: number, callback: Action): number;
    newFrameTimer(callback: Action): number;
    removeTimer(timerId: number): void;
    waitAsync(timeout: number, cancellationToken?: CancellationToken): Promise<void>;
    waitFrameAsync(cancellationToken?: CancellationToken): Promise<void>;
}
```

**Examples:**

```typescript
import { TimerMgr, CancellationToken } from 'moye-cocos';

const timerMgr = TimerMgr.get();

// Repeated timer (every 1 second)
const repeatedTimer = timerMgr.newRepeatedTimer(1000, () => {
    console.log('Tick every second');
});

// One-time timer (5 seconds)
const onceTimer = timerMgr.newOnceTimer(5000, () => {
    console.log('Executed after 5 seconds');
});

// Wait asynchronously with cancellation
const token = CancellationToken.create();
try {
    await timerMgr.waitAsync(3000, token);
    console.log('Wait completed');
} catch (error) {
    console.log('Wait was cancelled');
}

// Cancel timer
timerMgr.removeTimer(repeatedTimer);

// Wait for next frame
await timerMgr.waitFrameAsync();
```

### Logger

Configurable logging system with multiple levels and formatting.

#### Logger Class

```typescript
class Logger extends Singleton {
    level: LoggerLevel;
    iLog: ILog;
    
    debug(...args: any[]): void;
    debugF(str: string, ...args: any[]): void;
    log(...args: any[]): void;
    logF(str: string, ...args: any[]): void;
    warn(...args: any[]): void;
    warnF(str: string, ...args: any[]): void;
    error(...args: any[]): void;
    errorF(str: string, ...args: any[]): void;
}
```

**Global Functions:**

```typescript
// Direct logging functions
import { log, logF, debug, warn, error } from 'moye-cocos';

// Simple logging
log("Game started");
debug("Player position:", player.x, player.y);
warn("Low health warning");
error("Critical error occurred");

// Formatted logging
logF("Player {0} scored {1} points", playerName, score);
debugF("Position: ({0}, {1})", x, y);
warnF("Health low: {0}/{1}", currentHealth, maxHealth);
errorF("Failed to load asset: {0}", assetPath);
```

**Log Levels:**

```typescript
import { Logger, LoggerLevel } from 'moye-cocos';

// Set log level
Logger.get().level = LoggerLevel.Warn; // Only show warnings and errors

// Available levels:
// LoggerLevel.Debug
// LoggerLevel.Log  
// LoggerLevel.Warn
// LoggerLevel.Error
```

### Message System

Handles network message serialization and routing.

#### Message Handlers

```typescript
import { AMHandler, MsgHandler } from 'moye-cocos';

@MsgHandler(PlayerMoveMessage)
class PlayerMoveHandler extends AMHandler<PlayerMoveMessage> {
    async handle(session: Session, message: PlayerMoveMessage): Promise<void> {
        // Handle player movement
        const player = this.getPlayerBySession(session);
        player.moveTo(message.x, message.y);
    }
}
```

#### Session Management

```typescript
import { Session, SessionCom } from 'moye-cocos';

// Create session component
const sessionCom = entity.addComponent(SessionCom);

// Connect to server
await sessionCom.connect('ws://localhost:8080');

// Send message
sessionCom.send(PlayerActionMessage.create(action, data));

// Disconnect
sessionCom.disconnect();
```

## Examples

### Complete Game Scene Setup

```typescript
import { 
    Game, Scene, Entity, Logger, TimerMgr, EventSystem, 
    MoyeViewMgr, NetServices 
} from 'moye-cocos';

class GameScene {
    private scene: Scene;
    
    async initialize() {
        // Add core singletons
        Game.addSingleton(Logger);
        Game.addSingleton(TimerMgr);
        Game.addSingleton(EventSystem);
        Game.addSingleton(MoyeViewMgr);
        Game.addSingleton(NetServices);
        
        // Create game scene
        this.scene = Scene.create();
        this.scene.sceneType = "Game";
        
        // Initialize game systems
        await this.initializeGameSystems();
        
        Logger.get().log("Game scene initialized");
    }
    
    private async initializeGameSystems() {
        // Create player
        const player = Entity.create(this.scene, PlayerEntity);
        
        // Create game manager
        const gameManager = Entity.create(this.scene, GameManagerEntity);
        
        // Show initial UI
        await MoyeViewMgr.get().show(GameUIView);
    }
    
    update(dt: number) {
        Game.update(dt);
    }
    
    lateUpdate(dt: number) {
        Game.lateUpdate(dt);
    }
    
    onDestroy() {
        Game.dispose();
    }
}
```

### Player Entity with Components

```typescript
class PlayerEntity extends Entity {
    awake() {
        // Add components
        this.addComponent(HealthComponent);
        this.addComponent(MovementComponent);
        this.addComponent(InventoryComponent);
        this.addComponent(NetworkSyncComponent);
    }
}

class HealthComponent extends Entity {
    health: number = 100;
    maxHealth: number = 100;
    
    takeDamage(amount: number) {
        this.health = Math.max(0, this.health - amount);
        
        if (this.health <= 0) {
            EventSystem.get().publish(
                this.domain, 
                PlayerDiedEvent.create(this.parent.id.toString())
            );
        }
    }
}

class MovementComponent extends Entity {
    speed: number = 5.0;
    
    moveTo(x: number, y: number) {
        // Movement logic
        const playerNode = this.parent.getComponent(NodeComponent).node;
        playerNode.setPosition(x, y);
        
        // Sync position over network
        const networkSync = this.parent.getComponent(NetworkSyncComponent);
        networkSync.syncPosition(x, y);
    }
}
```

## Configuration

### Framework Configuration

```typescript
import { DEVELOP, MOYE_OUTPUT_DEBUG, MOYE_OUTPUT_LOG, MOYE_OUTPUT_WARN } from 'moye-cocos';

// Configuration constants (in src/Macro.ts)
export const DEVELOP = true;              // Development mode
export const MOYE_OUTPUT_DEBUG = true;    // Show debug output
export const MOYE_OUTPUT_LOG = true;      // Show log output  
export const MOYE_OUTPUT_WARN = true;     // Show warning output
```

### Logger Configuration

```typescript
import { Logger, LoggerLevel } from 'moye-cocos';

// Set custom logger implementation
class CustomLogger implements ILog {
    debug(...args: any[]) {
        console.log('[DEBUG]', ...args);
    }
    
    log(...args: any[]) {
        console.log('[LOG]', ...args);
    }
    
    warn(...args: any[]) {
        console.warn('[WARN]', ...args);
    }
    
    error(...args: any[]) {
        console.error('[ERROR]', ...args);
    }
}

Logger.get().iLog = new CustomLogger();
Logger.get().level = LoggerLevel.Debug;
```

### Build Configuration

The framework is built using TypeScript and Gulp. Key files:

- `package.json` - Dependencies and scripts
- `tsconfig.json` - TypeScript configuration
- `dist/moye.mjs` - Main distribution file
- `dist/moye.d.ts` - TypeScript definitions

---

This documentation covers the main public APIs of the Moye ECS framework. For more detailed information about specific components, see the individual module documentation files.