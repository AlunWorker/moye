# UI Framework Documentation

## Overview

The Moye UI Framework provides a comprehensive system for managing user interfaces in Cocos Creator games. It extends the Entity system to provide view management, layering, lifecycle events, and component organization.

## Core Classes

### AMoyeView

The base class for all UI views in the Moye framework.

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

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `viewName` | `string` | Unique identifier for the view |
| `layer` | `ViewLayer` | UI layer for rendering order |
| `node` | `Node` | Cocos Creator node instance |

#### Lifecycle Methods

Views support several lifecycle methods that are called automatically:

```typescript
class MainMenuView extends AMoyeView {
    protected awake() {
        // Called when view is created, before node is available
        console.log("View awakened");
    }
    
    protected onLoad() {
        // Called when the Cocos Creator node is loaded
        this.setupUI();
        this.bindEvents();
    }
    
    protected onShow() {
        // Called when view becomes visible
        this.playShowAnimation();
        this.refreshData();
    }
    
    protected onHide() {
        // Called when view becomes hidden
        this.playHideAnimation();
        this.pauseUpdates();
    }
    
    protected destroy() {
        // Called when view is being disposed
        this.cleanup();
        this.unbindEvents();
    }
}
```

#### Methods

##### bringToFront(): void

Brings the view to the front of its layer.

```typescript
const menuView = MoyeViewMgr.get().getView("MainMenu");
menuView.bringToFront();
```

##### dispose(): void

Disposes the view and hides it from the UI manager.

```typescript
view.dispose(); // Will call hide() and cleanup
```

### ViewDecorator

Decorator for configuring view properties and registration.

```typescript
@ViewDecorator(config: IMoyeViewConfig)
```

#### IMoyeViewConfig

```typescript
interface IMoyeViewConfig {
    prefabPath: string;        // Path to the UI prefab
    layer: ViewLayer;          // UI layer
    isPopup?: boolean;         // Whether view is a popup
    isModal?: boolean;         // Whether view blocks interaction
    preload?: boolean;         // Whether to preload the view
    cacheNode?: boolean;       // Whether to cache the node
}
```

#### Usage

```typescript
@ViewDecorator({
    prefabPath: "ui/MainMenu",
    layer: ViewLayer.UI,
    isPopup: false,
    preload: true,
    cacheNode: true
})
class MainMenuView extends AMoyeView {
    protected onLoad() {
        // UI setup code
    }
}

@ViewDecorator({
    prefabPath: "ui/dialogs/ConfirmDialog",
    layer: ViewLayer.POPUP,
    isPopup: true,
    isModal: true
})
class ConfirmDialogView extends AMoyeView {
    private onConfirm: () => void;
    private onCancel: () => void;
    
    show(message: string, onConfirm: () => void, onCancel: () => void) {
        this.onConfirm = onConfirm;
        this.onCancel = onCancel;
        this.node.getChildByName("Message").getComponent(Label).string = message;
    }
    
    protected onLoad() {
        this.node.getChildByName("ConfirmBtn").on('click', () => {
            this.onConfirm?.();
            this.dispose();
        });
        
        this.node.getChildByName("CancelBtn").on('click', () => {
            this.onCancel?.();
            this.dispose();
        });
    }
}
```

### ViewLayer

Enum defining UI rendering layers.

```typescript
enum ViewLayer {
    BACKGROUND = 0,    // Background elements
    UI = 1,           // Main UI elements
    POPUP = 2,        // Popup dialogs
    OVERLAY = 3,      // Overlays and tooltips
    DEBUG = 4         // Debug UI
}
```

### MoyeViewMgr

Singleton manager for all UI views.

```typescript
class MoyeViewMgr extends Singleton {
    show<T extends AMoyeView>(viewType: new() => T, ...args: any[]): Promise<T>;
    hide(viewName: string): void;
    isShowing(viewName: string): boolean;
    getView<T extends AMoyeView>(viewName: string): T;
    hideAll(): void;
    hideAllPopups(): void;
}
```

#### Methods

##### show<T extends AMoyeView>(viewType: new() => T, ...args: any[]): Promise<T>

Shows a view and returns it when ready.

