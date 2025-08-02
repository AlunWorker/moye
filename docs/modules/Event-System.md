# Event System Documentation

## Overview

The Event System provides a decoupled communication mechanism between components and systems in the Moye framework. It follows the observer pattern, allowing entities to communicate without direct references.

## Core Classes

### EventSystem

The main event dispatcher that manages event publishing and distribution.

```typescript
class EventSystem extends Singleton {
    publishAsync<T extends AEvent>(scene: IScene, eventType: T): Promise<void>;
    publish<T extends AEvent>(scene: IScene, eventType: T): void;
}
```

#### Methods

##### publishAsync<T extends AEvent>(scene: IScene, eventType: T): Promise<void>

Publishes an event asynchronously, waiting for all handlers to complete.

```typescript
await EventSystem.get().publishAsync(scene, PlayerDiedEvent.create("player1", "fall"));
```

**Parameters:**
- `scene`: The scene context for the event
- `eventType`: The event instance to publish

**Returns:** Promise that resolves when all handlers complete

##### publish<T extends AEvent>(scene: IScene, eventType: T): void

Publishes an event synchronously. Ensure handlers are not async methods.

```typescript
EventSystem.get().publish(scene, PlayerScoreEvent.create("player1", 100));
```

**Parameters:**
- `scene`: The scene context for the event
- `eventType`: The event instance to publish

### AEvent

Base class for all events in the system.

```typescript
abstract class AEvent extends Entity {
    static create(...args: any[]): AEvent;
    dispose(): void;
}
```

#### Creating Custom Events

Events should extend `AEvent` and provide a static `create` method:

```typescript
class PlayerDiedEvent extends AEvent {
    playerId: string;
    cause: string;
    position: { x: number, y: number };
    
    static create(playerId: string, cause: string, x: number = 0, y: number = 0): PlayerDiedEvent {
        const event = new PlayerDiedEvent();
        event.playerId = playerId;
        event.cause = cause;
        event.position = { x, y };
        return event;
    }
}

class ItemPickedUpEvent extends AEvent {
    playerId: string;
    itemId: string;
    itemType: string;
    quantity: number;
    
    static create(playerId: string, itemId: string, itemType: string, quantity: number = 1): ItemPickedUpEvent {
        const event = new ItemPickedUpEvent();
        event.playerId = playerId;
        event.itemId = itemId;
        event.itemType = itemType;
        event.quantity = quantity;
        return event;
    }
}

class SceneTransitionEvent extends AEvent {
    fromScene: string;
    toScene: string;
    transitionType: string;
    data?: any;
    
    static create(fromScene: string, toScene: string, transitionType: string = "fade", data?: any): SceneTransitionEvent {
        const event = new SceneTransitionEvent();
        event.fromScene = fromScene;
        event.toScene = toScene;
        event.transitionType = transitionType;
        event.data = data;
        return event;
    }
}
```

### AEventHandler

Base class for event handlers that respond to specific event types.

```typescript
abstract class AEventHandler<T extends AEvent> extends Entity {
    abstract handleAsync(scene: IScene, event: T): Promise<void>;
    handle?(scene: IScene, event: T): void;
}
```

#### Creating Event Handlers

Use the `@EventHandler` decorator to register handlers:

```typescript
import { AEventHandler, EventHandler } from 'moye-cocos';

@EventHandler("Game") // Scene type filter
class PlayerDiedHandler extends AEventHandler<PlayerDiedEvent> {
    async handleAsync(scene: IScene, event: PlayerDiedEvent): Promise<void> {
        console.log(`Player ${event.playerId} died: ${event.cause} at (${event.position.x}, ${event.position.y})`);
        
        // Handle respawn logic
        await this.handlePlayerRespawn(event.playerId);
        
        // Update statistics
        await this.updateDeathStatistics(event.playerId, event.cause);
        
        // Notify other players
        await this.notifyOtherPlayers(event);
    }
    
    private async handlePlayerRespawn(playerId: string): Promise<void> {
        // Wait for respawn timer
        await TimerMgr.get().waitAsync(3000);
        
        // Create respawn event
        EventSystem.get().publishAsync(
            this.scene,
            PlayerRespawnEvent.create(playerId, "spawn_point_1")
        );
    }
    
    private async updateDeathStatistics(playerId: string, cause: string): Promise<void> {
        const statsManager = this.scene.getComponent(StatisticsManagerComponent);
        await statsManager.recordDeath(playerId, cause);
    }
    
    private async notifyOtherPlayers(event: PlayerDiedEvent): Promise<void> {
        EventSystem.get().publishAsync(
            this.scene,
            ChatMessageEvent.create("system", `${event.playerId} was eliminated by ${event.cause}`)
        );
    }
}

@EventHandler("Game")
class ItemPickedUpHandler extends AEventHandler<ItemPickedUpEvent> {
    async handleAsync(scene: IScene, event: ItemPickedUpEvent): Promise<void> {
        // Add item to inventory
        const player = scene.getEntityById(event.playerId);
        const inventory = player.getComponent(InventoryComponent);
        
        inventory.addItem(event.itemId, event.itemType, event.quantity);
        
        // Play pickup sound
        EventSystem.get().publish(scene, PlaySoundEvent.create("item_pickup"));
        
        // Show pickup notification
        EventSystem.get().publish(scene, ShowNotificationEvent.create(
            `Picked up ${event.quantity}x ${event.itemType}`,
            "success"
        ));
        
        // Check for achievements
        await this.checkPickupAchievements(event);
    }
    
    private async checkPickupAchievements(event: ItemPickedUpEvent): Promise<void> {
        const achievementManager = this.scene.getComponent(AchievementManagerComponent);
        await achievementManager.checkItemPickupAchievements(event.playerId, event.itemType, event.quantity);
    }
}

// Multiple scene types
@EventHandler(["Game", "Battle", "Dungeon"])
class SceneTransitionHandler extends AEventHandler<SceneTransitionEvent> {
    async handleAsync(scene: IScene, event: SceneTransitionEvent): Promise<void> {
        console.log(`Transitioning from ${event.fromScene} to ${event.toScene} via ${event.transitionType}`);
        
        // Save current scene state
        await this.saveSceneState(event.fromScene);
        
        // Perform transition
        await this.performTransition(event);
        
        // Load new scene
        await this.loadNewScene(event.toScene, event.data);
    }
    
    private async saveSceneState(sceneType: string): Promise<void> {
        const saveManager = Game.getSingleton(SaveManagerComponent);
        await saveManager.saveScene(sceneType);
    }
    
    private async performTransition(event: SceneTransitionEvent): Promise<void> {
        const transitionManager = Game.getSingleton(TransitionManagerComponent);
        await transitionManager.performTransition(event.transitionType);
    }
    
    private async loadNewScene(sceneType: string, data?: any): Promise<void> {
        const sceneManager = Game.getSingleton(SceneManagerComponent);
        await sceneManager.loadScene(sceneType, data);
    }
}
```

### EventHandler Decorator

The `@EventHandler` decorator registers handlers with the event system.

```typescript
@EventHandler(sceneType?: string | string[])
```

**Parameters:**
- `sceneType`: Optional scene type filter. If provided, handler only responds to events from matching scenes
  - Single scene: `@EventHandler("Game")`
  - Multiple scenes: `@EventHandler(["Game", "Battle"])`
  - All scenes: `@EventHandler()` or `@EventHandler("None")`

## Advanced Usage

### Event Chains

Create complex event chains by publishing events from handlers:

```typescript
@EventHandler("Game")
class PlayerLevelUpHandler extends AEventHandler<PlayerLevelUpEvent> {
    async handleAsync(scene: IScene, event: PlayerLevelUpEvent): Promise<void> {
        const player = scene.getEntityById(event.playerId);
        const level = player.getComponent(LevelComponent);
        
        // Increase stats
        level.level = event.newLevel;
        level.skillPoints += event.skillPointsGained;
        
        // Chain additional events
        await EventSystem.get().publishAsync(scene, PlayerStatsChangedEvent.create(
            event.playerId,
            level.getStats()
        ));
        
        await EventSystem.get().publishAsync(scene, ShowLevelUpEffectEvent.create(
            event.playerId,
            event.newLevel
        ));
        
        await EventSystem.get().publishAsync(scene, PlaySoundEvent.create("level_up"));
        
        // Check for unlock conditions
        if (event.newLevel >= 10) {
            await EventSystem.get().publishAsync(scene, FeatureUnlockedEvent.create(
                event.playerId,
                "advanced_skills"
            ));
        }
    }
}
```

### Conditional Event Handling

Handle events conditionally based on game state:

```typescript
@EventHandler("Game")
class DamageHandler extends AEventHandler<DamageDealtEvent> {
    async handleAsync(scene: IScene, event: DamageDealtEvent): Promise<void> {
        const target = scene.getEntityById(event.targetId);
        const health = target.getComponent(HealthComponent);
        
        // Apply damage
        health.takeDamage(event.amount);
        
        // Conditional responses
        if (health.health <= 0) {
            // Target died
            await EventSystem.get().publishAsync(scene, EntityDeathEvent.create(
                event.targetId,
                event.attackerId,
                event.damageType
            ));
        } else if (health.health <= health.maxHealth * 0.25) {
            // Critical health
            await EventSystem.get().publishAsync(scene, CriticalHealthEvent.create(
                event.targetId
            ));
        }
        
        // Check for critical hit
        if (event.isCritical) {
            await EventSystem.get().publishAsync(scene, CriticalHitEvent.create(
                event.attackerId,
                event.targetId,
                event.amount
            ));
        }
        
        // Update combat statistics
        await EventSystem.get().publishAsync(scene, CombatStatsUpdateEvent.create(
            event.attackerId,
            "damage_dealt",
            event.amount
        ));
    }
}
```

### Event Batching

Batch multiple events for performance:

```typescript
class EventBatcher {
    private pendingEvents: Map<string, AEvent[]> = new Map();
    private batchTimer: number | null = null;
    
    batchEvent(scene: IScene, event: AEvent, batchKey: string) {
        if (!this.pendingEvents.has(batchKey)) {
            this.pendingEvents.set(batchKey, []);
        }
        
        this.pendingEvents.get(batchKey)!.push(event);
        
        if (!this.batchTimer) {
            this.batchTimer = TimerMgr.get().newOnceTimer(16, () => {
                this.flushBatch(scene);
            });
        }
    }
    
    private async flushBatch(scene: IScene) {
        for (const [batchKey, events] of this.pendingEvents) {
            if (events.length > 0) {
                await EventSystem.get().publishAsync(scene, BatchedEvent.create(batchKey, events));
            }
        }
        
        this.pendingEvents.clear();
        this.batchTimer = null;
    }
}

// Usage
const batcher = new EventBatcher();

// Batch damage events
for (let i = 0; i < 10; i++) {
    batcher.batchEvent(scene, DamageDealtEvent.create(attackerId, targetId, 10), "damage");
}
```

### Event Filtering

Filter events based on criteria:

```typescript
@EventHandler("Game")
class ConditionalHandler extends AEventHandler<PlayerActionEvent> {
    async handleAsync(scene: IScene, event: PlayerActionEvent): Promise<void> {
        const player = scene.getEntityById(event.playerId);
        
        // Filter based on player state
        if (!this.canPlayerPerformAction(player, event.actionType)) {
            return; // Ignore this event
        }
        
        // Process the action
        await this.processPlayerAction(player, event);
    }
    
    private canPlayerPerformAction(player: Entity, actionType: string): boolean {
        const state = player.getComponent(PlayerStateComponent);
        
        if (state.isStunned || state.isDead) {
            return false;
        }
        
        if (actionType === "cast_spell" && state.mana < 10) {
            return false;
        }
        
        if (actionType === "use_item" && state.isInCombat && !state.canUseItemsInCombat) {
            return false;
        }
        
        return true;
    }
}
```

## Built-in Event Types

### Core Framework Events

The framework provides several built-in events:

```typescript
// Lifecycle events
class AfterSingletonAdd extends AEvent {
    singletonType: new() => Singleton;
}

class BeforeSingletonAdd extends AEvent {
    singletonType: new() => Singleton;
}

class AfterProgramInit extends AEvent {
    // Framework initialization complete
}

class BeforeProgramStart extends AEvent {
    // Before main program loop starts
}

// Entity events
class EntityCreatedEvent extends AEvent {
    entityId: bigint;
    entityType: string;
}

class EntityDestroyedEvent extends AEvent {
    entityId: bigint;
    entityType: string;
}

// Scene events
class SceneLoadedEvent extends AEvent {
    sceneType: string;
    scene: IScene;
}

class SceneUnloadedEvent extends AEvent {
    sceneType: string;
}
```

## Best Practices

### 1. Event Naming

Use clear, descriptive names with consistent patterns:

```typescript
// Good: Clear action and subject
class PlayerDiedEvent extends AEvent { }
class ItemPickedUpEvent extends AEvent { }
class QuestCompletedEvent extends AEvent { }

// Avoid: Vague or inconsistent names
class PlayerEvent extends AEvent { }
class PickupEvent extends AEvent { }
class QuestEvent extends AEvent { }
```

