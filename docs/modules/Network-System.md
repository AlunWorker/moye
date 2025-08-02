# Network System Documentation

## Overview

The Moye Network System provides comprehensive networking capabilities including HTTP requests, WebSocket connections, and message handling. It supports both client-server and peer-to-peer communication patterns.

## Core Components

### NetServices

Singleton service for HTTP requests and RESTful API communication.

```typescript
class NetServices extends Singleton {
    httpRequest<T>(url: string, options?: RequestOptions): Promise<T>;
    httpPost<T>(url: string, data: any, options?: RequestOptions): Promise<T>;
    httpGet<T>(url: string, options?: RequestOptions): Promise<T>;
    httpPut<T>(url: string, data: any, options?: RequestOptions): Promise<T>;
    httpDelete<T>(url: string, options?: RequestOptions): Promise<T>;
}
```

#### HTTP Methods

##### httpGet<T>(url: string, options?: RequestOptions): Promise<T>

Performs a GET request.

```typescript
import { NetServices } from 'moye-cocos';

const netService = NetServices.get();

// Get user profile
const profile = await netService.httpGet<UserProfile>('/api/user/profile');

// Get with query parameters
const users = await netService.httpGet<User[]>('/api/users?page=1&limit=10');

// Get with headers
const data = await netService.httpGet<GameData>('/api/game/data', {
    headers: {
        'Authorization': 'Bearer ' + token,
        'Content-Type': 'application/json'
    }
});
```

##### httpPost<T>(url: string, data: any, options?: RequestOptions): Promise<T>

Performs a POST request with data.

```typescript
// User login
const loginResult = await netService.httpPost<LoginResponse>('/api/auth/login', {
    username: 'player1',
    password: 'password123'
});

// Create new game session
const session = await netService.httpPost<GameSession>('/api/game/sessions', {
    gameMode: 'battle_royale',
    maxPlayers: 100,
    map: 'island_01'
});

// Upload score
const scoreResult = await netService.httpPost<ScoreResponse>('/api/scores', {
    playerId: 'player1',
    score: 1500,
    gameMode: 'arcade'
});
```

##### httpPut<T>(url: string, data: any, options?: RequestOptions): Promise<T>

Updates existing resources.

```typescript
// Update user profile
const updatedProfile = await netService.httpPut<UserProfile>('/api/user/profile', {
    displayName: 'NewPlayerName',
    avatar: 'avatar_02',
    settings: {
        soundEnabled: true,
        musicVolume: 0.8
    }
});
```

##### httpDelete<T>(url: string, options?: RequestOptions): Promise<T>

Deletes resources.

```typescript
// Delete game session
await netService.httpDelete('/api/game/sessions/session123');

// Delete user account
await netService.httpDelete('/api/user/account', {
    headers: { 'Authorization': 'Bearer ' + token }
});
```

#### Request Options

```typescript
interface RequestOptions {
    headers?: Record<string, string>;
    timeout?: number;
    retries?: number;
    cache?: boolean;
    credentials?: 'include' | 'omit' | 'same-origin';
}
```

### WebSocket System

#### SessionCom

Component for managing WebSocket connections.

```typescript
class SessionCom extends Entity {
    connect(url: string, protocols?: string[]): Promise<void>;
    disconnect(): void;
    send(message: any): void;
    isConnected(): boolean;
}
```

##### Basic WebSocket Usage

```typescript
import { SessionCom } from 'moye-cocos';

class NetworkManager extends Entity {
    private session: SessionCom;
    
    async awake() {
        this.session = this.addComponent(SessionCom);
        await this.connectToServer();
    }
    
    async connectToServer() {
        try {
            await this.session.connect('ws://localhost:8080/game');
            console.log('Connected to game server');
            
            // Send initial authentication
            this.session.send({
                type: 'auth',
                token: this.getAuthToken()
            });
            
        } catch (error) {
            console.error('Failed to connect:', error);
            this.handleConnectionError(error);
        }
    }
    
    private getAuthToken(): string {
        // Get stored authentication token
        return localStorage.getItem('authToken') || '';
    }
    
    private handleConnectionError(error: any) {
        EventSystem.get().publish(
            this.domain,
            NetworkErrorEvent.create('connection_failed', error.message)
        );
    }
}
```

#### Session

Lower-level session management.

