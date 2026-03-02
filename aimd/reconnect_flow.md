# 断线玩家刷新页面后重连响应流程

## 一、概述

本文档详细说明断线玩家刷新页面后，在开始页面输入用户名和房间号的重连响应完整流程。

---

## 二、完整流程时序图

```
┌─────────────────────┐                              ┌─────────────────────┐                              ┌─────────────────────┐
│     开始页面        │                              │      客户端         │                              │      服务端         │
│   (index.html)      │                              │    (socket.io)      │                              │    (server.js)      │
└─────────┬───────────┘                              └──────────┬──────────┘                              └──────────┬──────────┘
          │                                                   │                                                  │
          │  1. 用户输入 username + roomCode                 │                                                  │
          │  点击"加入房间"                                    │                                                  │
          │────────────────────────────────────────────────> │                                                  │
          │                                                   │                                                  │
          │                                                   │  2. emit('join_room')                          │
          │                                                   │  { roomCode, username, avatar, voice }          │
          │                                                   │────────────────────────────────────────────────> │
          │                                                   │                                                  │
          │                                                   │                                                  │  3. socket.on('join_room')
          │                                                   │                                                  │  获取 roomCode, username
          │                                                   │                                                  │  const room = gameRooms.get(code)
          │                                                   │                                                  │                                                  │
          │                                                   │                                                  │  4. 检查房间是否存在
          │                                                   │                                                  │  if (!room) {
          │                                                   │                                                  │      emit('join_error')
          │                                                   │                                                  │      return;
          │                                                   │                                                  │  }
          │                                                   │                                                  │                                                  │
          │                                                   │                                                  │  5. 检查游戏是否正在进行
          │                                                   │                                                  │  if (room.gameRunning) {
          │                                                   │                                                  │      // 关键：检查是否是断线玩家重连
          │                                                   │                                                  │      const offlinePlayer = room.players.find(
          │                                                   │                                                  │          p => !p.isBot && p.offline && p.username === username
          │                                                   │                                                  │      );
          │                                                   │                                                  │      if (offlinePlayer) {
          │                                                   │                                                  │          // 允许重连
          │                                                   │                                                  │          room.addPlayer(socket, username, avatar, voice);
          │                                                   │                                                  │          return;
          │                                                   │                                                  │      }
          │                                                   │                                                  │  }
          │                                                   │                                                  │                                                  │
          │                                                   │                                                  │  6. 调用 room.addPlayer() 方法
          │                                                   │                                                  │  addPlayer(socket, username, avatar, voice) {
          │                                                   │                                                  │                                                  │
          │                                                   │                                                  │  7. 检查是否是重连
          │                                                   │                                                  │  const offlinePlayer = this.players.find(
          │                                                   │                                                  │      p => !p.isBot && p.offline && p.username === username
          │                                                   │                                                  │  );
          │                                                   │                                                  │                                                  │
          │                                                   │                                                  │  8. 恢复离线玩家状态
          │                                                   │                                                  │  offlinePlayer.id = socket.id;
          │                                                   │                                                  │  offlinePlayer.socket = socket;
          │                                                   │                                                  │  offlinePlayer.offline = false;
          │                                                   │                                                  │  offlinePlayer.offlineTime = null;
          │                                                   │                                                  │                                                  │
          │                                                   │                                                  │  9. 取消自动解散计时器
          │                                                   │                                                  │  if (this.autoDissolveTimer) {
          │                                                   │                                                  │      clearTimeout(this.autoDissolveTimer);
          │                                                   │                                                  │      emit('room_auto_dissolve_cancelled')
          │                                                   │                                                  │  }
          │                                                   │                                                  │                                                  │
          │                                                   │  10. broadcast('player_reconnected')            │                                                  │
          │                                                   │<───────────────────────────────────────────────── │                                                  │
          │                                                   │                                                  │                                                  │
          │                                                   │  11. broadcast('room_updated')                  │                                                  │
          │                                                   │<───────────────────────────────────────────────── │                                                  │
          │                                                   │                                                  │                                                  │
          │                                                   │                                                  │  12. 如果游戏正在进行
          │                                                   │                                                  │  if (this.gameRunning) {
          │                                                   │                                                  │      // 发送完整游戏状态
          │                                                   │  13. emit('game_started')                      │                                                  │
          │                                                   │  {                                              │                                                  │
          │                                                   │      gameState: getPlayerGameState(socket.id),  │                                                  │
          │                                                   │      dealerIndex,                               │                                                  │
          │                                                   │      yourSeat: offlinePlayer.seatIndex,        │                                                  │
          │                                                   │      currentRound,                             │                                                  │
          │                                                   │      totalRounds,                              │                                                  │
          │                                                   │      matchScores,                              │                                                  │
          │                                                   │      isReconnect: true                         │                                                  │
          │                                                   │  }                                             │                                                  │
          │                                                   │<───────────────────────────────────────────────── │                                                  │
          │                                                   │                                                  │                                                  │
          │                                                   │                                                  │  14. 如果轮到该玩家出牌
          │                                                   │                                                  │  if (currentPlayerIndex === offlinePlayer.seatIndex) {
          │                                                   │                                                  │      setDiscardTimeout(offlinePlayer);
          │                                                   │                                                  │      // 延迟发送确保socket稳定
          │                                                   │  15. emit('your_turn')                         │                                                  │
          │                                                   │  { phase: 'discard', message: '...' }         │                                                  │
          │                                                   │<───────────────────────────────────────────────── │                                                  │
          │                                                   │                                                  │                                                  │
          │                                                   │                                                  │  16. 检查待处理动作
          │                                                   │                                                  │  if (pendingActions.find(a => a.playerId === socket.id)) {
          │                                                   │  17. emit('action_available')                  │                                                  │
          │                                                   │  { actions, tile }                             │                                                  │
          │                                                   │<───────────────────────────────────────────────── │                                                  │
          │                                                   │                                                  │                                                  │
          │                                                   │                                                  │  }
          │                                                   │                                                  │
          │                                                   │  18. socket.on('room_joined')                   │                                                  │
          │                                                   │  currentRoom = data.roomCode;                   │                                                  │
          │                                                   │  sessionStorage.setItem(...);                   │                                                  │
          │                                                   │  showRoomScreen();                              │                                                  │
          │                                                   │────────────────────────────────────────────────> │                                                  │
          │                                                   │                                                  │
          │                                                   │  19. socket.on('player_reconnected')            │                                                  │
          │                                                   │  showToast('✅ xxx 已重连！');                   │                                                  │
          │                                                   │                                                  │
          │                                                   │  20. socket.on('game_started')                 │                                                  │
          │                                                   │  mySeatIndex = data.yourSeat;                   │                                                  │
          │                                                   │  gameState = data.gameState;                   │                                                  │
          │                                                   │  gameRunning = true;                           │                                                  │
          │                                                   │                                                  │
          │                                                   │  21. 检查 isReconnect 标志                      │                                                  │
          │                                                   │  if (data.isReconnect) {                        │                                                  │
          │                                                   │      showToast('🔄 已重新连接！继续游戏...');    │                                                  │
          │                                                   │      // 如果轮到自己                            │                                                  │
          │                                                   │      if (currentPlayerIndex === mySeatIndex) {  │                                                  │
          │                                                   │          if (turnPhase === 'draw') {            │                                                  │
          │                                                   │              autoDrawTile();                     │                                                  │
          │                                                   │          }                                      │                                                  │
          │                                                   │      }                                          │                                                  │
          │                                                   │  }                                              │                                                  │
          │                                                   │                                                  │
          │                                                   │  22. showGameScreen();                          │                                                  │
          │                                                   │  updateGameUI();                                │                                                  │
          │────────────────────────────────────────────────> │                                                  │
          │                                                   │                                                  │
```

