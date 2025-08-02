# Logger System Documentation

## Overview

The Moye Logger System provides comprehensive logging capabilities with configurable levels, formatted output, and debugging utilities. It supports multiple log levels, custom formatters, and pluggable output destinations.

## Core Components

### Logger

Main singleton class for logging operations.

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

### LoggerLevel

Enumeration of logging levels.

```typescript
enum LoggerLevel {
    Debug = 0,
    Log = 1,
    Warn = 2,
    Error = 3
}
```

### Global Logging Functions

Convenient global functions for logging.

```typescript
function debug(...args: any[]): void;
function debugF(str: string, ...args: any[]): void;
function log(...args: any[]): void;
function logF(str: string, ...args: any[]): void;
function warn(...args: any[]): void;
function warnF(str: string, ...args: any[]): void;
function error(...args: any[]): void;
function errorF(str: string, ...args: any[]): void;
```

## Basic Usage

### Simple Logging

```typescript
import { log, debug, warn, error } from 'moye-cocos';

// Basic logging
log("Game started");
debug("Player position:", 100, 200);
warn("Low health warning");
error("Critical error occurred");

// Using Logger singleton directly
import { Logger } from 'moye-cocos';

Logger.get().log("Direct logger usage");
Logger.get().debug("Debug message", { data: "example" });
```

### Formatted Logging

The framework supports C#-style string formatting with `{0}`, `{1}`, etc. placeholders:

```typescript
import { logF, debugF, warnF, errorF } from 'moye-cocos';

// Basic formatting
logF("Player {0} scored {1} points", "Alice", 1500);
// Output: Player Alice scored 1500 points

debugF("Position: ({0}, {1})", x, y);
// Output: Position: (100, 200)

warnF("Health low: {0}/{1}", currentHealth, maxHealth);
// Output: Health low: 25/100

errorF("Failed to load asset: {0}", assetPath);
// Output: Failed to load asset: textures/player.png

// Multiple references to same parameter
logF("Player {0} defeated {1}. {0} gains experience!", "Hero", "Dragon");
// Output: Player Hero defeated Dragon. Hero gains experience!

// Escaping braces
logF("JSON example: {{\"player\": \"{0}\"}}", playerName);
// Output: JSON example: {"player": "Alice"}
```

### Logger Configuration

```typescript
import { Logger, LoggerLevel } from 'moye-cocos';

const logger = Logger.get();

// Set log level (only messages at this level or higher will be shown)
logger.level = LoggerLevel.Warn; // Only show warnings and errors
logger.level = LoggerLevel.Debug; // Show all messages
logger.level = LoggerLevel.Error; // Only show errors

// Check current level
if (logger.level <= LoggerLevel.Debug) {
    console.log("Debug logging is enabled");
}
```

## Advanced Features

### Custom Log Implementation

You can provide your own logging implementation by implementing the `ILog` interface:

```typescript
interface ILog {
    debug(...args: any[]): void;
    log(...args: any[]): void;
    warn(...args: any[]): void;
    error(...args: any[]): void;
}

// Custom logger that writes to file
class FileLogger implements ILog {
    private logFile: string = 'game.log';
    
    debug(...args: any[]): void {
        this.writeToFile('DEBUG', args);
    }
    
    log(...args: any[]): void {
        this.writeToFile('INFO', args);
    }
    
    warn(...args: any[]): void {
        this.writeToFile('WARN', args);
    }
    
    error(...args: any[]): void {
        this.writeToFile('ERROR', args);
    }
    
    private writeToFile(level: string, args: any[]): void {
        const timestamp = new Date().toISOString();
        const message = args.map(arg => 
            typeof arg === 'object' ? JSON.stringify(arg) : String(arg)
        ).join(' ');
        
        const logEntry = `[${timestamp}] [${level}] ${message}\n`;
        
        // In browser environment, you might use localStorage or IndexedDB
        // In Node.js environment, you would use fs.appendFile
        console.log(logEntry);
    }
}

// Use custom logger
Logger.get().iLog = new FileLogger();
```

### Enhanced Logger with Context

