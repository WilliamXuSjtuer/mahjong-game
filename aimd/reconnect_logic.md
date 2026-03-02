# 上海麻将 - 离线重连与数据恢复逻辑

## 一、概述

本文档说明玩家离线后重连时，如何恢复数据并回到牌桌UI界面的完整逻辑。

---

## 二、现有重连机制架构

### 2.1 技术栈

| 层级 | 技术 |
|-----|------|
| 通信协议 | Socket.IO (WebSocket + HTTP Long Polling) |
| 心跳配置 | pingTimeout: 30s, pingInterval: 10s |
| 重连策略 | 最多5次重连，延迟1-5秒指数递增 |

### 2.2 关键数据存储

- **服务端**: `gameRooms` (Map) 存储房间对象，`playerSockets` (Map) 存储SocketID与房间的映射
- **客户端**: `localStorage` 存储 `roomCode` 和 `username`

---

## 三、重连流程时序图

```
┌─────────┐                           ┌─────────┐                         ┌─────────┐
│  客户端  │                           │  服务器  │                         │  牌桌UI  │
└────┬────┘                           └────┬────┘                         └────┬────┘
     │                                     │                                   │
     │  1. Socket断开                       │                                   │
     │───────────────────────────────────> │                                   │
     │                                     │  标记offline=true                  │
     │                                     │  广播player_offline               │
     │<─────────────────────────────────── │                                   │
     │                                     │                                   │
     │  2. 自动重连尝试 (最多5次)            │                                   │
     │─────┐                                │                                   │
     │     │ reconnectionDelay              │                                   │
     │<────┘                                │                                   │
     │                                     │                                   │
     │  3. emit('join_room')               │                                   │
     │───────────────────────────────────> │                                   │
     │                                     │  验证离线玩家身份                  │
     │                                     │  恢复player状态                    │
     │                                     │  清除autoDissolveTimer            │
     │                                     │  广播player_reconnected          │
     │<─────────────────────────────────── │                                   │
     │                                     │                                   │
     │  4. emit('game_started')            │                                   │
     │     + isReconnect=true              │                                   │
     │<─────────────────────────────────── │                                   │
     │                                     │                                   │
     │  5. 恢复游戏状态到UI                 │                                   │
     │───────────────────────────────────> │                                   │
     │                                     │  显示完整牌桌界面                  │
     │                                     │                                   │
```

---

## 四、客户端重连逻辑

### 4.1 Socket初始化与重连配置

**文件**: `index.html` 第3650-3660行

```javascript
socket = io(serverUrl, {
    reconnectionAttempts: 5,      // 最多重连5次
    reconnectionDelay: 1000,      // 初始重连延迟1秒
    reconnectionDelayMax: 5000,   // 最大延迟5秒
    transports: ['websocket', 'polling'],
});
```

### 4.2 断线检测与自动重连

**文件**: `index.html` 第3669-3685行

```javascript
// 连接成功后，检查是否有保存的房间信息，尝试重连
socket.on('connect', () => {
    myPlayerId = socket.id;
    const savedRoomCode = localStorage.getItem('roomCode');
    const savedUsername = localStorage.getItem('username');
    
    if (savedRoomCode && savedUsername) {
        console.log('尝试自动重连到房间:', savedRoomCode);
        socket.emit('join_room', { 
            roomCode: savedRoomCode, 
            username: savedUsername,
            avatar: '👤',
            voice: myVoice
        });
    }
});
```

### 4.3 接收重连事件

**文件**: `index.html` 第3720-3726行

```javascript
// 玩家离线提示
socket.on('player_offline', (data) => {
    showToast(`⚠️ ${data.username} 断线了，等待重连...`, 5000);
});

// 玩家重连提示
socket.on('player_reconnected', (data) => {
    showToast(`✅ ${data.username} 已重连！`);
});
```

---

## 五、服务端重连处理逻辑

### 5.1 断线处理

**文件**: `server.js` 第3054-3061行

```javascript
socket.on('disconnect', () => {
    console.log('断开连接:', socket.id);
    const room = playerSockets.get(socket.id);
    if (room) {
        room.removePlayer(socket.id);
    }
});
```

### 5.2 离线玩家标记

**文件**: `server.js` 第400-474行 (`removePlayer`方法)