```typescript
// Show main menu
const mainMenu = await MoyeViewMgr.get().show(MainMenuView);

// Show dialog with parameters
const confirmDialog = await MoyeViewMgr.get().show(ConfirmDialogView, 
    "Are you sure?", 
    () => console.log("Confirmed"),
    () => console.log("Cancelled")
);
```

##### hide(viewName: string): void

Hides a view by name.

```typescript
MoyeViewMgr.get().hide("MainMenuView");
```

##### isShowing(viewName: string): boolean

Checks if a view is currently visible.

```typescript
if (MoyeViewMgr.get().isShowing("InventoryView")) {
    console.log("Inventory is open");
}
```

##### getView<T extends AMoyeView>(viewName: string): T

Gets a reference to an active view.

```typescript
const inventory = MoyeViewMgr.get().getView<InventoryView>("InventoryView");
if (inventory) {
    inventory.refreshItems();
}
```

##### hideAll(): void

Hides all active views.

```typescript
MoyeViewMgr.get().hideAll(); // Close everything
```

##### hideAllPopups(): void

Hides all popup views only.

```typescript
MoyeViewMgr.get().hideAllPopups(); // Close popups but keep main UI
```

## UI Components

### Common UI Patterns

#### Game HUD

```typescript
@ViewDecorator({
    prefabPath: "ui/GameHUD",
    layer: ViewLayer.UI,
    preload: true
})
class GameHUDView extends AMoyeView {
    private healthBar: ProgressBar;
    private manaBar: ProgressBar;
    private scoreLabel: Label;
    private minimap: Node;
    
    protected onLoad() {
        this.healthBar = this.node.getChildByPath("Stats/HealthBar").getComponent(ProgressBar);
        this.manaBar = this.node.getChildByPath("Stats/ManaBar").getComponent(ProgressBar);
        this.scoreLabel = this.node.getChildByPath("Score/Label").getComponent(Label);
        this.minimap = this.node.getChildByName("Minimap");
        
        this.setupEventListeners();
    }
    
    private setupEventListeners() {
        // Listen for player stat changes
        EventSystem.get().register(PlayerStatsChangedEvent, this.onPlayerStatsChanged.bind(this));
        EventSystem.get().register(PlayerScoreChangedEvent, this.onScoreChanged.bind(this));
    }
    
    private onPlayerStatsChanged(event: PlayerStatsChangedEvent) {
        this.updateHealthBar(event.health, event.maxHealth);
        this.updateManaBar(event.mana, event.maxMana);
    }
    
    private onScoreChanged(event: PlayerScoreChangedEvent) {
        this.scoreLabel.string = event.score.toString();
        this.playScoreAnimation();
    }
    
    updateHealthBar(current: number, max: number) {
        this.healthBar.progress = current / max;
        
        // Change color based on health percentage
        const percentage = current / max;
        if (percentage < 0.25) {
            this.healthBar.node.color = Color.RED;
        } else if (percentage < 0.5) {
            this.healthBar.node.color = Color.YELLOW;
        } else {
            this.healthBar.node.color = Color.GREEN;
        }
    }
    
    updateManaBar(current: number, max: number) {
        this.manaBar.progress = current / max;
    }
    
    private playScoreAnimation() {
        tween(this.scoreLabel.node)
            .to(0.1, { scale: new Vec3(1.2, 1.2, 1) })
            .to(0.1, { scale: new Vec3(1, 1, 1) })
            .start();
    }
}
```

#### Inventory System