```typescript
class ContextualLogger implements ILog {
    private context: string;
    
    constructor(context: string) {
        this.context = context;
    }
    
    debug(...args: any[]): void {
        console.log(`[DEBUG] [${this.context}]`, ...args);
    }
    
    log(...args: any[]): void {
        console.log(`[INFO] [${this.context}]`, ...args);
    }
    
    warn(...args: any[]): void {
        console.warn(`[WARN] [${this.context}]`, ...args);
    }
    
    error(...args: any[]): void {
        console.error(`[ERROR] [${this.context}]`, ...args);
    }
}

// Usage with different contexts
class GameManager extends Singleton {
    private logger = new ContextualLogger('GameManager');
    
    start() {
        this.logger.log('Game starting...');
        Logger.get().iLog = this.logger;
    }
}

class PlayerController extends Entity {
    private logger = new ContextualLogger('PlayerController');
    
    awake() {
        this.logger.debug('Player controller initialized');
    }
}
```

### Conditional Logging

```typescript
import { DEVELOP, MOYE_OUTPUT_DEBUG, MOYE_OUTPUT_LOG } from 'moye-cocos';

class ConditionalLogger {
    static debugDev(...args: any[]): void {
        if (DEVELOP && MOYE_OUTPUT_DEBUG) {
            debug(...args);
        }
    }
    
    static logDev(...args: any[]): void {
        if (DEVELOP && MOYE_OUTPUT_LOG) {
            log(...args);
        }
    }
    
    static performance<T>(label: string, operation: () => T): T {
        if (DEVELOP) {
            const start = performance.now();
            const result = operation();
            const end = performance.now();
            debugF('Performance [{0}]: {1}ms', label, (end - start).toFixed(2));
            return result;
        }
        return operation();
    }
    
    static async performanceAsync<T>(label: string, operation: () => Promise<T>): Promise<T> {
        if (DEVELOP) {
            const start = performance.now();
            const result = await operation();
            const end = performance.now();
            debugF('Performance [{0}]: {1}ms', label, (end - start).toFixed(2));
            return result;
        }
        return await operation();
    }
}

// Usage
ConditionalLogger.debugDev('Debug info only in development');

const result = ConditionalLogger.performance('Database Query', () => {
    return database.query('SELECT * FROM players');
});

const asyncResult = await ConditionalLogger.performanceAsync('API Call', async () => {
    return await fetch('/api/data');
});
```

## Specialized Loggers

### Network Logger

```typescript
class NetworkLogger {
    static logRequest(method: string, url: string, data?: any): void {
        debugF('→ {0} {1}', method, url);
        if (data) {
            debug('Request data:', data);
        }
    }
    
    static logResponse(method: string, url: string, status: number, data?: any): void {
        if (status >= 200 && status < 300) {
            debugF('← {0} {1} [{2}] Success', method, url, status);
        } else {
            warnF('← {0} {1} [{2}] Error', method, url, status);
        }
        
        if (data) {
            debug('Response data:', data);
        }
    }
    
    static logWebSocketEvent(event: string, data?: any): void {
        debugF('WebSocket {0}', event);
        if (data) {
            debug('WebSocket data:', data);
        }
    }
}

// Usage in network code
NetworkLogger.logRequest('POST', '/api/login', { username: 'player1' });
NetworkLogger.logResponse('POST', '/api/login', 200, { token: 'abc123' });
NetworkLogger.logWebSocketEvent('message', { type: 'player_move', x: 100, y: 200 });
```

### Game State Logger

```typescript
class GameStateLogger {
    static logPlayerAction(playerId: string, action: string, data?: any): void {
        logF('Player {0} performed action: {1}', playerId, action);
        if (data) {
            debug('Action data:', data);
        }
    }
    
    static logGameEvent(event: string, details?: any): void {
        logF('Game event: {0}', event);
        if (details) {
            debug('Event details:', details);
        }
    }
    
    static logSceneTransition(from: string, to: string): void {
        logF('Scene transition: {0} → {1}', from, to);
    }
    
    static logResourceUsage(): void {
        if (DEVELOP) {
            const memInfo = (performance as any).memory;
            if (memInfo) {
                debugF('Memory usage: {0}MB used, {1}MB total', 
                    Math.round(memInfo.usedJSHeapSize / 1024 / 1024),
                    Math.round(memInfo.totalJSHeapSize / 1024 / 1024)
                );
            }
        }
    }
}

// Usage
GameStateLogger.logPlayerAction('player1', 'move', { x: 100, y: 200 });
GameStateLogger.logGameEvent('level_completed', { level: 5, score: 1500 });
GameStateLogger.logSceneTransition('menu', 'game');
GameStateLogger.logResourceUsage();
```