```javascript
// 游戏正在进行中，只标记离线，不真正移除
if (this.gameRunning && !player.isBot) {
    player.offline = true;
    player.offlineTime = Date.now();
    player.socket = null;
    
    // 广播离线状态
    this.broadcast('player_offline', { 
        username: player.username, 
        seatIndex: player.seatIndex 
    });
    
    // 如果轮到离线玩家，AI立即接管
    if (this.gameState.currentPlayerIndex === player.seatIndex) {
        this.aiAction(player);
    }
    
    // 所有真人玩家离线，30秒后自动解散
    if (onlineRealPlayers.length === 0) {
        this.autoDissolveTimer = setTimeout(() => {
            this.endGameForDissolve();
        }, 30000);
    }
    return;
}
```

### 5.3 重连验证与状态恢复

**文件**: `server.js` 第238-321行

```javascript
// 检查是否是重连（相同用户名的离线玩家）
const offlinePlayer = this.players.find(p => !p.isBot && p.offline && p.username === username);

if (offlinePlayer) {
    // 1. 恢复玩家状态
    offlinePlayer.id = socket.id;
    offlinePlayer.socket = socket;
    offlinePlayer.offline = false;
    offlinePlayer.offlineTime = null;
    
    // 2. 取消自动解散计时器
    if (this.autoDissolveTimer) {
        clearTimeout(this.autoDissolveTimer);
    }
    
    // 3. 广播重连
    this.broadcast('player_reconnected', { 
        username: username, 
        seatIndex: offlinePlayer.seatIndex 
    });
    
    // 4. 如果游戏正在进行，发送当前状态并恢复控制权
    if (this.gameRunning) {
        socket.emit('game_started', {
            gameState: this.getPlayerGameState(socket.id),
            isReconnect: true
        });
        
        // 如果正好轮到他，恢复控制权
        if (this.gameState.currentPlayerIndex === offlinePlayer.seatIndex) {
            this.setDiscardTimeout(offlinePlayer);
            socket.emit('your_turn', { phase: 'discard' });
        }
    }
}
```

---

## 六、数据恢复逻辑

### 6.1 需要恢复的核心数据

| 数据类型 | 说明 |
|---------|------|
| 玩家手牌 | 离线玩家当前持有的牌 |
| 副露区 | 碰、杠、吃的数据 |
| 弃牌区 | 玩家打出的牌 |
| 花牌 | 玩家摸到的花牌 |
| 听牌状态 | 是否已听牌 |
| 敲牌状态 | 是否已敲牌 |
| 座位信息 | 玩家座位和风向 |
| 分数 | 当前累计分数 |

### 6.2 服务端数据获取

**文件**: `server.js` (`getPlayerGameState`方法)

```javascript
getPlayerGameState(socketId) {
    const player = this.players.find(p => p.id === socketId);
    if (!player) return null;
    
    // 只返回该玩家可见的数据
    const visiblePlayers = this.players.map(p => ({
        ...p,
        // 对手牌进行隐藏处理
        hand: p.seatIndex === player.seatIndex ? p.hand : 
              (p.offline ? p.hand : this.getMaskedHand(p.hand)),
        socket: undefined  // 不发送socket对象
    }));
    
    return {
        players: visiblePlayers,
        gameState: this.gameState,
        mySeatIndex: player.seatIndex,
        gameRunning: this.gameRunning
    };
}
```

### 6.3 客户端接收并恢复数据

```javascript
socket.on('game_started', (data) => {
    if (data.isReconnect) {
        // 恢复游戏状态
        gameState = data.gameState;
        mySeatIndex = data.mySeatIndex;
        isReconnecting = true;
        
        // 更新UI显示
        updateGameUI();
        
        // 显示重连成功提示
        showToast('重连成功！游戏继续...', 3000);
        
        // 恢复房间信息
        showRoom(gameRoomCode);
    } else {
        // 正常开始新游戏
        startNewGame(data);
    }
});
```

---

## 七、牌桌UI恢复逻辑

### 7.1 UI更新主函数

**文件**: `index.html` 第4466-4638行

