# Entity System Documentation

## Overview

The Entity System is the core of the Moye ECS framework, providing a robust Entity-Component-System architecture. It manages entity lifecycles, component relationships, and hierarchical structures.

## Core Classes

### Entity

The base class for all entities and components in the framework. Entities can contain other entities as children and components.

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

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `parent` | `Entity` | Parent entity in the hierarchy |
| `domain` | `Scene` | Scene that owns this entity |
| `instanceId` | `bigint` | Unique instance identifier |
| `id` | `bigint` | Entity identifier |
| `isDisposed` | `boolean` | Whether the entity has been disposed |
| `children` | `Map<bigint, Entity>` | Child entities |
| `components` | `Map<Type<Entity>, Entity>` | Attached components |

#### Methods

##### addComponent<T extends Entity>(type: new() => T): T

Adds a component to the entity.

```typescript
class HealthComponent extends Entity {
    health: number = 100;
    maxHealth: number = 100;
}

const entity = Entity.create(scene);
const healthComponent = entity.addComponent(HealthComponent);
```

**Parameters:**
- `type`: Component class constructor

**Returns:** The created component instance

##### getComponent<T extends Entity>(type: new() => T): T

Retrieves a component from the entity.

```typescript
const healthComponent = entity.getComponent(HealthComponent);
if (healthComponent) {
    healthComponent.health -= 10;
}
```

**Parameters:**
- `type`: Component class constructor

**Returns:** The component instance or null if not found

##### removeComponent<T extends Entity>(type: new() => T): void

Removes a component from the entity.

```typescript
entity.removeComponent(HealthComponent);
```

**Parameters:**
- `type`: Component class constructor

##### addChild(child: Entity): void

Adds a child entity to this entity.

```typescript
const parent = Entity.create(scene);
const child = Entity.create(scene);
parent.addChild(child);
```

**Parameters:**
- `child`: Entity to add as child

##### removeChild(id: bigint): Entity

Removes a child entity by ID.

```typescript
const removedChild = parent.removeChild(childId);
```

**Parameters:**
- `id`: Child entity ID

**Returns:** The removed child entity

##### dispose(): void

Disposes the entity and all its children and components.

```typescript
entity.dispose(); // Cleans up all resources
```

#### Lifecycle Methods

Override these methods in your entity classes to handle lifecycle events:

```typescript
class MyEntity extends Entity {
    awake() {
        // Called when entity is created and domain is set
        console.log("Entity awakened");
    }
    
    update(dt: number) {
        // Called every frame if entity implements ILifeCycle
        console.log("Entity updating");
    }
    
    lateUpdate(dt: number) {
        // Called after all updates
        console.log("Entity late updating");
    }
    
    destroy() {
        // Called when entity is being disposed
        console.log("Entity destroying");
    }
}
```

### Scene

Scenes are the root containers for entities. They represent different game states or levels.

```typescript
class Scene extends Entity {
    sceneType: string;
    isDisposed: boolean;
}
```

#### Creating Scenes

```typescript
import { Scene } from 'moye-cocos';

// Direct creation
const scene = Scene.create();
scene.sceneType = "Game";

// Using scene factory
const menuScene = SceneFactory.create("MainMenu");
```

#### Scene Types

Scene types allow you to filter events and systems:

```typescript
// Common scene types
const SCENE_TYPES = {
    MAIN_MENU: "MainMenu",
    GAME: "Game", 
    BATTLE: "Battle",
    INVENTORY: "Inventory"
};
```

### EntityCenter

Manages global entity registration and lookup.

```typescript
class EntityCenter extends Singleton {
    static get(id: bigint): Entity;
    static getByInstanceId(instanceId: bigint): Entity;
}
```

#### Usage

```typescript
import { EntityCenter } from 'moye-cocos';

// Get entity by ID
const entity = EntityCenter.get(entityId);

// Get entity by instance ID
const entity = EntityCenter.getByInstanceId(instanceId);
```

## Advanced Usage

### Custom Entity Types

Create specialized entity types for different game objects:

```typescript
class PlayerEntity extends Entity {
    awake() {
        // Add required components
        this.addComponent(HealthComponent);
        this.addComponent(MovementComponent);
        this.addComponent(InputComponent);
        this.addComponent(AnimationComponent);
    }
    
    takeDamage(amount: number) {
        const health = this.getComponent(HealthComponent);
        health.takeDamage(amount);
    }
    
    moveTo(x: number, y: number) {
        const movement = this.getComponent(MovementComponent);
        movement.setTarget(x, y);
    }
}

class NPCEntity extends Entity {
    awake() {
        this.addComponent(HealthComponent);
        this.addComponent(AIComponent);
        this.addComponent(DialogueComponent);
    }
    
    startDialogue(player: PlayerEntity) {
        const dialogue = this.getComponent(DialogueComponent);
        dialogue.begin(player);
    }
}
```

### Component System

Components are specialized entities that add functionality:

```typescript
// Health component
class HealthComponent extends Entity {
    health: number = 100;
    maxHealth: number = 100;
    
    takeDamage(amount: number) {
        this.health = Math.max(0, this.health - amount);
        
        if (this.health <= 0) {
            this.onDeath();
        }
    }
    
    heal(amount: number) {
        this.health = Math.min(this.maxHealth, this.health + amount);
    }
    
    private onDeath() {
        // Trigger death event
        EventSystem.get().publish(
            this.domain,
            EntityDeathEvent.create(this.parent.id)
        );
    }
}

// Movement component
class MovementComponent extends Entity {
    speed: number = 5.0;
    position: { x: number, y: number } = { x: 0, y: 0 };
    target: { x: number, y: number } | null = null;
    
    update(dt: number) {
        if (this.target) {
            this.moveTowardsTarget(dt);
        }
    }
    
    setTarget(x: number, y: number) {
        this.target = { x, y };
    }
    
    private moveTowardsTarget(dt: number) {
        if (!this.target) return;
        
        const dx = this.target.x - this.position.x;
        const dy = this.target.y - this.position.y;
        const distance = Math.sqrt(dx * dx + dy * dy);
        
        if (distance < 0.1) {
            this.position.x = this.target.x;
            this.position.y = this.target.y;
            this.target = null;
            return;
        }
        
        const moveDistance = this.speed * dt;
        const ratio = moveDistance / distance;
        
        this.position.x += dx * ratio;
        this.position.y += dy * ratio;
    }
}

// Input component
class InputComponent extends Entity {
    private keys: Set<string> = new Set();
    
    awake() {
        this.setupInputListeners();
    }
    
    private setupInputListeners() {
        document.addEventListener('keydown', (e) => {
            this.keys.add(e.code);
        });
        
        document.addEventListener('keyup', (e) => {
            this.keys.delete(e.code);
        });
    }
    
    isKeyPressed(key: string): boolean {
        return this.keys.has(key);
    }
    
    update(dt: number) {
        const movement = this.parent.getComponent(MovementComponent);
        
        if (this.isKeyPressed('KeyW')) {
            movement.setTarget(movement.position.x, movement.position.y - 1);
        }
        if (this.isKeyPressed('KeyS')) {
            movement.setTarget(movement.position.x, movement.position.y + 1);
        }
        if (this.isKeyPressed('KeyA')) {
            movement.setTarget(movement.position.x - 1, movement.position.y);
        }
        if (this.isKeyPressed('KeyD')) {
            movement.setTarget(movement.position.x + 1, movement.position.y);
        }
    }
}
```

### Entity Hierarchies

Create complex hierarchies for game objects:

```typescript
class GameWorldEntity extends Entity {
    awake() {
        // Create world regions
        const grasslands = this.addChild(Entity.create(this.domain, RegionEntity));
        grasslands.getComponent(RegionComponent).setType("grasslands");
        
        const mountains = this.addChild(Entity.create(this.domain, RegionEntity));
        mountains.getComponent(RegionComponent).setType("mountains");
        
        // Create players container
        const players = this.addChild(Entity.create(this.domain));
        players.name = "Players";
        
        // Create NPCs container
        const npcs = this.addChild(Entity.create(this.domain));
        npcs.name = "NPCs";
    }
}

class RegionEntity extends Entity {
    awake() {
        this.addComponent(RegionComponent);
        this.addComponent(WeatherComponent);
        this.addComponent(SpawnComponent);
    }
}
```