### Entity Lifecycle Logger

```typescript
class EntityLifecycleLogger {
    static logEntityCreated(entity: Entity): void {
        debugF('Entity created: {0} (ID: {1})', entity.constructor.name, entity.id);
    }
    
    static logEntityDestroyed(entity: Entity): void {
        debugF('Entity destroyed: {0} (ID: {1})', entity.constructor.name, entity.id);
    }
    
    static logComponentAdded(entity: Entity, component: Entity): void {
        debugF('Component added to {0}: {1}', entity.constructor.name, component.constructor.name);
    }
    
    static logComponentRemoved(entity: Entity, componentType: string): void {
        debugF('Component removed from {0}: {1}', entity.constructor.name, componentType);
    }
}

// Integration with Entity system
class LoggedEntity extends Entity {
    awake() {
        EntityLifecycleLogger.logEntityCreated(this);
        super.awake();
    }
    
    destroy() {
        EntityLifecycleLogger.logEntityDestroyed(this);
        super.destroy();
    }
    
    addComponent<T extends Entity>(type: new() => T): T {
        const component = super.addComponent(type);
        EntityLifecycleLogger.logComponentAdded(this, component);
        return component;
    }
    
    removeComponent<T extends Entity>(type: new() => T): void {
        EntityLifecycleLogger.logComponentRemoved(this, type.name);
        super.removeComponent(type);
    }
}
```

## Debug Utilities

### Assert Functions

```typescript
class Assert {
    static isTrue(condition: boolean, message?: string): void {
        if (!condition) {
            const msg = message || 'Assertion failed: condition is false';
            error(msg);
            throw new Error(msg);
        }
    }
    
    static isFalse(condition: boolean, message?: string): void {
        Assert.isTrue(!condition, message || 'Assertion failed: condition is true');
    }
    
    static isNotNull<T>(value: T | null | undefined, message?: string): T {
        if (value == null) {
            const msg = message || 'Assertion failed: value is null or undefined';
            error(msg);
            throw new Error(msg);
        }
        return value;
    }
    
    static equals<T>(expected: T, actual: T, message?: string): void {
        if (expected !== actual) {
            const msg = message || `Assertion failed: expected ${expected}, got ${actual}`;
            error(msg);
            throw new Error(msg);
        }
    }
    
    static isType<T>(value: any, type: new() => T, message?: string): T {
        if (!(value instanceof type)) {
            const msg = message || `Assertion failed: value is not of type ${type.name}`;
            error(msg);
            throw new Error(msg);
        }
        return value as T;
    }
}

// Usage
function processPlayer(player: Entity | null) {
    const validPlayer = Assert.isNotNull(player, 'Player cannot be null');
    const healthComponent = Assert.isNotNull(
        validPlayer.getComponent(HealthComponent),
        'Player must have health component'
    );
    
    Assert.isTrue(healthComponent.health > 0, 'Player must be alive');
    
    // Process player...
}
```

### Debug Drawing

