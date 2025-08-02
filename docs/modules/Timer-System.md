# Timer and Task System Documentation

## Overview

The Moye Timer and Task System provides comprehensive timing and asynchronous operation management. It includes timers, tasks, coroutines, and cancellation tokens for efficient game loop management.

## Core Components

### TimerMgr

Singleton manager for all timer operations.

```typescript
class TimerMgr extends Singleton {
    newRepeatedTimer(interval: number, callback: Action, immediately?: boolean): number;
    newOnceTimer(timeout: number, callback: Action): number;
    newFrameTimer(callback: Action): number;
    removeTimer(timerId: number): void;
    waitAsync(timeout: number, cancellationToken?: CancellationToken): Promise<void>;
    waitFrameAsync(cancellationToken?: CancellationToken): Promise<void>;
    waitUntil(condition: () => boolean, cancellationToken?: CancellationToken): Promise<void>;
}
```

#### Timer Methods

##### newRepeatedTimer(interval: number, callback: Action, immediately?: boolean): number

Creates a timer that repeats at specified intervals.

```typescript
import { TimerMgr } from 'moye-cocos';

const timerMgr = TimerMgr.get();

// Basic repeated timer
const timerId = timerMgr.newRepeatedTimer(1000, () => {
    console.log('Every second');
});

// Execute immediately, then repeat
const immediateTimer = timerMgr.newRepeatedTimer(2000, () => {
    console.log('Every 2 seconds, starting now');
}, true);

// Game loop example
const gameLoopTimer = timerMgr.newRepeatedTimer(16, () => {
    // 60 FPS game loop
    GameManager.get().update(16);
});
```

##### newOnceTimer(timeout: number, callback: Action): number

Creates a one-time timer that executes after a delay.

```typescript
// Simple delay
const delayTimer = timerMgr.newOnceTimer(3000, () => {
    console.log('Executed after 3 seconds');
});

// Respawn timer
const respawnTimer = timerMgr.newOnceTimer(5000, () => {
    const player = Game.getCurrentPlayer();
    player.respawn();
    EventSystem.get().publish(scene, PlayerRespawnEvent.create(player.id));
});

// Auto-save
const autoSaveTimer = timerMgr.newOnceTimer(60000, () => {
    GameSaveManager.get().autoSave();
});
```

##### newFrameTimer(callback: Action): number

Creates a timer that executes on the next frame.

```typescript
// Execute on next frame
const frameTimer = timerMgr.newFrameTimer(() => {
    console.log('Next frame execution');
});

// Defer UI update
const uiUpdateTimer = timerMgr.newFrameTimer(() => {
    UIManager.get().refreshInventory();
});
```

##### removeTimer(timerId: number): void

Cancels and removes a timer.

```typescript
// Store timer ID
const timerId = timerMgr.newRepeatedTimer(1000, () => {
    console.log('Repeating task');
});

// Cancel timer later
timerMgr.removeTimer(timerId);
```

#### Async Methods

##### waitAsync(timeout: number, cancellationToken?: CancellationToken): Promise<void>

Waits for a specified duration asynchronously.

```typescript
import { CancellationToken } from 'moye-cocos';

// Simple wait
await timerMgr.waitAsync(2000); // Wait 2 seconds

// Wait with cancellation
const token = CancellationToken.create();
try {
    await timerMgr.waitAsync(5000, token);
    console.log('Wait completed');
} catch (error) {
    console.log('Wait was cancelled');
}

// In a game method
class GameController extends Entity {
    async showCountdown() {
        for (let i = 3; i > 0; i--) {
            EventSystem.get().publish(scene, ShowCountdownEvent.create(i));
            await TimerMgr.get().waitAsync(1000);
        }
        
        EventSystem.get().publish(scene, GameStartEvent.create());
    }
}
```

##### waitFrameAsync(cancellationToken?: CancellationToken): Promise<void>