```typescript
@ViewDecorator({
    prefabPath: "ui/InventoryView",
    layer: ViewLayer.UI,
    isPopup: true
})
class InventoryView extends AMoyeView {
    private itemContainer: Node;
    private itemPrefab: Prefab;
    private selectedItem: InventoryItemView | null = null;
    private items: InventoryItemView[] = [];
    
    protected onLoad() {
        this.itemContainer = this.node.getChildByPath("Content/ScrollView/View/Content");
        this.itemPrefab = resources.get("ui/InventoryItem", Prefab);
        
        this.setupControls();
        this.refreshItems();
    }
    
    private setupControls() {
        const closeBtn = this.node.getChildByName("CloseBtn");
        closeBtn.on('click', () => this.dispose());
        
        const sortBtn = this.node.getChildByName("SortBtn");
        sortBtn.on('click', () => this.sortItems());
        
        const useBtn = this.node.getChildByName("UseBtn");
        useBtn.on('click', () => this.useSelectedItem());
    }
    
    refreshItems() {
        // Clear existing items
        this.items.forEach(item => item.node.destroy());
        this.items = [];
        
        // Get inventory data
        const player = Game.currentPlayer;
        const inventory = player.getComponent(InventoryComponent);
        
        // Create item views
        for (const item of inventory.items) {
            this.createItemView(item);
        }
    }
    
    private createItemView(itemData: ItemData) {
        const itemNode = instantiate(this.itemPrefab);
        const itemView = itemNode.getComponent(InventoryItemView);
        
        itemView.setData(itemData);
        itemView.onSelected = (item) => this.selectItem(item);
        itemView.onDoubleClick = (item) => this.useItem(item);
        
        this.itemContainer.addChild(itemNode);
        this.items.push(itemView);
    }
    
    private selectItem(item: InventoryItemView) {
        // Deselect previous item
        if (this.selectedItem) {
            this.selectedItem.setSelected(false);
        }
        
        // Select new item
        this.selectedItem = item;
        item.setSelected(true);
        
        // Update item details
        this.updateItemDetails(item.itemData);
    }
    
    private updateItemDetails(itemData: ItemData) {
        const detailsPanel = this.node.getChildByName("ItemDetails");
        const nameLabel = detailsPanel.getChildByName("Name").getComponent(Label);
        const descLabel = detailsPanel.getChildByName("Description").getComponent(Label);
        const iconSprite = detailsPanel.getChildByName("Icon").getComponent(Sprite);
        
        nameLabel.string = itemData.name;
        descLabel.string = itemData.description;
        iconSprite.spriteFrame = itemData.icon;
    }
    
    private useSelectedItem() {
        if (!this.selectedItem) return;
        
        const player = Game.currentPlayer;
        const inventory = player.getComponent(InventoryComponent);
        
        if (inventory.useItem(this.selectedItem.itemData.id)) {
            EventSystem.get().publish(
                this.domain,
                ItemUsedEvent.create(player.id, this.selectedItem.itemData.id)
            );
            
            this.refreshItems();
        }
    }
    
    private sortItems() {
        this.items.sort((a, b) => {
            // Sort by type, then by name
            if (a.itemData.type !== b.itemData.type) {
                return a.itemData.type.localeCompare(b.itemData.type);
            }
            return a.itemData.name.localeCompare(b.itemData.name);
        });
        
        // Reorder nodes
        this.items.forEach((item, index) => {
            item.node.setSiblingIndex(index);
        });
    }
}

class InventoryItemView extends Component {
    itemData: ItemData;
    onSelected: (item: InventoryItemView) => void;
    onDoubleClick: (item: InventoryItemView) => void;
    
    private iconSprite: Sprite;
    private nameLabel: Label;
    private countLabel: Label;
    private background: Sprite;
    private clickCount: number = 0;
    
    onLoad() {
        this.iconSprite = this.node.getChildByName("Icon").getComponent(Sprite);
        this.nameLabel = this.node.getChildByName("Name").getComponent(Label);
        this.countLabel = this.node.getChildByName("Count").getComponent(Label);
        this.background = this.node.getComponent(Sprite);
        
        this.node.on('click', this.onClick, this);
    }
    
    setData(itemData: ItemData) {
        this.itemData = itemData;
        this.iconSprite.spriteFrame = itemData.icon;
        this.nameLabel.string = itemData.name;
        this.countLabel.string = itemData.count > 1 ? itemData.count.toString() : "";
    }
    
    setSelected(selected: boolean) {
        this.background.color = selected ? Color.YELLOW : Color.WHITE;
    }
    
    private onClick() {
        this.clickCount++;
        
        if (this.clickCount === 1) {
            TimerMgr.get().newOnceTimer(300, () => {
                if (this.clickCount === 1) {
                    this.onSelected?.(this);
                }
                this.clickCount = 0;
            });
        } else if (this.clickCount === 2) {
            this.onDoubleClick?.(this);
            this.clickCount = 0;
        }
    }
}
```

#### Modal Dialog System