```typescript
class DebugDraw {
    private static enabled: boolean = DEVELOP;
    private static debugNode: Node | null = null;
    
    static setEnabled(enabled: boolean): void {
        this.enabled = enabled;
    }
    
    static getDebugNode(): Node {
        if (!this.debugNode) {
            this.debugNode = new Node('DebugDraw');
            find('Canvas')?.addChild(this.debugNode);
        }
        return this.debugNode;
    }
    
    static drawCircle(center: Vec2, radius: number, color: Color = Color.RED): void {
        if (!this.enabled) return;
        
        debugF('Drawing circle at ({0}, {1}) with radius {2}', center.x, center.y, radius);
        
        // Create visual debug circle (implementation depends on your needs)
        const graphics = this.getDebugNode().getComponent(Graphics) || 
                        this.getDebugNode().addComponent(Graphics);
        
        graphics.circle(center.x, center.y, radius);
        graphics.fillColor = color;
        graphics.fill();
    }
    
    static drawLine(from: Vec2, to: Vec2, color: Color = Color.GREEN): void {
        if (!this.enabled) return;
        
        debugF('Drawing line from ({0}, {1}) to ({2}, {3})', from.x, from.y, to.x, to.y);
        
        const graphics = this.getDebugNode().getComponent(Graphics) || 
                        this.getDebugNode().addComponent(Graphics);
        
        graphics.moveTo(from.x, from.y);
        graphics.lineTo(to.x, to.y);
        graphics.strokeColor = color;
        graphics.stroke();
    }
    
    static drawText(position: Vec2, text: string, color: Color = Color.WHITE): void {
        if (!this.enabled) return;
        
        debugF('Drawing text at ({0}, {1}): {2}', position.x, position.y, text);
        
        const textNode = new Node('DebugText');
        const label = textNode.addComponent(Label);
        label.string = text;
        label.color = color;
        label.fontSize = 12;
        
        textNode.position = new Vec3(position.x, position.y, 0);
        this.getDebugNode().addChild(textNode);
        
        // Remove after 5 seconds
        TimerMgr.get().newOnceTimer(5000, () => {
            if (textNode.isValid) {
                textNode.destroy();
            }
        });
    }
    
    static clear(): void {
        if (this.debugNode) {
            this.debugNode.destroyAllChildren();
            const graphics = this.debugNode.getComponent(Graphics);
            if (graphics) {
                graphics.clear();
            }
        }
    }
}

// Usage
DebugDraw.drawCircle(new Vec2(100, 100), 50, Color.BLUE);
DebugDraw.drawLine(new Vec2(0, 0), new Vec2(200, 200), Color.GREEN);
DebugDraw.drawText(new Vec2(150, 50), 'Player Position');
```

### Performance Profiler

```typescript
class Profiler {
    private static profiles: Map<string, {
        totalTime: number;
        calls: number;
        startTime?: number;
    }> = new Map();
    
    static start(name: string): void {
        if (!DEVELOP) return;
        
        let profile = this.profiles.get(name);
        if (!profile) {
            profile = { totalTime: 0, calls: 0 };
            this.profiles.set(name, profile);
        }
        
        profile.startTime = performance.now();
    }
    
    static end(name: string): void {
        if (!DEVELOP) return;
        
        const profile = this.profiles.get(name);
        if (profile && profile.startTime !== undefined) {
            const elapsed = performance.now() - profile.startTime;
            profile.totalTime += elapsed;
            profile.calls++;
            profile.startTime = undefined;
            
            debugF('Profile [{0}]: {1}ms (call #{2})', name, elapsed.toFixed(2), profile.calls);
        }
    }
    
    static measure<T>(name: string, operation: () => T): T {
        this.start(name);
        try {
            return operation();
        } finally {
            this.end(name);
        }
    }
    
    static async measureAsync<T>(name: string, operation: () => Promise<T>): Promise<T> {
        this.start(name);
        try {
            return await operation();
        } finally {
            this.end(name);
        }
    }
    
    static getReport(): string {
        if (!DEVELOP) return 'Profiling not available in production';
        
        const lines = ['=== Performance Report ==='];
        
        for (const [name, profile] of this.profiles) {
            const avgTime = profile.calls > 0 ? profile.totalTime / profile.calls : 0;
            lines.push(`${name}: ${profile.calls} calls, ${avgTime.toFixed(2)}ms avg, ${profile.totalTime.toFixed(2)}ms total`);
        }
        
        return lines.join('\n');
    }
    
    static logReport(): void {
        log(this.getReport());
    }
    
    static reset(): void {
        this.profiles.clear();
    }
}

// Usage
class GameSystem {
    update(dt: number) {
        Profiler.measure('GameSystem.update', () => {
            // Game update logic
            this.updateEntities(dt);
            this.updatePhysics(dt);
            this.updateRendering(dt);
        });
    }
    
    async loadLevel(levelId: string) {
        await Profiler.measureAsync('Level.load', async () => {
            // Load level assets
            const assets = await this.loadAssets(levelId);
            this.setupLevel(assets);
        });
    }
}

// Generate report periodically
TimerMgr.get().newRepeatedTimer(30000, () => {
    Profiler.logReport();
});
```