Waits for the next frame asynchronously.

```typescript
// Wait for next frame
await timerMgr.waitFrameAsync();

// Multi-frame operation
async function heavyOperation() {
    for (let i = 0; i < 1000; i++) {
        // Do some work
        processChunk(i);
        
        // Yield control every 10 iterations
        if (i % 10 === 0) {
            await timerMgr.waitFrameAsync();
        }
    }
}
```

##### waitUntil(condition: () => boolean, cancellationToken?: CancellationToken): Promise<void>

Waits until a condition becomes true.

```typescript
// Wait for player to be ready
await timerMgr.waitUntil(() => {
    const player = Game.getCurrentPlayer();
    return player && player.isReady();
});

// Wait for animation to complete
await timerMgr.waitUntil(() => {
    const animationComponent = entity.getComponent(AnimationComponent);
    return !animationComponent.isPlaying();
});

// Wait with timeout
const token = CancellationToken.create();
TimerMgr.get().newOnceTimer(10000, () => token.cancel()); // 10 second timeout

try {
    await timerMgr.waitUntil(() => {
        return GameState.isLoaded();
    }, token);
} catch (error) {
    console.log('Timeout waiting for game to load');
}
```

### Task

Represents an asynchronous operation that can be awaited.

```typescript
class Task<T = void> {
    static create<T>(): Task<T>;
    setResult(result?: T): void;
    setException(error: Error): void;
    then<U>(onFulfilled?: (value: T) => U | Promise<U>): Promise<U>;
    catch<U>(onRejected?: (reason: any) => U | Promise<U>): Promise<U>;
}
```

#### Creating and Using Tasks

```typescript
import { Task } from 'moye-cocos';

// Create a task
const task = Task.create<number>();

// Set result later
setTimeout(() => {
    task.setResult(42);
}, 1000);

// Await the task
const result = await task;
console.log(result); // 42

// Task with error handling
const errorTask = Task.create<string>();

setTimeout(() => {
    errorTask.setException(new Error('Something went wrong'));
}, 500);

try {
    await errorTask;
} catch (error) {
    console.error('Task failed:', error.message);
}
```

#### Practical Examples

```typescript
class AssetLoader extends Entity {
    private loadingTasks: Map<string, Task<any>> = new Map();
    
    async loadAsset<T>(path: string): Promise<T> {
        // Check if already loading
        if (this.loadingTasks.has(path)) {
            return await this.loadingTasks.get(path);
        }
        
        // Create new loading task
        const task = Task.create<T>();
        this.loadingTasks.set(path, task);
        
        // Load asset
        resources.load(path, (err, asset) => {
            this.loadingTasks.delete(path);
            
            if (err) {
                task.setException(new Error(`Failed to load ${path}: ${err.message}`));
            } else {
                task.setResult(asset as T);
            }
        });
        
        return await task;
    }
}

class NetworkRequest extends Entity {
    async makeRequest<T>(url: string): Promise<T> {
        const task = Task.create<T>();
        
        const xhr = new XMLHttpRequest();
        xhr.open('GET', url);
        
        xhr.onload = () => {
            if (xhr.status === 200) {
                task.setResult(JSON.parse(xhr.responseText));
            } else {
                task.setException(new Error(`HTTP ${xhr.status}: ${xhr.statusText}`));
            }
        };
        
        xhr.onerror = () => {
            task.setException(new Error('Network error'));
        };
        
        xhr.send();
        return await task;
    }
}
```

### CancellationToken

Provides a mechanism to cancel asynchronous operations.

```typescript
class CancellationToken {
    static create(): CancellationToken;
    cancel(reason?: string): void;
    isCancelled(): boolean;
    throwIfCancelled(): void;
    register(callback: (reason?: string) => void): void;
}
```

#### Basic Usage