```typescript
abstract class ModalDialogView extends AMoyeView {
    protected overlay: Node;
    protected dialog: Node;
    protected isAnimating: boolean = false;
    
    protected onLoad() {
        this.overlay = this.node.getChildByName("Overlay");
        this.dialog = this.node.getChildByName("Dialog");
        
        // Setup overlay click to close
        this.overlay.on('click', () => {
            if (!this.isModal) {
                this.dispose();
            }
        });
        
        // Prevent dialog clicks from closing
        this.dialog.on('click', (event: Event) => {
            event.stopPropagation();
        });
    }
    
    protected onShow() {
        this.playShowAnimation();
    }
    
    protected onHide() {
        this.playHideAnimation();
    }
    
    private async playShowAnimation() {
        if (this.isAnimating) return;
        this.isAnimating = true;
        
        // Start with invisible dialog
        this.dialog.scale = new Vec3(0.8, 0.8, 1);
        this.dialog.opacity = 0;
        this.overlay.opacity = 0;
        
        // Animate overlay fade in
        const overlayTween = tween(this.overlay)
            .to(0.2, { opacity: 180 });
        
        // Animate dialog scale and fade in
        const dialogTween = tween(this.dialog)
            .to(0.3, { 
                scale: new Vec3(1, 1, 1), 
                opacity: 255 
            }, { 
                easing: 'backOut' 
            });
        
        await Promise.all([
            new Promise(resolve => overlayTween.call(resolve).start()),
            new Promise(resolve => dialogTween.call(resolve).start())
        ]);
        
        this.isAnimating = false;
    }
    
    private async playHideAnimation() {
        if (this.isAnimating) return;
        this.isAnimating = true;
        
        // Animate dialog scale down and fade out
        const dialogTween = tween(this.dialog)
            .to(0.2, { 
                scale: new Vec3(0.8, 0.8, 1), 
                opacity: 0 
            }, { 
                easing: 'backIn' 
            });
        
        // Animate overlay fade out
        const overlayTween = tween(this.overlay)
            .to(0.2, { opacity: 0 });
        
        await Promise.all([
            new Promise(resolve => dialogTween.call(resolve).start()),
            new Promise(resolve => overlayTween.call(resolve).start())
        ]);
        
        this.isAnimating = false;
    }
}

@ViewDecorator({
    prefabPath: "ui/dialogs/MessageDialog",
    layer: ViewLayer.POPUP,
    isPopup: true,
    isModal: true
})
class MessageDialogView extends ModalDialogView {
    private titleLabel: Label;
    private messageLabel: Label;
    private okButton: Button;
    private onOk: () => void;
    
    protected onLoad() {
        super.onLoad();
        
        this.titleLabel = this.dialog.getChildByName("Title").getComponent(Label);
        this.messageLabel = this.dialog.getChildByName("Message").getComponent(Label);
        this.okButton = this.dialog.getChildByName("OkBtn").getComponent(Button);
        
        this.okButton.node.on('click', () => {
            this.onOk?.();
            this.dispose();
        });
    }
    
    show(title: string, message: string, onOk?: () => void) {
        this.titleLabel.string = title;
        this.messageLabel.string = message;
        this.onOk = onOk;
    }
}

// Usage
const messageDialog = await MoyeViewMgr.get().show(MessageDialogView);
messageDialog.show("Success", "Game saved successfully!", () => {
    console.log("User acknowledged save");
});
```

## UI Components Library

### Custom Components

#### MoyeLabel

Enhanced label component with additional features.

```typescript
import { MoyeLabel } from 'moye-cocos';

// Usage in view
class MyView extends AMoyeView {
    protected onLoad() {
        const label = this.node.getChildByName("Title").getComponent(MoyeLabel);
        label.typewriterEffect("Hello World!", 50); // Typewriter animation
        label.fadeIn(1.0); // Fade in over 1 second
    }
}
```

#### YYJJoystick

Virtual joystick for mobile controls.

```typescript
import { YYJJoystick } from 'moye-cocos';

class GameControlsView extends AMoyeView {
    private joystick: YYJJoystick;
    
    protected onLoad() {
        this.joystick = this.node.getChildByName("Joystick").getComponent(YYJJoystick);
        
        this.joystick.onMove = (direction: Vec2) => {
            // Handle joystick movement
            EventSystem.get().publish(
                this.domain,
                PlayerMoveEvent.create(direction.x, direction.y)
            );
        };
        
        this.joystick.onEnd = () => {
            // Handle joystick release
            EventSystem.get().publish(
                this.domain,
                PlayerStopEvent.create()
            );
        };
    }
}
```