---

## 三、客户端发起阶段

### 3.1 用户输入并点击加入房间

**文件**: `index.html` 第4331行

```javascript
socket.emit('join_room', { 
    roomCode, 
    username, 
    avatar: '👤', 
    voice: myVoice 
});
```

### 3.2 参数说明

| 参数 | 说明 |
|-----|------|
| `roomCode` | 房间号（6位大写字母） |
| `username` | 用户名（必须与断线前一致才能重连） |
| `avatar` | 头像标识 |
| `voice` | 语音类型（female01/female02/male/male02） |

---

## 四、服务端处理阶段

### 4.1 接收 join_room 事件

**文件**: `server.js` 第2796-2842行

```javascript
socket.on('join_room', (data) => {
    const { roomCode, username, avatar, voice } = data;
    const code = roomCode.toUpperCase().trim();
    const room = gameRooms.get(code);
    
    // 检查房间是否存在
    if (!room) {
        socket.emit('join_error', { message: `房间 ${code} 不存在` });
        return;
    }
    
    // 游戏已进行，检查是否是断线玩家重连
    if (room.gameRunning) {
        const offlinePlayer = room.players.find(
            p => !p.isBot && p.offline && p.username === username
        );
        if (offlinePlayer) {
            room.addPlayer(socket, username, avatar, voice || 'female01');
            return;
        }
        socket.emit('join_error', { message: '游戏已开始，无法加入' });
        return;
    }
    
    // 正常加入房间逻辑...
});
```