## Best Practices

### 1. Structured Logging

```typescript
interface LogContext {
    component?: string;
    playerId?: string;
    sessionId?: string;
    level?: string;
    timestamp?: number;
}

class StructuredLogger {
    static logWithContext(level: 'debug' | 'log' | 'warn' | 'error', 
                         message: string, context?: LogContext, data?: any): void {
        const logEntry = {
            level,
            message,
            timestamp: Date.now(),
            ...context,
            data
        };
        
        const logger = Logger.get();
        switch (level) {
            case 'debug':
                logger.debug(JSON.stringify(logEntry));
                break;
            case 'log':
                logger.log(JSON.stringify(logEntry));
                break;
            case 'warn':
                logger.warn(JSON.stringify(logEntry));
                break;
            case 'error':
                logger.error(JSON.stringify(logEntry));
                break;
        }
    }
}

// Usage
StructuredLogger.logWithContext('log', 'Player action performed', {
    component: 'PlayerController',
    playerId: 'player_123',
    sessionId: 'session_456'
}, { action: 'move', x: 100, y: 200 });
```

### 2. Log Rotation

```typescript
class RotatingLogger implements ILog {
    private logs: string[] = [];
    private maxLogs: number = 1000;
    private logFile: string = 'game.log';
    
    debug(...args: any[]): void {
        this.addLog('DEBUG', args);
    }
    
    log(...args: any[]): void {
        this.addLog('INFO', args);
    }
    
    warn(...args: any[]): void {
        this.addLog('WARN', args);
    }
    
    error(...args: any[]): void {
        this.addLog('ERROR', args);
    }
    
    private addLog(level: string, args: any[]): void {
        const timestamp = new Date().toISOString();
        const message = args.map(arg => 
            typeof arg === 'object' ? JSON.stringify(arg) : String(arg)
        ).join(' ');
        
        const logEntry = `[${timestamp}] [${level}] ${message}`;
        
        this.logs.push(logEntry);
        
        // Rotate logs if needed
        if (this.logs.length > this.maxLogs) {
            this.logs = this.logs.slice(-this.maxLogs / 2); // Keep last half
        }
        
        // Also output to console
        console.log(logEntry);
    }
    
    exportLogs(): string {
        return this.logs.join('\n');
    }
    
    clearLogs(): void {
        this.logs = [];
    }
}
```

### 3. Environment-specific Configuration

```typescript
class LoggerConfig {
    static configure(): void {
        const logger = Logger.get();
        
        if (DEVELOP) {
            // Development: verbose logging
            logger.level = LoggerLevel.Debug;
            logger.iLog = new ConsoleLogger();
        } else {
            // Production: minimal logging
            logger.level = LoggerLevel.Error;
            logger.iLog = new RemoteLogger();
        }
    }
}

class ConsoleLogger implements ILog {
    debug(...args: any[]): void {
        console.debug('[DEBUG]', ...args);
    }
    
    log(...args: any[]): void {
        console.log('[INFO]', ...args);
    }
    
    warn(...args: any[]): void {
        console.warn('[WARN]', ...args);
    }
    
    error(...args: any[]): void {
        console.error('[ERROR]', ...args);
    }
}

class RemoteLogger implements ILog {
    private errorQueue: any[] = [];
    
    debug(...args: any[]): void {
        // No debug logging in production
    }
    
    log(...args: any[]): void {
        // No info logging in production
    }
    
    warn(...args: any[]): void {
        // No warning logging in production
    }
    
    error(...args: any[]): void {
        const errorData = {
            level: 'error',
            message: args.join(' '),
            timestamp: Date.now(),
            userAgent: navigator.userAgent,
            url: window.location.href
        };
        
        this.errorQueue.push(errorData);
        
        // Send errors to remote logging service
        this.flushErrors();
    }
    
    private async flushErrors(): Promise<void> {
        if (this.errorQueue.length === 0) return;
        
        const errors = [...this.errorQueue];
        this.errorQueue = [];
        
        try {
            await NetServices.get().httpPost('/api/logs/errors', { errors });
        } catch (err) {
            // Re-queue errors if send fails
            this.errorQueue.unshift(...errors);
        }
    }
}

// Configure logger at application start
LoggerConfig.configure();
```