#### RoundBoxSprite

Sprite with rounded corners.

```typescript
import { RoundBoxSprite } from 'moye-cocos';

class ModernUIView extends AMoyeView {
    protected onLoad() {
        const panel = this.node.getChildByName("Panel").getComponent(RoundBoxSprite);
        panel.setRadius(10); // 10px corner radius
        panel.setFillColor(Color.WHITE);
        panel.setBorderColor(Color.GRAY);
        panel.setBorderWidth(2);
    }
}
```

### UI Layout Components

#### CenterLayout

Automatically centers child elements.

```typescript
import { CenterLayout } from 'moye-cocos';

class CenteredMenuView extends AMoyeView {
    protected onLoad() {
        const menuContainer = this.node.getChildByName("MenuContainer");
        const centerLayout = menuContainer.addComponent(CenterLayout);
        
        centerLayout.horizontal = true;
        centerLayout.vertical = true;
        centerLayout.spacing = 20;
    }
}
```

#### BgAdapter

Background adapter for different screen sizes.

```typescript
import { BgAdapter } from 'moye-cocos';

class BackgroundView extends AMoyeView {
    protected onLoad() {
        const bg = this.node.getChildByName("Background");
        const adapter = bg.addComponent(BgAdapter);
        
        adapter.fitMode = BgAdapter.FitMode.COVER; // Cover entire screen
        adapter.alignMode = BgAdapter.AlignMode.CENTER; // Center alignment
    }
}
```

## Advanced UI Patterns

### View State Management

```typescript
enum ViewState {
    LOADING,
    READY,
    ERROR,
    UPDATING
}

abstract class StatefulView extends AMoyeView {
    protected currentState: ViewState = ViewState.LOADING;
    protected loadingSpinner: Node;
    protected contentContainer: Node;
    protected errorPanel: Node;
    
    protected onLoad() {
        this.loadingSpinner = this.node.getChildByName("LoadingSpinner");
        this.contentContainer = this.node.getChildByName("Content");
        this.errorPanel = this.node.getChildByName("ErrorPanel");
        
        this.updateStateDisplay();
    }
    
    protected setState(newState: ViewState) {
        if (this.currentState === newState) return;
        
        this.currentState = newState;
        this.updateStateDisplay();
        this.onStateChanged(newState);
    }
    
    protected updateStateDisplay() {
        this.loadingSpinner.active = this.currentState === ViewState.LOADING;
        this.contentContainer.active = this.currentState === ViewState.READY;
        this.errorPanel.active = this.currentState === ViewState.ERROR;
    }
    
    protected onStateChanged(state: ViewState) {
        // Override in subclasses
    }
}

class PlayerProfileView extends StatefulView {
    protected async onShow() {
        this.setState(ViewState.LOADING);
        
        try {
            const playerData = await this.loadPlayerData();
            this.displayPlayerData(playerData);
            this.setState(ViewState.READY);
        } catch (error) {
            this.displayError(error.message);
            this.setState(ViewState.ERROR);
        }
    }
    
    private async loadPlayerData(): Promise<PlayerData> {
        const netService = NetServices.get();
        return await netService.httpGet<PlayerData>('/api/player/profile');
    }
}
```

### Responsive UI

```typescript
class ResponsiveView extends AMoyeView {
    private screenSize: Size;
    private orientation: 'portrait' | 'landscape';
    
    protected onLoad() {
        this.updateLayout();
        
        // Listen for screen size changes
        view.on('canvas-resize', this.onScreenResize, this);
    }
    
    private onScreenResize() {
        this.updateLayout();
    }
    
    private updateLayout() {
        const canvas = find('Canvas');
        this.screenSize = canvas.getComponent(Canvas).designResolution;
        this.orientation = this.screenSize.width > this.screenSize.height ? 'landscape' : 'portrait';
        
        if (this.orientation === 'portrait') {
            this.setupPortraitLayout();
        } else {
            this.setupLandscapeLayout();
        }
    }
    
    private setupPortraitLayout() {
        // Vertical layout for portrait
        const layout = this.node.getComponent(Layout);
        layout.type = Layout.Type.VERTICAL;
        layout.spacingY = 20;
    }
    
    private setupLandscapeLayout() {
        // Horizontal layout for landscape
        const layout = this.node.getComponent(Layout);
        layout.type = Layout.Type.HORIZONTAL;
        layout.spacingX = 20;
    }
}
```