```typescript
class Session extends Entity {
    sessionId: string;
    remoteAddress: IPEndPoint;
    
    send(message: IMessage): void;
    disconnect(): void;
    isConnected(): boolean;
}
```

### Message System

#### Message Handlers

Use decorators to register message handlers.

```typescript
import { AMHandler, MsgHandler } from 'moye-cocos';

// Define message types
class LoginMessage {
    username: string;
    password: string;
    clientVersion: string;
    
    static create(username: string, password: string, version: string = '1.0.0'): LoginMessage {
        const msg = new LoginMessage();
        msg.username = username;
        msg.password = password;
        msg.clientVersion = version;
        return msg;
    }
}

class LoginResponseMessage {
    success: boolean;
    playerId?: string;
    token?: string;
    errorMessage?: string;
    serverTime: number;
}

class PlayerMoveMessage {
    playerId: string;
    x: number;
    y: number;
    timestamp: number;
    
    static create(playerId: string, x: number, y: number): PlayerMoveMessage {
        const msg = new PlayerMoveMessage();
        msg.playerId = playerId;
        msg.x = x;
        msg.y = y;
        msg.timestamp = Date.now();
        return msg;
    }
}

class ChatMessage {
    playerId: string;
    message: string;
    channel: string;
    timestamp: number;
    
    static create(playerId: string, message: string, channel: string = 'global'): ChatMessage {
        const msg = new ChatMessage();
        msg.playerId = playerId;
        msg.message = message;
        msg.channel = channel;
        msg.timestamp = Date.now();
        return msg;
    }
}
```

#### Message Handler Implementation

```typescript
@MsgHandler(LoginResponseMessage)
class LoginResponseHandler extends AMHandler<LoginResponseMessage> {
    async handle(session: Session, message: LoginResponseMessage): Promise<void> {
        if (message.success) {
            console.log(`Login successful. Player ID: ${message.playerId}`);
            
            // Store authentication token
            localStorage.setItem('authToken', message.token);
            localStorage.setItem('playerId', message.playerId);
            
            // Sync server time
            const serverTime = message.serverTime;
            const clientTime = Date.now();
            const timeDiff = serverTime - clientTime;
            TimeHelper.setServerTimeDifference(timeDiff);
            
            // Notify game of successful login
            EventSystem.get().publishAsync(
                this.domain,
                PlayerLoginSuccessEvent.create(message.playerId, message.token)
            );
            
            // Transition to game scene
            EventSystem.get().publishAsync(
                this.domain,
                SceneTransitionEvent.create('login', 'game')
            );
            
        } else {
            console.error('Login failed:', message.errorMessage);
            
            // Show error message to user
            EventSystem.get().publishAsync(
                this.domain,
                ShowErrorMessageEvent.create('Login Failed', message.errorMessage)
            );
        }
    }
}

@MsgHandler(PlayerMoveMessage)
class PlayerMoveHandler extends AMHandler<PlayerMoveMessage> {
    async handle(session: Session, message: PlayerMoveMessage): Promise<void> {
        // Find the player entity
        const playerEntity = this.getPlayerById(message.playerId);
        if (!playerEntity) {
            console.warn(`Player not found: ${message.playerId}`);
            return;
        }
        
        // Update player position
        const movementComponent = playerEntity.getComponent(MovementComponent);
        if (movementComponent) {
            movementComponent.setPosition(message.x, message.y);
            
            // Interpolate movement for smooth animation
            movementComponent.interpolateToPosition(message.x, message.y, message.timestamp);
        }
        
        // Update game state
        EventSystem.get().publish(
            this.domain,
            PlayerPositionUpdatedEvent.create(message.playerId, message.x, message.y)
        );
    }
    
    private getPlayerById(playerId: string): Entity | null {
        const gameScene = SceneMgr.get().getScene('Game');
        return gameScene?.getEntityById(playerId) || null;
    }
}

@MsgHandler(ChatMessage)
class ChatMessageHandler extends AMHandler<ChatMessage> {
    async handle(session: Session, message: ChatMessage): Promise<void> {
        // Add message to chat history
        const chatManager = Game.getSingleton(ChatManagerComponent);
        chatManager.addMessage(message.playerId, message.message, message.channel, message.timestamp);
        
        // Update chat UI
        EventSystem.get().publish(
            this.domain,
            ChatMessageReceivedEvent.create(message)
        );
        
        // Check for commands
        if (message.message.startsWith('/')) {
            await this.handleChatCommand(message);
        }
    }
    
    private async handleChatCommand(message: ChatMessage): Promise<void> {
        const parts = message.message.split(' ');
        const command = parts[0].substring(1); // Remove '/'
        
        switch (command) {
            case 'help':
                await this.sendHelpMessage(message.playerId);
                break;
            case 'time':
                await this.sendServerTime(message.playerId);
                break;
            case 'players':
                await this.sendPlayerList(message.playerId);
                break;
            default:
                console.log(`Unknown command: ${command}`);
        }
    }
}
```