### 4.2 addPlayer 方法处理重连

**文件**: `server.js` 第233-321行

#### 步骤1: 检查重连玩家

```javascript
const offlinePlayer = this.players.find(
    p => !p.isBot && p.offline && p.username === username
);
```

#### 步骤2: 恢复玩家状态

```javascript
if (offlinePlayer) {
    offlinePlayer.id = socket.id;
    offlinePlayer.socket = socket;
    offlinePlayer.offline = false;
    offlinePlayer.offlineTime = null;
    playerSockets.set(socket.id, this);
    
    console.log(`玩家 ${username} 重连房间 ${this.code}，座位: ${offlinePlayer.seatIndex}`);
}
```

#### 步骤3: 取消自动解散计时器

```javascript
if (this.autoDissolveTimer) {
    clearTimeout(this.autoDissolveTimer);
    this.autoDissolveTimer = null;
    
    this.broadcast('room_auto_dissolve_cancelled', {
        message: '有玩家重连，房间自动解散已取消'
    });
}
```

#### 步骤4: 广播重连事件

```javascript
this.broadcast('player_reconnected', { 
    username: username, 
    seatIndex: offlinePlayer.seatIndex 
});
this.broadcastRoomUpdate();
```

#### 步骤5: 发送游戏状态

```javascript
if (this.gameRunning) {
    socket.emit('game_started', {
        gameState: this.getPlayerGameState(socket.id),
        dealerIndex: this.gameState.dealerIndex,
        yourSeat: offlinePlayer.seatIndex,
        currentRound: this.currentRound,
        totalRounds: this.totalRounds,
        matchScores: this.matchScores,
        isReconnect: true
    });
}
```

#### 步骤6: 恢复控制权

```javascript
if (this.gameState.currentPlayerIndex === offlinePlayer.seatIndex) {
    if (this.gameState.turnPhase === 'discard') {
        this.setDiscardTimeout(offlinePlayer);
        
        setTimeout(() => {
            socket.emit('your_turn', {
                phase: 'discard',
                message: '轮到你出牌了！'
            });
            socket.emit('discard_countdown', { 
                seconds: GAME_TIMEOUT_CONFIG.DISCARD_TIMEOUT_MS / 1000 
            });
        }, 200);
    } else if (this.gameState.turnPhase === 'draw') {
        socket.emit('your_turn', {
            phase: 'draw',
            message: '轮到你摸牌了！'
        });
    }
}
```

#### 步骤7: 恢复待处理动作

```javascript
const pendingAction = this.gameState.pendingActions?.find(
    a => a.playerId === socket.id
);
if (pendingAction && !pendingAction.resolved) {
    socket.emit('action_available', {
        actions: pendingAction.actions,
        tile: pendingAction.tile
    });
}
```