### Entity Pooling

Optimize performance with entity pooling:

```typescript
class EntityPool {
    private static pools: Map<string, Entity[]> = new Map();
    
    static get<T extends Entity>(type: new() => T, scene: Scene): T {
        const typeName = type.name;
        const pool = this.pools.get(typeName) || [];
        
        if (pool.length > 0) {
            const entity = pool.pop() as T;
            entity.domain = scene;
            return entity;
        }
        
        return Entity.create(scene, type);
    }
    
    static release<T extends Entity>(entity: T) {
        const typeName = entity.constructor.name;
        const pool = this.pools.get(typeName) || [];
        
        entity.reset(); // Custom reset method
        pool.push(entity);
        this.pools.set(typeName, pool);
    }
}

// Usage
const bullet = EntityPool.get(BulletEntity, scene);
// ... use bullet
EntityPool.release(bullet);
```

## Best Practices

### 1. Component Composition

Prefer composition over inheritance:

```typescript
// Good: Composable components
class PlayerEntity extends Entity {
    awake() {
        this.addComponent(HealthComponent);
        this.addComponent(MovementComponent);
        this.addComponent(InputComponent);
        this.addComponent(InventoryComponent);
    }
}

// Avoid: Deep inheritance
class MovableEntity extends Entity { }
class LivingEntity extends MovableEntity { }
class PlayerEntity extends LivingEntity { }
```

### 2. Single Responsibility

Keep components focused on a single responsibility:

```typescript
// Good: Focused components
class HealthComponent extends Entity { }
class MovementComponent extends Entity { }
class RenderComponent extends Entity { }

// Avoid: Monolithic components
class PlayerComponent extends Entity {
    // health, movement, rendering, input, etc.
}
```

### 3. Entity Lifecycle Management

Always clean up resources properly:

```typescript
class NetworkComponent extends Entity {
    private connection: WebSocket;
    
    awake() {
        this.connection = new WebSocket('ws://localhost:8080');
    }
    
    destroy() {
        // Clean up resources
        if (this.connection) {
            this.connection.close();
        }
    }
}
```

### 4. Entity Relationships

Use proper parent-child relationships:

```typescript
// Create weapon as child of player
const player = Entity.create(scene, PlayerEntity);
const weapon = Entity.create(scene, WeaponEntity);
player.addChild(weapon);

// Weapon can access player
class WeaponEntity extends Entity {
    getOwner(): PlayerEntity {
        return this.parent as PlayerEntity;
    }
}
```

## Performance Considerations

### 1. Component Caching

Cache frequently accessed components:

```typescript
class PlayerEntity extends Entity {
    private healthComponent: HealthComponent;
    private movementComponent: MovementComponent;
    
    awake() {
        this.healthComponent = this.addComponent(HealthComponent);
        this.movementComponent = this.addComponent(MovementComponent);
    }
    
    update(dt: number) {
        // Use cached references instead of getComponent() calls
        this.healthComponent.update(dt);
        this.movementComponent.update(dt);
    }
}
```

### 2. Batch Operations

Batch entity operations when possible:

```typescript
// Good: Batch creation
const entities: Entity[] = [];
for (let i = 0; i < 100; i++) {
    entities.push(Entity.create(scene, EnemyEntity));
}

// Process all at once
entities.forEach(entity => {
    entity.getComponent(SpawnComponent).spawn();
});
```

### 3. Memory Management

Dispose entities when no longer needed:

```typescript
class EnemySpawner extends Entity {
    private enemies: Set<Entity> = new Set();
    
    spawnEnemy() {
        const enemy = Entity.create(this.domain, EnemyEntity);
        this.enemies.add(enemy);
    }
    
    despawnEnemy(enemy: Entity) {
        this.enemies.delete(enemy);
        enemy.dispose(); // Important: clean up
    }
    
    destroy() {
        // Clean up all enemies
        for (const enemy of this.enemies) {
            enemy.dispose();
        }
        this.enemies.clear();
    }
}
```