### 2. Event Data Design

Include all necessary data in events:

```typescript
// Good: Complete data
class CombatEvent extends AEvent {
    attackerId: string;
    targetId: string;
    damage: number;
    damageType: string;
    isCritical: boolean;
    position: { x: number, y: number };
    timestamp: number;
    
    static create(attackerId: string, targetId: string, damage: number, damageType: string, 
                  isCritical: boolean = false, x: number = 0, y: number = 0): CombatEvent {
        const event = new CombatEvent();
        event.attackerId = attackerId;
        event.targetId = targetId;
        event.damage = damage;
        event.damageType = damageType;
        event.isCritical = isCritical;
        event.position = { x, y };
        event.timestamp = Date.now();
        return event;
    }
}

// Avoid: Missing context
class CombatEvent extends AEvent {
    damage: number; // Not enough information
}
```

### 3. Handler Organization

Group related handlers in modules:

```typescript
// PlayerEventHandlers.ts
@EventHandler("Game")
class PlayerLevelUpHandler extends AEventHandler<PlayerLevelUpEvent> { }

@EventHandler("Game")
class PlayerDiedHandler extends AEventHandler<PlayerDiedEvent> { }

@EventHandler("Game")
class PlayerRespawnHandler extends AEventHandler<PlayerRespawnEvent> { }

// CombatEventHandlers.ts
@EventHandler("Game")
class DamageDealtHandler extends AEventHandler<DamageDealtEvent> { }

@EventHandler("Game")
class CriticalHitHandler extends AEventHandler<CriticalHitEvent> { }

@EventHandler("Game")
class HealingHandler extends AEventHandler<HealingEvent> { }
```

### 4. Error Handling

Handle errors gracefully in event handlers:

```typescript
@EventHandler("Game")
class RobustHandler extends AEventHandler<MyEvent> {
    async handleAsync(scene: IScene, event: MyEvent): Promise<void> {
        try {
            await this.processEvent(event);
        } catch (error) {
            Logger.get().error(`Error handling ${event.constructor.name}:`, error);
            
            // Publish error event for centralized error handling
            await EventSystem.get().publishAsync(scene, ErrorEvent.create(
                error.message,
                event.constructor.name,
                this.constructor.name
            ));
        }
    }
    
    private async processEvent(event: MyEvent): Promise<void> {
        // Main event processing logic
    }
}
```

### 5. Performance Considerations

Use appropriate publishing methods:

```typescript
// For time-critical events (UI updates, input)
EventSystem.get().publish(scene, InputEvent.create(keyCode));

// For complex operations that can wait
await EventSystem.get().publishAsync(scene, SaveGameEvent.create());

// For events that need ordering
await EventSystem.get().publishAsync(scene, TurnBasedActionEvent.create(playerId, action));
```

## Debugging Events

### Event Logging

Enable event debugging:

```typescript
@EventHandler("Game")
class DebugEventHandler extends AEventHandler<AEvent> {
    async handleAsync(scene: IScene, event: AEvent): Promise<void> {
        if (DEVELOP) {
            Logger.get().debug(`Event: ${event.constructor.name}`, event);
        }
    }
}
```

### Event Monitoring

Create an event monitor for development:

```typescript
class EventMonitor extends Singleton {
    private eventCounts: Map<string, number> = new Map();
    private eventTimes: Map<string, number[]> = new Map();
    
    logEvent(eventType: string, duration: number) {
        // Count events
        const count = this.eventCounts.get(eventType) || 0;
        this.eventCounts.set(eventType, count + 1);
        
        // Track timing
        const times = this.eventTimes.get(eventType) || [];
        times.push(duration);
        if (times.length > 100) times.shift(); // Keep last 100
        this.eventTimes.set(eventType, times);
    }
    
    getStatistics(): EventStatistics {
        const stats: EventStatistics = {};
        
        for (const [eventType, count] of this.eventCounts) {
            const times = this.eventTimes.get(eventType) || [];
            const avgTime = times.length > 0 ? times.reduce((a, b) => a + b) / times.length : 0;
            
            stats[eventType] = {
                count,
                avgTime,
                maxTime: Math.max(...times),
                minTime: Math.min(...times)
            };
        }
        
        return stats;
    }
}
```