```typescript
import { CancellationToken } from 'moye-cocos';

// Create a cancellation token
const token = CancellationToken.create();

// Cancel after 5 seconds
TimerMgr.get().newOnceTimer(5000, () => {
    token.cancel('Timeout');
});

// Use with async operations
try {
    await TimerMgr.get().waitAsync(10000, token);
    console.log('Wait completed');
} catch (error) {
    console.log('Operation cancelled:', error.message);
}
```

#### Advanced Cancellation Patterns

```typescript
class DownloadManager extends Entity {
    private activeDownloads: Map<string, CancellationToken> = new Map();
    
    async downloadFile(url: string, filename: string): Promise<void> {
        // Cancel any existing download of the same file
        if (this.activeDownloads.has(filename)) {
            this.activeDownloads.get(filename).cancel('New download started');
        }
        
        const token = CancellationToken.create();
        this.activeDownloads.set(filename, token);
        
        try {
            await this.performDownload(url, filename, token);
            console.log(`Download completed: ${filename}`);
        } catch (error) {
            if (token.isCancelled()) {
                console.log(`Download cancelled: ${filename}`);
            } else {
                console.error(`Download failed: ${filename}`, error);
            }
        } finally {
            this.activeDownloads.delete(filename);
        }
    }
    
    private async performDownload(url: string, filename: string, token: CancellationToken): Promise<void> {
        const task = Task.create<void>();
        
        // Register cancellation handler
        token.register(() => {
            task.setException(new Error('Download cancelled'));
        });
        
        // Simulate download with progress
        let progress = 0;
        const progressTimer = TimerMgr.get().newRepeatedTimer(100, () => {
            if (token.isCancelled()) {
                TimerMgr.get().removeTimer(progressTimer);
                return;
            }
            
            progress += Math.random() * 10;
            EventSystem.get().publish(
                this.domain,
                DownloadProgressEvent.create(filename, progress)
            );
            
            if (progress >= 100) {
                TimerMgr.get().removeTimer(progressTimer);
                task.setResult();
            }
        });
        
        await task;
    }
    
    cancelDownload(filename: string) {
        const token = this.activeDownloads.get(filename);
        if (token) {
            token.cancel('User requested cancellation');
        }
    }
    
    cancelAllDownloads() {
        for (const [filename, token] of this.activeDownloads) {
            token.cancel('All downloads cancelled');
        }
    }
}
```

## Advanced Patterns

### Coroutine-like Operations

```typescript
class CoroutineManager extends Entity {
    async fadeIn(node: Node, duration: number, token?: CancellationToken): Promise<void> {
        const startOpacity = node.opacity;
        const targetOpacity = 255;
        const frameTime = 16; // ~60 FPS
        const totalFrames = duration / frameTime;
        
        for (let frame = 0; frame <= totalFrames; frame++) {
            token?.throwIfCancelled();
            
            const progress = frame / totalFrames;
            const currentOpacity = startOpacity + (targetOpacity - startOpacity) * progress;
            node.opacity = currentOpacity;
            
            if (frame < totalFrames) {
                await TimerMgr.get().waitFrameAsync(token);
            }
        }
    }
    
    async moveToPosition(node: Node, targetPos: Vec3, duration: number, token?: CancellationToken): Promise<void> {
        const startPos = node.position.clone();
        const frameTime = 16;
        const totalFrames = duration / frameTime;
        
        for (let frame = 0; frame <= totalFrames; frame++) {
            token?.throwIfCancelled();
            
            const progress = frame / totalFrames;
            const currentPos = startPos.lerp(targetPos, progress);
            node.position = currentPos;
            
            if (frame < totalFrames) {
                await TimerMgr.get().waitFrameAsync(token);
            }
        }
    }
    
    async sequence(...operations: Array<() => Promise<void>>): Promise<void> {
        for (const operation of operations) {
            await operation();
        }
    }
    
    async parallel(...operations: Array<() => Promise<void>>): Promise<void> {
        await Promise.all(operations.map(op => op()));
    }
}

// Usage
const coroutines = new CoroutineManager();
const token = CancellationToken.create();

// Sequence of operations
await coroutines.sequence(
    () => coroutines.fadeIn(node, 1000, token),
    () => coroutines.moveToPosition(node, new Vec3(100, 100, 0), 2000, token),
    () => coroutines.fadeOut(node, 1000, token)
);

// Parallel operations
await coroutines.parallel(
    () => coroutines.fadeIn(node1, 1000),
    () => coroutines.fadeIn(node2, 1000),
    () => coroutines.fadeIn(node3, 1000)
);
```