```javascript
function updateGameUI() {
    if (!gameState) return;
    scheduleUpdate(() => {
        _doUpdateGameUI();
    });
}

function _doUpdateGameUI() {
    const myPlayer = gameState.players.find(p => p.seatIndex === mySeatIndex);
    
    // 1. 更新自己的手牌（可选择、可操作）
    playerHandEl.innerHTML = myPlayer.hand.map((tile, index) => {
        return renderTile(tile, { selectable: true, index });
    }).join('');
    
    // 2. 更新花牌显示
    playerFlowersEl.innerHTML = myPlayer.flowers.map(tile => 
        renderTile(tile, { small: true })
    ).join('');
    
    // 3. 更新副露区
    playerMeldsEl.innerHTML = renderMelds(myPlayer.melds);
    
    // 4. 更新弃牌区
    updateDiscards();
    
    // 5. 更新所有玩家状态（包括离线状态）
    updateAllPlayerInfo();
    
    // 6. 更新当前回合和阶段
    updateTurnIndicator();
    
    // 7. 更新操作按钮（碰/杠/胡/过）
    updateActionButtons();
}
```

### 7.2 座位映射逻辑

**文件**: `index.html` 第4639-4642行

```javascript
// 获取显示座位（相对于自己的位置）
// 0=自己, 1=右家, 2=对家, 3=左家
function getDisplaySeat(seatIndex) {
    return (seatIndex - mySeatIndex + 4) % 4;
}
```

### 7.3 离线状态显示

```javascript
// 更新离线状态样式
avatarEl.classList.toggle('offline', player.offline === true);
```

**CSS样式** (第442行):
```css
.player-avatar.offline {
    opacity: 0.5;
    /* 其他样式 */
}
```

### 7.4 恢复控制权

如果重连后正好轮到该玩家出牌：

```javascript
socket.on('your_turn', (data) => {
    // 显示操作提示
    showToast('轮到你出牌了！', 2000);
    
    // 启用手牌选择
    enableHandSelection();
    
    // 如果有听牌提示，显示听牌牌型
    if (myPlayer.isTing) {
        showTing提示(myPlayer.tingTiles);
    }
});
```

---

## 八、重连完整流程总结

```
┌─────────────────────────────────────────────────────────────┐
│                      玩家重连流程                            │
├─────────────────────────────────────────────────────────────┤
│  步骤1: 断线检测                                              │
│    - Socket.IO 自动检测连接断开                              │
│    - 触发 disconnect 事件                                    │
│                                                             │
│  步骤2: 服务端离线处理                                       │
│    - 标记 player.offline = true                              │
│    - 广播 'player_offline' 事件                              │
│    - 若轮到离线玩家，AI立即接管                               │
│    - 若所有真人离线，启动30秒解散计时器                       │
│                                                             │
│  步骤3: 客户端自动重连                                       │
│    - Socket.IO 自动重连（最多5次）                           │
│    - 连接成功后检查 localStorage                            │
│    - 发送 join_room 请求                                     │
│                                                             │
│  步骤4: 服务端验证重连                                       │
│    - 通过 username 查找离线玩家                               │
│    - 恢复 player 状态（offline=false）                      │
│    - 清除解散计时器                                          │
│    - 广播 'player_reconnected' 事件                          │
│                                                             │
│  步骤5: 数据恢复                                             │
│    - 发送完整 gameState                                      │
│    - 设置 isReconnect = true                                │
│    - 恢复座位信息和游戏阶段                                   │
│                                                             │
│  步骤6: UI恢复                                               │
│    - updateGameUI() 渲染完整牌桌                            │
│    - 显示手牌、副露、花牌、弃牌                              │
│    - 显示所有玩家状态（在线/离线）                            │
│    - 若轮到该玩家，显示操作按钮                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 九、关键文件位置索引

| 功能 | 文件位置 |
|-----|---------|
| Socket初始化 | `index.html` 第3650-3660行 |
| 自动重连逻辑 | `index.html` 第3669-3685行 |
| 断线事件处理 | `server.js` 第3054-3061行 |
| 离线玩家移除 | `server.js` 第400-474行 |
| 重连验证恢复 | `server.js` 第238-321行 |
| 游戏状态获取 | `server.js` getPlayerGameState方法 |
| UI更新函数 | `index.html` 第4466-4638行 |
| 座位映射 | `index.html` 第4639-4642行 |
| 离线样式 | `index.html` 第442行, 第4656行 |

---

## 十、待优化建议

1. **数据增量同步**: 当前重连时发送完整游戏状态，可考虑改为增量同步减少数据量
2. **断线补偿机制**: 考虑为长时间断线的玩家提供积分补偿
3. **重连动画**: 添加重连成功的动画效果提升用户体验
4. **后台数据持久化**: 考虑将游戏状态持久化到数据库，防止服务器重启导致数据丢失
