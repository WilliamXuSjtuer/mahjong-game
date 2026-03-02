# 加杠后倒计时和超时 Bug 分析

## 问题描述
点击加杠后，倒计时没有结束，但是弹出超时，并且之后的轮次都无法再加杠。

## 根本原因分析

### 问题 1：加杠后没有清除出牌倒计时计时器

**位置**: `server.js` 第 1646-1711 行（`jia_gang` 处理逻辑）

**问题代码**:
```javascript
} else if (action.action === 'jia_gang') {
    // ... 加杠处理 ...
    
    // 杠后摸一张牌
    this.gameState.currentPlayerIndex = action.playerIndex;
    this.gameState.turnPhase = 'draw';  // 设置为 'draw' 阶段
    
    this.broadcastGameState();
    
    // 检查加杠后是否自摸
    setTimeout(() => {
        if (this.gameRunning) {
            const newTile = this.drawTileForPlayer(player, false);
            // ...
            if (player.isQiao && this.canHu(player.hand, player.melds)) {
                // 自摸逻辑
            } else {
                // 没有自摸，通知玩家出牌
                this.gameState.turnPhase = 'discard';
                this.notifyCurrentPlayer();  // 这里才调用 notifyCurrentPlayer
            }
        }
    }, 300);
}
```

**问题分析**:
1. 加杠后，`turnPhase` 被设置为 `'draw'`，然后广播了 `broadcastGameState()`
2. 但是**没有立即调用 `notifyCurrentPlayer()`**
3. 前端在收到 `game_state_update` 时，会检查 `turnPhase !== 'discard'`，从而调用 `stopDiscardCountdown()`
4. **然而**，服务器端的 `gameState.discardTimeout` 计时器**没有被清除**！
5. 300ms 后，服务器执行摸牌逻辑，如果没有自摸，才调用 `notifyCurrentPlayer()`
6. `notifyCurrentPlayer()` 会清除旧的 `discardTimeout` 并设置新的计时器
7. **但是**，如果前端在 300ms 内收到了 `game_state_update`，会停止倒计时显示
8. 然后 300ms 后服务器摸牌，再调用 `notifyCurrentPlayer()`，这时会重新设置倒计时
9. **关键问题**: 如果在加杠前，该玩家的 `discardTimeout` 已经设置，加杠时没有清除，会导致超时

### 问题 2：前端倒计时停止逻辑过于严格

**位置**: `index.html` 第 3929-3933 行

**问题代码**:
```javascript
// 【新增】如果不是我的出牌回合，停止倒计时
if (!isMyTurn || gameState.turnPhase !== 'discard') {
    stopDiscardCountdown('game_state_update: not my turn or not discard phase');
}
```

**问题分析**:
1. 前端只根据 `gameState.turnPhase` 来判断是否停止倒计时
2. 当加杠后 `turnPhase` 变为 `'draw'`，前端会停止倒计时
3. 但服务器的 `discardTimeout` 计时器可能还在运行
4. 这导致**前端显示停止，但服务器端计时器仍在运行**的不一致状态

### 问题 3：加杠后无法再次加杠的原因

**位置**: `server.js` 第 1080-1104 行（检查加杠逻辑）

**问题代码**:
```javascript
// 检查加杠（摸到的牌可以与副露中的刻子组成杠）
const jiaGangActions = [];
for (const meld of player.melds) {
    if (meld.type === 'peng' && meld.tiles && meld.tiles.length > 0) {
        const pengTile = meld.tiles[0];
        if (pengTile.type === tile.type && pengTile.value === tile.value) {
            jiaGangActions.push({
                meldIndex: i,
                tile: tile
            });
        }
    }
}

// 如果有加杠选项且没有自摸，优先提示加杠
if (jiaGangActions.length > 0 && !this.gameState.pendingZimo) {
    if (player.socket) {
        player.socket.emit('action_available', {
            playerId: player.id,
            actions: ['jia_gang'],
            tile: tile,
            jiaGangOptions: jiaGangActions
        });
    }
}
```

**问题分析**:
1. 加杠后，刻子（`peng`）会升级为杠（`gang`）
2. 下次摸到相同的牌时，由于 `meld.type` 已经是 `'gang'`，不再是 `'peng'`
3. 所以不会再检测到加杠选项
4. **这是正常的游戏逻辑**，但如果是因为超时导致加杠失败，则会影响后续轮次

## 解决方案

### 方案 1：加杠后立即清除倒计时并重新通知

修改 `server.js` 中 `jia_gang` 的处理逻辑：