### 4.3 getPlayerGameState 方法

**文件**: `server.js` 第652-702行

```javascript
getPlayerGameState(playerId, lightweight = false) {
    return {
        players: this.players.map(p => ({
            id: p.id,
            username: p.username,
            avatar: p.avatar,
            voice: p.voice || 'female01',
            seatIndex: p.seatIndex,
            wind: p.wind,
            windName: WIND_NAMES[p.wind],
            isBot: p.isBot,
            isHost: p.isHost,
            offline: p.offline || false,
            aiTakeover: p.aiTakeover || false,
            handCount: p.hand.length,
            hand: p.id === playerId ? p.hand : null,  // 只发送自己的手牌
            melds: p.melds,
            discards: p.discards,
            flowers: p.flowers,
            isTing: p.isTing,
            isQiao: p.isQiao
        })),
        currentPlayerIndex: this.gameState.currentPlayerIndex,
        turnPhase: this.gameState.turnPhase,
        lastDiscard: this.gameState.lastDiscard,
        lastDiscardPlayer: this.gameState.lastDiscardPlayer,
        deckRemaining: this.gameState.deck.length,
        dealerIndex: this.gameState.dealerIndex,
        roundNumber: this.gameState.roundNumber
    };
}
```

---

## 五、客户端接收阶段

### 5.1 接收 room_joined

**文件**: `index.html` 第3704-3709行

```javascript
socket.on('room_joined', (data) => {
    currentRoom = data.roomCode;
    sessionStorage.setItem('mahjong_room_code', data.roomCode);
    sessionStorage.setItem('mahjong_username', username);
    showRoomScreen();
});
```

### 5.2 接收 player_reconnected

**文件**: `index.html` 第3725-3727行

```javascript
socket.on('player_reconnected', (data) => {
    showToast(`✅ ${data.username} 已重连！`);
});
```

### 5.3 接收 game_started（核心重连处理）

**文件**: `index.html` 第3730-3787行

```javascript
socket.on('game_started', (data) => {
    mySeatIndex = data.yourSeat;
    gameState = data.gameState;
    gameRunning = true;
    
    // 重置听牌状态
    isTing = false;
    isQiao = false;
    tingList = [];
    lastDrawnTileId = null;
    selectedTileId = null;
    pendingQiao = false;
    
    // 检查是否被AI接管
    const myPlayer = gameState.players.find(p => p.seatIndex === mySeatIndex);
    if (myPlayer && myPlayer.aiTakeover) {
        isAITakeover = true;
        showTakeoverButton();
        showToast('⚠️ AI正在代替你进行游戏，点击"接管AI"恢复控制', 5000);
    } else {
        isAITakeover = false;
        hideTakeoverButton();
    }
    
    // 更新局数和积分显示
    if (data.currentRound !== undefined) {
        updateRoundDisplay(data.currentRound, data.totalRounds);
    }
    if (data.matchScores) {
        updateScorePanel(data.matchScores);
    }
    
    showGameScreen();
    updateGameUI();
    
    // 处理重连
    if (data.isReconnect) {
        showToast('🔄 已重新连接！继续游戏...', 3000);
        console.log('断线重连成功，座位:', mySeatIndex, '当前轮到:', gameState.currentPlayerIndex);
        
        // 如果轮到自己，自动摸牌或提示出牌
        if (gameState.currentPlayerIndex === mySeatIndex) {
            if (gameState.turnPhase === 'draw') {
                setTimeout(() => autoDrawTile(), 500);
            }
        }
    } else {
        showToast(`第 ${data.currentRound || 1}/${data.totalRounds || 10} 局开始！`);
    }
});
```

### 5.4 接收 your_turn

**文件**: `index.html` 第3790-3798行

```javascript
socket.on('your_turn', (data) => {
    console.log('轮到你了:', data);
    showToast(`🎯 ${data.message}`, 3000);
    
    if (data.phase === 'draw') {
        setTimeout(() => autoDrawTile(), 300);
    }
});
```