#### Message Serialization

```typescript
import { MsgSerializeMgr } from 'moye-cocos';

class CustomMessageSerializer {
    static register() {
        const serializeMgr = MsgSerializeMgr.get();
        
        // Register serializers for different message types
        serializeMgr.registerSerializer(LoginMessage, {
            serialize: (msg: LoginMessage) => JSON.stringify(msg),
            deserialize: (data: string) => JSON.parse(data) as LoginMessage
        });
        
        serializeMgr.registerSerializer(PlayerMoveMessage, {
            serialize: (msg: PlayerMoveMessage) => {
                // Use binary encoding for frequent messages
                const buffer = new ArrayBuffer(20);
                const view = new DataView(buffer);
                view.setFloat32(0, msg.x);
                view.setFloat32(4, msg.y);
                view.setBigUint64(8, BigInt(msg.timestamp));
                return buffer;
            },
            deserialize: (data: ArrayBuffer) => {
                const view = new DataView(data);
                const msg = new PlayerMoveMessage();
                msg.x = view.getFloat32(0);
                msg.y = view.getFloat32(4);
                msg.timestamp = Number(view.getBigUint64(8));
                return msg;
            }
        });
    }
}
```

## Advanced Features

### Connection Management

```typescript
class ConnectionManager extends Singleton {
    private connections: Map<string, SessionCom> = new Map();
    private reconnectAttempts: Map<string, number> = new Map();
    private maxReconnectAttempts: number = 5;
    
    async connectToServer(serverId: string, url: string): Promise<SessionCom> {
        const session = Entity.create(Game.currentScene, SessionCom);
        
        try {
            await session.connect(url);
            this.connections.set(serverId, session);
            this.reconnectAttempts.set(serverId, 0);
            
            // Setup disconnect handler
            session.onDisconnect = () => this.handleDisconnect(serverId);
            
            return session;
        } catch (error) {
            console.error(`Failed to connect to ${serverId}:`, error);
            throw error;
        }
    }
    
    private async handleDisconnect(serverId: string) {
        const attempts = this.reconnectAttempts.get(serverId) || 0;
        
        if (attempts < this.maxReconnectAttempts) {
            console.log(`Attempting to reconnect to ${serverId} (${attempts + 1}/${this.maxReconnectAttempts})`);
            
            // Wait before reconnecting
            await TimerMgr.get().waitAsync(Math.pow(2, attempts) * 1000); // Exponential backoff
            
            try {
                const url = this.getServerUrl(serverId);
                await this.connectToServer(serverId, url);
                console.log(`Reconnected to ${serverId}`);
            } catch (error) {
                this.reconnectAttempts.set(serverId, attempts + 1);
                this.handleDisconnect(serverId); // Retry
            }
        } else {
            console.error(`Failed to reconnect to ${serverId} after ${this.maxReconnectAttempts} attempts`);
            
            // Notify game of connection failure
            EventSystem.get().publishAsync(
                Game.currentScene,
                ConnectionFailedEvent.create(serverId, 'max_reconnect_attempts_exceeded')
            );
        }
    }
    
    getConnection(serverId: string): SessionCom | null {
        return this.connections.get(serverId) || null;
    }
    
    disconnectFromServer(serverId: string) {
        const session = this.connections.get(serverId);
        if (session) {
            session.disconnect();
            this.connections.delete(serverId);
            this.reconnectAttempts.delete(serverId);
        }
    }
    
    disconnectAll() {
        for (const [serverId, session] of this.connections) {
            session.disconnect();
        }
        this.connections.clear();
        this.reconnectAttempts.clear();
    }
}
```

### Real-time Game Synchronization