```javascript
} else if (action.action === 'jia_gang') {
    // ... 加杠处理 ...
    
    const meld = player.melds[meldIndex];
    meld.type = 'gang';
    meld.from = player.seatIndex;
    meld.tiles.push(tile);
    
    this.broadcast('action_executed', {
        playerIndex: action.playerIndex,
        action: 'jia_gang',
        tile: tile,
        tileName: getTileName(tile)
    });
    
    // 杠后摸一张牌
    this.gameState.currentPlayerIndex = action.playerIndex;
    this.gameState.turnPhase = 'draw';
    
    // 【修复】清除之前的出牌超时计时器
    if (this.gameState.discardTimeout) {
        clearTimeout(this.gameState.discardTimeout);
        this.gameState.discardTimeout = null;
    }
    
    this.broadcastGameState();
    
    // 检查加杠后是否自摸
    setTimeout(() => {
        if (this.gameRunning) {
            const newTile = this.drawTileForPlayer(player, false);
            if (!newTile) {
                this.endRound('draw', -1, -1, false, false);
                return;
            }
            
            this.broadcastGameState();
            
            if (player.isQiao && this.canHu(player.hand, player.melds)) {
                this.gameState.pendingZimo = {
                    playerId: player.id,
                    playerIndex: player.seatIndex,
                    tile: newTile
                };
                
                if (player.socket) {
                    player.socket.emit('action_available', {
                        playerId: player.id,
                        actions: ['hu_zimo'],
                        tile: newTile
                    });
                }
            } else {
                // 没有自摸，通知玩家出牌
                this.gameState.turnPhase = 'discard';
                this.notifyCurrentPlayer();
            }
        }
    }, 300);
}
```

### 方案 2：前端收到 action_executed 时也停止倒计时

修改 `index.html` 中 `action_executed` 的监听逻辑：

```javascript
socket.on('action_executed', (data) => {
    const player = gameState.players.find(p => p.seatIndex === data.playerIndex);
    const playerName = player?.username || '玩家';
    const playerVoice = player?.voice || getPlayerVoiceBySeat(data.playerIndex);
    const isMe = data.playerIndex === mySeatIndex;
    
    // 【修复】如果是加杠动作，立即停止倒计时
    if (data.action === 'jia_gang' && isMe) {
        stopDiscardCountdown('action_executed: jia_gang');
    }
    
    // 显示动作特效（碰/杠/胡/加杠/吃 大字动画）
    if (data.action === 'peng' || data.action === 'gang' || data.action === 'hu' || data.action === 'hu_zimo' || data.action === 'jia_gang' || data.action === 'chi') {
        showActionEffect(data.action, playerName);
    }
    
    // ... 其他逻辑 ...
});
```

### 方案 3：服务器端统一在 action_executed 后清除倒计时

在 `resolveActions` 函数的开头，清除所有玩家的倒计时：

```javascript
resolveActions() {
    // 通知所有玩家隐藏动作按钮
    this.broadcast('action_timeout', {});
    
    // 【修复】清除出牌超时计时器
    if (this.gameState.discardTimeout) {
        clearTimeout(this.gameState.discardTimeout);
        this.gameState.discardTimeout = null;
    }
    
    // 【修复】通知前端停止倒计时
    this.broadcast('stop_countdown', {});
    
    // ... 其他逻辑 ...
}
```

## 推荐修复方案

**采用方案 1 + 方案 2 的组合**：

1. 服务器端在加杠处理时立即清除 `discardTimeout`
2. 前端在收到 `action_executed` 且动作为 `jia_gang` 时立即停止倒计时显示
3. 这样可以确保服务器和前端的状态完全同步

## 已实施的修复

### 修复 1: server.js 第 1674-1681 行

在加杠后、广播游戏状态前，立即清除出牌超时计时器：

```javascript
// 杠后摸一张牌
this.gameState.currentPlayerIndex = action.playerIndex;
this.gameState.turnPhase = 'draw';

// 【修复】清除之前的出牌超时计时器，避免加杠后超时
if (this.gameState.discardTimeout) {
    clearTimeout(this.gameState.discardTimeout);
    this.gameState.discardTimeout = null;
}

this.broadcastGameState();
```

### 修复 2: index.html 第 4051-4056 行

在收到加杠动作时，立即停止倒计时显示：

```javascript
// 【修复】如果是加杠动作，立即停止倒计时，避免超时
if (data.action === 'jia_gang' && isMe) {
    stopDiscardCountdown('action_executed: jia_gang');
}
```

## 验证步骤

1. **正常加杠流程**:
   - 玩家摸牌后点击加杠
   - 倒计时应立即消失
   - 服务器不应触发超时
   - 玩家正常摸牌并出牌

2. **连续加杠测试**:
   - 第一轮加杠后正常出牌
   - 后续轮次再次摸到可加杠的牌
   - 应能正常加杠，不受之前影响

3. **边界情况**:
   - 加杠后自摸
   - 加杠后流局
   - 接近超时时加杠

## 额外建议

1. **增加日志**: 在加杠处理的关键步骤增加 `console.log`，方便调试
2. **统一状态管理**: 确保服务器和前端的状态变更顺序一致
3. **测试场景**:
   - 正常加杠后摸牌
   - 加杠后自摸
   - 加杠后流局
   - 连续多轮加杠
   - 加杠时接近超时

## 总结

这个 Bug 的核心原因是**服务器端和前端的状态不同步**：
- 服务器端保留了旧的 `discardTimeout` 计时器
- 前端在收到 `game_state_update` 后停止了倒计时显示
- 导致用户看到倒计时停止，但服务器端可能触发超时

修复的关键是**在加杠处理时立即清除所有倒计时相关的状态**，并确保前端同步更新。