---

## 六、关键判断条件

### 6.1 重连成功的条件

| 条件 | 说明 |
|-----|------|
| 房间存在 | `gameRooms.has(roomCode)` |
| 游戏正在进行 | `room.gameRunning === true` |
| 用户名匹配 | `offlinePlayer.username === username` |
| 玩家离线状态 | `offlinePlayer.offline === true` |
| 玩家不是AI | `offlinePlayer.isBot === false` |

### 6.2 恢复控制权的条件

```javascript
if (this.gameState.currentPlayerIndex === offlinePlayer.seatIndex) {
    // 恢复控制权
    if (this.gameState.turnPhase === 'discard') {
        // 出牌阶段
    } else if (this.gameState.turnPhase === 'draw') {
        // 摸牌阶段
    }
}
```

---

## 七、数据恢复内容

### 7.1 服务端发送给客户端的数据

```javascript
{
    gameState: {
        players: [
            {
                id: string,
                username: string,
                seatIndex: number,
                wind: number,
                offline: boolean,
                aiTakeover: boolean,
                hand: number[] | null,  // 只有自己可见
                melds: array,
                discards: array,
                flowers: array,
                isTing: boolean,
                isQiao: boolean
            }
        ],
        currentPlayerIndex: number,
        turnPhase: 'draw' | 'discard' | 'action',
        lastDiscard: number | null,
        deckRemaining: number
    },
    dealerIndex: number,
    yourSeat: number,
    currentRound: number,
    totalRounds: number,
    matchScores: array,
    isReconnect: true
}
```

### 7.2 客户端本地状态恢复

| 状态变量 | 恢复值 |
|---------|-------|
| `mySeatIndex` | `data.yourSeat` |
| `gameState` | `data.gameState` |
| `gameRunning` | `true` |
| `isTing` | `false` |
| `isQiao` | `false` |
| `isAITakeover` | 根据 `aiTakeover` 字段 |
| `currentRoom` | `data.roomCode` |

---

## 八、关键文件位置索引

| 功能 | 文件位置 |
|-----|---------|
| 客户端发送 join_room | `index.html:4331` |
| 服务端接收 join_room | `server.js:2796-2842` |
| addPlayer 重连处理 | `server.js:233-321` |
| getPlayerGameState | `server.js:652-702` |
| 客户端接收 room_joined | `index.html:3704-3709` |
| 客户端接收 player_reconnected | `index.html:3725-3727` |
| 客户端接收 game_started | `index.html:3730-3787` |
| 客户端接收 your_turn | `index.html:3790-3798` |

---

## 九、流程要点总结

```
┌─────────────────────────────────────────────────────────────────────┐
│                     重连流程关键要点                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 客户端发起                                                      │
│     - 用户输入相同的 username 和 roomCode                           │
│     - 发送 join_room 事件                                           │
│                                                                     │
│  2. 服务端验证                                                      │
│     - 检查房间存在                                                  │
│     - 检查游戏进行中                                                │
│     - 通过 username 匹配离线玩家                                    │
│                                                                     │
│  3. 状态恢复                                                        │
│     - 更新 socket.id                                                │
│     - 设置 offline = false                                         │
│     - 清除离线时间                                                  │
│     - 取消自动解散计时器                                            │
│                                                                     │
│  4. 事件广播                                                        │
│     - 广播 player_reconnected                                       │
│     - 广播 room_updated                                             │
│     - 广播 room_auto_dissolve_cancelled                            │
│                                                                     │
│  5. 数据同步                                                        │
│     - 发送完整 gameState（isReconnect: true）                      │
│     - 发送 your_turn（如果轮到该玩家）                              │
│     - 发送 action_available（有待处理动作时）                       │
│                                                                     │
│  6. 客户端恢复                                                      │
│     - 显示房间界面 → 游戏界面                                       │
│     - 更新 gameState                                                │
│     - 显示重连成功提示                                              │
│     - 如轮到自动摸牌                                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```