### Timer Pools and Optimization

```typescript
class OptimizedTimerManager extends Singleton {
    private timerPool: Timer[] = [];
    private activeTimers: Set<Timer> = new Set();
    
    createTimer(interval: number, callback: Action, repeat: boolean = false): Timer {
        let timer = this.timerPool.pop();
        
        if (!timer) {
            timer = new Timer();
        }
        
        timer.interval = interval;
        timer.callback = callback;
        timer.repeat = repeat;
        timer.expireTime = Date.now() + interval;
        timer.isActive = true;
        
        this.activeTimers.add(timer);
        return timer;
    }
    
    destroyTimer(timer: Timer) {
        if (this.activeTimers.has(timer)) {
            this.activeTimers.delete(timer);
            timer.reset();
            this.timerPool.push(timer);
        }
    }
    
    update() {
        const now = Date.now();
        const expiredTimers: Timer[] = [];
        
        for (const timer of this.activeTimers) {
            if (timer.expireTime <= now) {
                expiredTimers.push(timer);
            }
        }
        
        for (const timer of expiredTimers) {
            timer.callback();
            
            if (timer.repeat) {
                timer.expireTime = now + timer.interval;
            } else {
                this.destroyTimer(timer);
            }
        }
    }
}
```

### Scheduled Task Queue

```typescript
class TaskScheduler extends Singleton {
    private taskQueue: Array<{ task: () => Promise<void>, priority: number, delay: number }> = [];
    private running: boolean = false;
    
    schedule(task: () => Promise<void>, priority: number = 0, delay: number = 0) {
        this.taskQueue.push({ task, priority, delay });
        this.taskQueue.sort((a, b) => b.priority - a.priority); // Higher priority first
        
        if (!this.running) {
            this.processQueue();
        }
    }
    
    private async processQueue() {
        this.running = true;
        
        while (this.taskQueue.length > 0) {
            const { task, delay } = this.taskQueue.shift()!;
            
            if (delay > 0) {
                await TimerMgr.get().waitAsync(delay);
            }
            
            try {
                await task();
            } catch (error) {
                Logger.get().error('Scheduled task failed:', error);
            }
            
            // Yield control periodically
            await TimerMgr.get().waitFrameAsync();
        }
        
        this.running = false;
    }
    
    clear() {
        this.taskQueue = [];
    }
}

// Usage
const scheduler = TaskScheduler.get();

// High priority immediate task
scheduler.schedule(async () => {
    await GameManager.get().saveGame();
}, 10);

// Low priority delayed task
scheduler.schedule(async () => {
    await AssetManager.get().preloadAssets();
}, 1, 5000);
```

## Best Practices

### 1. Resource Management