```typescript
class GameSyncManager extends Entity {
    private gameSession: SessionCom;
    private lastSyncTime: number = 0;
    private syncInterval: number = 50; // 20 FPS
    private pendingUpdates: Map<string, any> = new Map();
    
    awake() {
        this.gameSession = this.getComponent(SessionCom);
        this.setupPeriodicSync();
    }
    
    private setupPeriodicSync() {
        TimerMgr.get().newRepeatedTimer(this.syncInterval, () => {
            this.sendPendingUpdates();
        });
    }
    
    queuePlayerUpdate(playerId: string, data: any) {
        this.pendingUpdates.set(playerId, {
            ...this.pendingUpdates.get(playerId),
            ...data,
            timestamp: Date.now()
        });
    }
    
    private sendPendingUpdates() {
        if (this.pendingUpdates.size === 0) return;
        
        const batchUpdate = {
            type: 'batch_update',
            updates: Array.from(this.pendingUpdates.entries()),
            timestamp: Date.now()
        };
        
        this.gameSession.send(batchUpdate);
        this.pendingUpdates.clear();
    }
    
    handlePlayerInput(inputType: string, data: any) {
        // Queue input for next sync
        const currentPlayer = Game.getCurrentPlayer();
        this.queuePlayerUpdate(currentPlayer.id, {
            input: {
                type: inputType,
                data: data,
                clientTime: Date.now()
            }
        });
        
        // For critical inputs, send immediately
        if (this.isCriticalInput(inputType)) {
            this.sendImmediateUpdate(currentPlayer.id, inputType, data);
        }
    }
    
    private isCriticalInput(inputType: string): boolean {
        return ['fire', 'use_item', 'cast_spell'].includes(inputType);
    }
    
    private sendImmediateUpdate(playerId: string, inputType: string, data: any) {
        this.gameSession.send({
            type: 'immediate_input',
            playerId,
            inputType,
            data,
            timestamp: Date.now()
        });
    }
}
```

### HTTP Interceptors

```typescript
class HttpInterceptorManager {
    private requestInterceptors: Array<(config: RequestConfig) => RequestConfig> = [];
    private responseInterceptors: Array<(response: any) => any> = [];
    
    addRequestInterceptor(interceptor: (config: RequestConfig) => RequestConfig) {
        this.requestInterceptors.push(interceptor);
    }
    
    addResponseInterceptor(interceptor: (response: any) => any) {
        this.responseInterceptors.push(interceptor);
    }
    
    processRequest(config: RequestConfig): RequestConfig {
        return this.requestInterceptors.reduce((conf, interceptor) => 
            interceptor(conf), config);
    }
    
    processResponse(response: any): any {
        return this.responseInterceptors.reduce((resp, interceptor) => 
            interceptor(resp), response);
    }
}

// Setup common interceptors
const interceptorMgr = new HttpInterceptorManager();

// Add auth token to all requests
interceptorMgr.addRequestInterceptor((config) => {
    const token = localStorage.getItem('authToken');
    if (token) {
        config.headers = config.headers || {};
        config.headers['Authorization'] = `Bearer ${token}`;
    }
    return config;
});

// Handle auth errors
interceptorMgr.addResponseInterceptor((response) => {
    if (response.status === 401) {
        // Token expired, redirect to login
        EventSystem.get().publishAsync(
            Game.currentScene,
            AuthenticationExpiredEvent.create()
        );
    }
    return response;
});

// Add request timing
interceptorMgr.addRequestInterceptor((config) => {
    config.startTime = Date.now();
    return config;
});

interceptorMgr.addResponseInterceptor((response) => {
    const duration = Date.now() - response.config.startTime;
    Logger.get().debug(`Request to ${response.config.url} took ${duration}ms`);
    return response;
});
```

## Best Practices

### 1. Error Handling

```typescript
class RobustNetworkManager extends Entity {
    async makeApiCall<T>(endpoint: string, data?: any): Promise<T> {
        const maxRetries = 3;
        let lastError: Error;
        
        for (let attempt = 1; attempt <= maxRetries; attempt++) {
            try {
                if (data) {
                    return await NetServices.get().httpPost<T>(endpoint, data);
                } else {
                    return await NetServices.get().httpGet<T>(endpoint);
                }
            } catch (error) {
                lastError = error;
                
                // Don't retry on client errors (4xx)
                if (error.status >= 400 && error.status < 500) {
                    throw error;
                }
                
                // Wait before retrying
                if (attempt < maxRetries) {
                    await TimerMgr.get().waitAsync(1000 * attempt);
                }
            }
        }
        
        throw lastError;
    }
}
```

