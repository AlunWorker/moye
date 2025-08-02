# Moye ECS Framework Documentation

Welcome to the comprehensive documentation for the Moye ECS Framework - a powerful Entity-Component-System game framework designed for Cocos Creator.

## Documentation Structure

### [Main API Documentation](API.md)
Complete overview of the framework with installation, quick start guide, and all core APIs in one place.

### Module Documentation

#### Core Framework
- [**Entity System**](modules/Entity-System.md) - Core ECS implementation with entities, components, and hierarchies
- [**Event System**](modules/Event-System.md) - Decoupled communication system with events and handlers
- [**Timer System**](modules/Timer-System.md) - Timer management, tasks, and asynchronous operations
- [**Logger System**](modules/Logger-System.md) - Comprehensive logging with levels, formatting, and debugging

#### Game Features
- [**UI Framework**](modules/UI-Framework.md) - Complete UI management system with views, dialogs, and components
- [**Network System**](modules/Network-System.md) - HTTP requests, WebSocket connections, and message handling

## Quick Navigation

### Getting Started
- [Installation](#installation)
- [Basic Setup](#basic-setup) 
- [Your First Entity](#creating-components)

### Core Concepts
- **Entities** - Game objects that can have components
- **Components** - Modular functionality attached to entities
- **Systems** - Logic that operates on entities with specific components
- **Events** - Decoupled communication between systems
- **Scenes** - Containers that manage entity lifecycles

### Framework Features
- **ECS Architecture** - Efficient entity-component-system design
- **Event-Driven** - Reactive programming with events
- **UI Management** - Comprehensive UI framework
- **Networking** - Built-in HTTP and WebSocket support
- **Timer System** - Advanced timing and scheduling
- **Logging** - Configurable logging with multiple levels

## Installation

```bash
npm install moye-cocos
```

Or include directly in your Cocos Creator project:

```typescript
import * from 'moye-cocos';
```

## Basic Setup

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

## Key APIs at a Glance

### Entity Management
```typescript
// Create entities
const player = Entity.create(scene, PlayerEntity);

// Add components
const health = player.addComponent(HealthComponent);
const movement = player.addComponent(MovementComponent);

// Access components
const healthComp = player.getComponent(HealthComponent);
```

### Event System
```typescript
// Define events
class PlayerDiedEvent extends AEvent {
    static create(playerId: string): PlayerDiedEvent { /* ... */ }
}

// Handle events
@EventHandler("Game")
class PlayerDiedHandler extends AEventHandler<PlayerDiedEvent> {
    async handleAsync(scene: IScene, event: PlayerDiedEvent): Promise<void> {
        // Handle player death
    }
}

// Publish events
EventSystem.get().publishAsync(scene, PlayerDiedEvent.create("player1"));
```

### UI Framework
```typescript
@ViewDecorator({
    prefabPath: "ui/MainMenu",
    layer: ViewLayer.UI
})
class MainMenuView extends AMoyeView {
    protected onLoad() {
        // Setup UI
    }
}

// Show views
const menu = await MoyeViewMgr.get().show(MainMenuView);
```

### Network Operations
```typescript
// HTTP requests
const userData = await NetServices.get().httpGet<UserData>('/api/user');

// WebSocket connections
const session = entity.addComponent(SessionCom);
await session.connect('ws://localhost:8080');
session.send(LoginMessage.create('username', 'password'));
```

### Timer Operations
```typescript
// Simple timers
const timerId = TimerMgr.get().newRepeatedTimer(1000, () => {
    console.log('Every second');
});

// Async operations
await TimerMgr.get().waitAsync(3000); // Wait 3 seconds
await TimerMgr.get().waitFrameAsync(); // Wait next frame
```

### Logging
```typescript
import { log, debug, warn, error, logF } from 'moye-cocos';

log("Game started");
debug("Player position:", x, y);
warn("Low health warning");
error("Critical error");

// Formatted logging
logF("Player {0} scored {1} points", playerName, score);
```

## Architecture Overview

The Moye framework follows these design principles:

- **Composition over Inheritance** - Build complex behavior by combining simple components
- **Event-Driven Architecture** - Loose coupling through events and messages
- **Lifecycle Management** - Automatic resource cleanup and lifecycle handling
- **Performance Focus** - Optimized for game development with object pooling and efficient updates
- **Developer Experience** - TypeScript support with comprehensive type safety

## Examples and Patterns

Each module documentation includes:
- **Basic Usage** - Simple examples to get started
- **Advanced Patterns** - Complex scenarios and optimizations  
- **Best Practices** - Recommended approaches and common pitfalls
- **Performance Tips** - Optimization strategies
- **Real-world Examples** - Complete implementations

## Support and Community

- **GitHub Repository**: [https://github.com/Alistairot/moye](https://github.com/Alistairot/moye)
- **Issues and Bug Reports**: Use GitHub Issues
- **API Questions**: Refer to module documentation
- **Feature Requests**: Submit via GitHub

## Version Information

- **Current Version**: 2.1.0
- **Compatibility**: Cocos Creator 3.x
- **TypeScript**: Full support with type definitions
- **License**: MIT

---

For detailed information about any specific system, navigate to the corresponding module documentation. The main API documentation provides a comprehensive overview of all systems in one place.