## Best Practices

### 1. View Organization

Structure views logically:

```typescript
// Good: Organized by feature
// ui/menus/MainMenuView.ts
// ui/menus/SettingsView.ts
// ui/game/GameHUDView.ts
// ui/game/InventoryView.ts
// ui/dialogs/ConfirmDialogView.ts

// Avoid: All views in one folder
// ui/MainMenuView.ts
// ui/SettingsView.ts
// ui/GameHUDView.ts
// ui/InventoryView.ts
// ui/ConfirmDialogView.ts
```

### 2. Data Binding

Use proper data binding patterns:

```typescript
class DataBoundView extends AMoyeView {
    private data: any;
    private bindings: Array<() => void> = [];
    
    bindData(data: any) {
        this.data = data;
        this.updateBindings();
    }
    
    private updateBindings() {
        this.bindings.forEach(binding => binding());
    }
    
    protected bindProperty(property: string, element: Node, component: string = 'Label', field: string = 'string') {
        const binding = () => {
            const comp = element.getComponent(component);
            if (comp && this.data) {
                comp[field] = this.data[property];
            }
        };
        
        this.bindings.push(binding);
        binding(); // Initial update
    }
    
    protected onLoad() {
        // Bind UI elements to data properties
        this.bindProperty('playerName', this.node.getChildByName('PlayerName'));
        this.bindProperty('level', this.node.getChildByName('Level'));
        this.bindProperty('experience', this.node.getChildByName('ExpBar'), 'ProgressBar', 'progress');
    }
}
```

### 3. Memory Management

Properly manage UI resources:

```typescript
class EfficientView extends AMoyeView {
    private textures: SpriteFrame[] = [];
    private animations: Animation[] = [];
    
    protected destroy() {
        // Clean up textures
        this.textures.forEach(texture => {
            if (texture && texture.isValid) {
                texture.destroy();
            }
        });
        this.textures = [];
        
        // Stop animations
        this.animations.forEach(anim => {
            if (anim && anim.isValid) {
                anim.stop();
            }
        });
        this.animations = [];
        
        super.destroy();
    }
    
    private loadTexture(path: string): Promise<SpriteFrame> {
        return new Promise((resolve, reject) => {
            resources.load(path, SpriteFrame, (err, spriteFrame) => {
                if (err) {
                    reject(err);
                } else {
                    this.textures.push(spriteFrame);
                    resolve(spriteFrame);
                }
            });
        });
    }
}
```

### 4. Performance Optimization

Optimize UI performance:

```typescript
class OptimizedListView extends AMoyeView {
    private itemPool: Node[] = [];
    private visibleItems: Node[] = [];
    private scrollView: ScrollView;
    private content: Node;
    
    protected onLoad() {
        this.scrollView = this.node.getComponent(ScrollView);
        this.content = this.scrollView.content;
        
        // Use object pooling for list items
        this.initializeItemPool();
        
        // Only update visible items
        this.scrollView.node.on('scrolling', this.onScrolling, this);
    }
    
    private initializeItemPool() {
        for (let i = 0; i < 20; i++) { // Pool size
            const item = instantiate(this.itemPrefab);
            item.active = false;
            this.itemPool.push(item);
        }
    }
    
    private onScrolling() {
        // Virtual scrolling - only render visible items
        this.updateVisibleItems();
    }
    
    private updateVisibleItems() {
        // Calculate which items should be visible
        const viewportHeight = this.scrollView.node.height;
        const scrollOffset = this.content.position.y;
        
        // Return unused items to pool
        this.visibleItems.forEach(item => {
            item.active = false;
            this.itemPool.push(item);
        });
        this.visibleItems = [];
        
        // Show items in viewport
        for (let i = this.getFirstVisibleIndex(); i <= this.getLastVisibleIndex(); i++) {
            const item = this.getPooledItem();
            this.setupItem(item, i);
            this.visibleItems.push(item);
        }
    }
}
```