### 2. Message Validation

```typescript
class MessageValidator {
    static validateLoginMessage(msg: LoginMessage): boolean {
        if (!msg.username || msg.username.length < 3) return false;
        if (!msg.password || msg.password.length < 6) return false;
        if (!msg.clientVersion) return false;
        return true;
    }
    
    static validatePlayerMove(msg: PlayerMoveMessage): boolean {
        if (!msg.playerId) return false;
        if (typeof msg.x !== 'number' || typeof msg.y !== 'number') return false;
        if (!msg.timestamp || msg.timestamp <= 0) return false;
        return true;
    }
}

@MsgHandler(LoginMessage)
class ValidatedLoginHandler extends AMHandler<LoginMessage> {
    async handle(session: Session, message: LoginMessage): Promise<void> {
        if (!MessageValidator.validateLoginMessage(message)) {
            session.send({
                type: 'error',
                message: 'Invalid login message format'
            });
            return;
        }
        
        // Process valid message
        await this.processLogin(session, message);
    }
}
```

### 3. Connection Pooling

```typescript
class ConnectionPool {
    private pools: Map<string, SessionCom[]> = new Map();
    private active: Map<string, SessionCom[]> = new Map();
    private maxPoolSize: number = 10;
    
    async getConnection(serverUrl: string): Promise<SessionCom> {
        const pool = this.pools.get(serverUrl) || [];
        
        if (pool.length > 0) {
            const session = pool.pop()!;
            this.addToActive(serverUrl, session);
            return session;
        }
        
        // Create new connection
        const session = Entity.create(Game.currentScene, SessionCom);
        await session.connect(serverUrl);
        this.addToActive(serverUrl, session);
        return session;
    }
    
    releaseConnection(serverUrl: string, session: SessionCom) {
        this.removeFromActive(serverUrl, session);
        
        const pool = this.pools.get(serverUrl) || [];
        if (pool.length < this.maxPoolSize && session.isConnected()) {
            pool.push(session);
            this.pools.set(serverUrl, pool);
        } else {
            session.disconnect();
        }
    }
    
    private addToActive(serverUrl: string, session: SessionCom) {
        const active = this.active.get(serverUrl) || [];
        active.push(session);
        this.active.set(serverUrl, active);
    }
    
    private removeFromActive(serverUrl: string, session: SessionCom) {
        const active = this.active.get(serverUrl) || [];
        const index = active.indexOf(session);
        if (index >= 0) {
            active.splice(index, 1);
        }
    }
}
```

### 4. Network Monitoring

```typescript
class NetworkMonitor extends Singleton {
    private metrics: {
        requestCount: number;
        responseTime: number[];
        errorCount: number;
        bytesReceived: number;
        bytesSent: number;
    } = {
        requestCount: 0,
        responseTime: [],
        errorCount: 0,
        bytesReceived: 0,
        bytesSent: 0
    };
    
    recordRequest(url: string, responseTime: number, success: boolean, bytes: number) {
        this.metrics.requestCount++;
        this.metrics.responseTime.push(responseTime);
        
        if (success) {
            this.metrics.bytesReceived += bytes;
        } else {
            this.metrics.errorCount++;
        }
        
        // Keep only last 100 response times
        if (this.metrics.responseTime.length > 100) {
            this.metrics.responseTime.shift();
        }
    }
    
    getAverageResponseTime(): number {
        const times = this.metrics.responseTime;
        return times.length > 0 ? times.reduce((a, b) => a + b) / times.length : 0;
    }
    
    getErrorRate(): number {
        return this.metrics.requestCount > 0 ? 
            this.metrics.errorCount / this.metrics.requestCount : 0;
    }
    
    getNetworkHealth(): 'good' | 'poor' | 'bad' {
        const avgResponseTime = this.getAverageResponseTime();
        const errorRate = this.getErrorRate();
        
        if (avgResponseTime < 100 && errorRate < 0.05) return 'good';
        if (avgResponseTime < 500 && errorRate < 0.15) return 'poor';
        return 'bad';
    }
}
```