```typescript
class TimerResource {
    private timers: Set<number> = new Set();
    private tokens: Set<CancellationToken> = new Set();
    
    createTimer(interval: number, callback: Action): number {
        const timerId = TimerMgr.get().newRepeatedTimer(interval, callback);
        this.timers.add(timerId);
        return timerId;
    }
    
    createToken(): CancellationToken {
        const token = CancellationToken.create();
        this.tokens.add(token);
        return token;
    }
    
    cleanup() {
        // Cancel all timers
        for (const timerId of this.timers) {
            TimerMgr.get().removeTimer(timerId);
        }
        this.timers.clear();
        
        // Cancel all tokens
        for (const token of this.tokens) {
            if (!token.isCancelled()) {
                token.cancel('Cleanup');
            }
        }
        this.tokens.clear();
    }
}

class GameEntity extends Entity {
    private timerResource = new TimerResource();
    
    awake() {
        // Use timer resource instead of direct timer creation
        this.timerResource.createTimer(1000, () => {
            this.updateHealth();
        });
    }
    
    destroy() {
        // Automatically cleanup all timers and tokens
        this.timerResource.cleanup();
        super.destroy();
    }
}
```

### 2. Error Handling

```typescript
async function robustAsyncOperation(token?: CancellationToken): Promise<void> {
    try {
        // Check cancellation before starting
        token?.throwIfCancelled();
        
        await TimerMgr.get().waitAsync(1000, token);
        
        // Check cancellation during operation
        token?.throwIfCancelled();
        
        // Perform work
        await performWork();
        
    } catch (error) {
        if (token?.isCancelled()) {
            Logger.get().debug('Operation cancelled');
        } else {
            Logger.get().error('Operation failed:', error);
            throw error; // Re-throw non-cancellation errors
        }
    }
}
```

### 3. Performance Optimization

```typescript
class PerformantAnimationManager {
    private animatingNodes: Set<Node> = new Set();
    private animationTimer: number | null = null;
    
    startAnimation(node: Node, animation: (progress: number) => void, duration: number) {
        this.animatingNodes.add(node);
        
        const startTime = Date.now();
        node['_animation'] = { animation, startTime, duration };
        
        // Start global animation timer if not already running
        if (!this.animationTimer) {
            this.animationTimer = TimerMgr.get().newRepeatedTimer(16, () => {
                this.updateAnimations();
            });
        }
    }
    
    private updateAnimations() {
        const now = Date.now();
        const completedNodes: Node[] = [];
        
        for (const node of this.animatingNodes) {
            const anim = node['_animation'];
            const elapsed = now - anim.startTime;
            const progress = Math.min(elapsed / anim.duration, 1);
            
            anim.animation(progress);
            
            if (progress >= 1) {
                completedNodes.push(node);
            }
        }
        
        // Remove completed animations
        for (const node of completedNodes) {
            this.animatingNodes.delete(node);
            delete node['_animation'];
        }
        
        // Stop timer if no more animations
        if (this.animatingNodes.size === 0 && this.animationTimer) {
            TimerMgr.get().removeTimer(this.animationTimer);
            this.animationTimer = null;
        }
    }
}
```

### 4. Debugging and Monitoring

```typescript
class TimerDebugger extends Singleton {
    private timerStats: Map<number, { created: number, calls: number, totalTime: number }> = new Map();
    
    wrapTimer(interval: number, callback: Action): number {
        const wrappedCallback = () => {
            const startTime = Date.now();
            callback();
            const endTime = Date.now();
            
            const stats = this.timerStats.get(timerId);
            if (stats) {
                stats.calls++;
                stats.totalTime += (endTime - startTime);
            }
        };
        
        const timerId = TimerMgr.get().newRepeatedTimer(interval, wrappedCallback);
        
        this.timerStats.set(timerId, {
            created: Date.now(),
            calls: 0,
            totalTime: 0
        });
        
        return timerId;
    }
    
    getTimerStats(): Array<{ id: number, avgTime: number, calls: number, age: number }> {
        const now = Date.now();
        const stats: Array<{ id: number, avgTime: number, calls: number, age: number }> = [];
        
        for (const [id, stat] of this.timerStats) {
            stats.push({
                id,
                avgTime: stat.calls > 0 ? stat.totalTime / stat.calls : 0,
                calls: stat.calls,
                age: now - stat.created
            });
        }
        
        return stats.sort((a, b) => b.avgTime - a.avgTime);
    }
}
```