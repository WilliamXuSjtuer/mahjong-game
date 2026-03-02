# 敲后不能暗杠 Bug 分析

## 问题描述
听牌后手牌中有三个北风，又摸了一张北风，但是直接打掉了，应该可以自己选择是否暗杠。

## 问题分析

### 当前游戏逻辑

目前游戏中存在三种杠：
1. **明杠（gang）**：别人打出的牌，自己手中有 3 张相同的
2. **加杠（jia_gang）**：已有副露刻子（碰的牌），自己摸到第 4 张
3. **暗杠（an_gang）**：手中有 3 张相同的牌，自己摸到第 4 张（**缺失**）

### 问题根源

**位置**: `server.js` 第 1040-1119 行（`playerDraw` 函数）

在玩家摸牌后，服务器只检查了：
1. 自摸胡牌（第 1062-1078 行）
2. 加杠（第 1080-1104 行）

**但是没有检查暗杠**！

```javascript
// 检查加杠（摸到的牌可以与副露中的刻子组成杠）
const jiaGangActions = [];
for (const meld of player.melds) {
    if (meld.type === 'peng' && meld.tiles && meld.tiles.length > 0) {
        const pengTile = meld.tiles[0];
        if (pengTile.type === tile.type && pengTile.value === tile.value) {
            jiaGangActions.push({...});
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

**缺失的暗杠检查逻辑**：
```javascript
// 应该检查暗杠（手中有 3 张相同的牌，摸到第 4 张）
const anGangActions = [];
for (const meld of player.melds) {
    // 暗杠要求副露为空（没有副露过的牌）
    // 或者检查手牌中是否有 3 张相同的
}
```

### 敲牌后的特殊情况

根据游戏规则：
- 敲牌后，玩家只能打刚摸的牌
- 但敲牌后**应该仍然可以暗杠**

当前代码在第 1134-1139 行有敲牌限制：
```javascript
// 【敲牌限制】如果已敲牌，只能打刚摸的牌
if (player.isQiao && this.gameState.lastDrawnTile) {
    if (tileId !== this.gameState.lastDrawnTile.id) {
        return { error: '已敲牌，只能打刚摸的牌！' };
    }
}
```

这个限制是正确的，但问题是：
1. 摸牌时没有提示暗杠选项
2. 玩家不知道可以暗杠
3. 所以直接打出了牌

## 解决方案

### 方案 1：添加暗杠检测（推荐）

在 `playerDraw` 函数中，添加暗杠检测逻辑：

```javascript
// 检查暗杠（手中有 3 张相同的牌，摸到第 4 张）
// 暗杠的条件：
// 1. 手中有 3 张相同的牌
// 2. 这 3 张牌没有被副露过（即不是从副露中来的）
const anGangActions = [];

// 统计手牌中每种牌的数量
const tileCount = {};
for (const handTile of player.hand) {
    const key = `${handTile.type}-${handTile.value}`;
    tileCount[key] = (tileCount[key] || 0) + 1;
}

// 检查是否有 3 张相同的牌
for (const [key, count] of Object.entries(tileCount)) {
    if (count === 3) {
        const [type, value] = key.split('-');
        if (tile.type === type && tile.value === parseInt(value)) {
            anGangActions.push({
                tile: tile
            });
        }
    }
}

// 如果有暗杠选项且没有自摸，提示暗杠
if (anGangActions.length > 0 && !this.gameState.pendingZimo) {
    if (player.socket) {
        player.socket.emit('action_available', {
            playerId: player.id,
            actions: ['an_gang'],
            tile: tile,
            anGangOptions: anGangActions
        });
    }
}
```

### 方案 2：修改杠的处理逻辑

在 `resolveAction` 中，区分暗杠和明杠：

```javascript
} else if (action.action === 'gang') {
    // 判断是暗杠还是明杠
    const sameTiles = player.hand.filter(t => 
        t.type === tile.type && t.value === tile.value
    );
    
    let isAnGang = false;
    if (sameTiles.length === 3) {
        // 手中有 3 张，是暗杠
        isAnGang = true;
    } else if (sameTiles.length === 0 && tile.from === undefined) {
        // 手中没有，牌是从牌堆摸的，也是暗杠
        isAnGang = true;
    }
    
    // 执行杠操作
    ...
    
    // 广播时区分暗杠和明杠
    this.broadcast('action_executed', {
        playerIndex: action.playerIndex,
        action: isAnGang ? 'an_gang' : 'gang',
        tile: tile,
        tileName: getTileName(tile)
    });
}
```

### 方案 3：前端支持暗杠选项

修改前端，支持暗杠按钮和逻辑：

1. 添加暗杠按钮
2. 监听 `action_available` 事件，当有 `an_gang` 选项时显示按钮
3. 点击暗杠按钮时发送 `an_gang` 动作

## 推荐修复方案

**采用方案 1 + 方案 3 的组合**：

1. 服务器端在摸牌时检测暗杠选项
2. 通过 `action_available` 事件通知前端
3. 前端显示暗杠按钮
4. 玩家可以选择暗杠或出牌

## 注意事项

1. **暗杠 vs 加杠的优先级**：
   - 如果同时有暗杠和加杠选项，应该都显示
   - 或者根据游戏规则确定优先级

2. **敲牌后的限制**：
   - 敲牌后只能打刚摸的牌
   - 但敲牌后应该仍然可以暗杠刚摸的牌

3. **暗杠的计分**：
   - 暗杠计 2 花（普通牌）或 3 花（风牌）
   - 需要在计分逻辑中正确处理

4. **AI 的暗杠逻辑**：
   - AI 应该自动暗杠（必杠策略）
   - 或者根据策略决定

## 已实施的修复

### 修复 1: server.js - 添加暗杠检测逻辑（第 1101-1125 行）

在 `playerDraw` 函数中，摸牌后检查是否可以暗杠：

```javascript
// 检查暗杠（手中有 3 张相同的牌，摸到第 4 张）
const anGangActions = [];
const sameTilesInHand = player.hand.filter(t => 
    t.type === tile.type && t.value === tile.value
);
if (sameTilesInHand.length === 3) {
    // 手中有 3 张，加上刚摸的这张正好 4 张，可以暗杠
    anGangActions.push({
        tile: tile
    });
}

// 如果有暗杠选项且没有自摸，优先提示暗杠
if (anGangActions.length > 0 && !this.gameState.pendingZimo) {
    if (player.socket) {
        player.socket.emit('action_available', {
            playerId: player.id,
            actions: ['an_gang'],
            tile: tile,
            anGangOptions: anGangActions
        });
    }
} else if (jiaGangActions.length > 0 && !this.gameState.pendingZimo) {
    // 如果没有暗杠，再检查加杠
    ...
}
```

### 修复 2: server.js - 添加暗杠动作处理（第 1616-1645 行）

在 `resolveAction` 函数中，处理暗杠动作：

```javascript
} else if (action.action === 'an_gang') {
    // 暗杠：手中有 3 张相同的牌，摸到第 4 张
    const sameTiles = player.hand.filter(t => 
        t.type === tile.type && t.value === tile.value
    );
    
    sameTiles.forEach(t => {
        const idx = player.hand.findIndex(h => h.id === t.id);
        if (idx !== -1) player.hand.splice(idx, 1);
    });
    
    player.melds.push({
        type: 'gang',
        tiles: sameTiles,
        from: player.seatIndex  // 暗杠的 from 是自己
    });
    
    this.broadcast('action_executed', {
        playerIndex: action.playerIndex,
        action: 'an_gang',
        tile: tile,
        tileName: getTileName(tile)
    });
    
    // 杠后摸一张牌
    this.gameState.currentPlayerIndex = action.playerIndex;
    this.gameState.turnPhase = 'draw';
    
    this.broadcastGameState();
    this.notifyCurrentPlayer();
}
```

### 修复 3: server.js - AI 暗杠策略（第 1974-1977 行）

在 `aiDecideAction` 函数中，AI 遇到暗杠必杠：

```javascript
} else if (action.actions.includes('an_gang')) {
    // 暗杠必杠
    action.resolved = true;
    action.action = 'an_gang';
}
```

### 修复 4: index.html - 添加暗杠按钮（第 2593 行）

在操作提示区域添加暗杠按钮：

```html
<button class="hint-btn" id="anGangBtn" onclick="doAction('an_gang')" style="display: none; background: linear-gradient(135deg, #e74c3c 0%, #c0392b 100%);">暗杠</button>
```

### 修复 5: index.html - 显示暗杠按钮（第 5080-5098 行）

在 `showResponseButtons` 函数中支持暗杠按钮：

```javascript
const anGangBtn = document.getElementById('anGangBtn');
// 重置按钮
if (anGangBtn) anGangBtn.style.display = 'none';
// 显示按钮
if (anGangBtn && actions.includes('an_gang')) {
    anGangBtn.style.display = 'inline-block';
}
// 保存暗杠选项
if (data && data.anGangOptions) {
    window.currentAnGangOptions = data.anGangOptions;
}
```

### 修复 6: index.html - 处理暗杠动作（第 5246 行）

在 `doAction` 函数中，暗杠动作显示特效：

```javascript
if (action === 'peng' || action === 'gang' || action === 'hu' || action === 'hu_zimo' || action === 'jia_gang' || action === 'an_gang') {
    showActionEffect(action);
}
```

### 修复 7: index.html - 显示暗杠特效（第 4053-4076 行）

在 `action_executed` 监听中处理暗杠：

```javascript
// 停止倒计时
if ((data.action === 'jia_gang' || data.action === 'an_gang') && isMe) {
    stopDiscardCountdown('action_executed: gang');
}
// 显示特效
if (data.action === 'peng' || data.action === 'gang' || data.action === 'hu' || data.action === 'hu_zimo' || data.action === 'jia_gang' || data.action === 'chi' || data.action === 'an_gang') {
    showActionEffect(data.action, playerName);
}
// 显示提示和播放语音
} else if (data.action === 'an_gang') {
    showToast(`${isMe ? '你' : playerName} 暗杠！`);
    speakGang(playerVoice);
}
```

### 修复 8: index.html - 暗杠大字动画（第 6322 行）

在 `showActionEffect` 函数中添加暗杠文本：

```javascript
const effectTexts = {
    'peng': '碰！',
    'gang': '杠！',
    'jia_gang': '加杠！',
    'an_gang': '暗杠！',
    'hu': '胡！',
    'zimo': '自摸！',
    'hu_zimo': '自摸！',
    'chi': '吃！'
};
```

## 修改文件

1. `server.js`:
   - `playerDraw` 函数：添加暗杠检测（第 1101-1125 行）
   - `resolveAction` 函数：处理暗杠动作（第 1616-1645 行）
   - `aiDecideAction` 函数：AI 暗杠策略（第 1974-1977 行）

2. `index.html`:
   - 添加暗杠按钮 HTML（第 2593 行）
   - `showResponseButtons` 函数：显示暗杠按钮（第 5080-5098 行）
   - `doAction` 函数：处理暗杠动作（第 5246 行）
   - `action_executed` 监听：显示暗杠特效（第 4053-4076 行）
   - `showActionEffect` 函数：暗杠大字动画（第 6322 行）

## 测试场景

1. **普通暗杠**：
   - 手中有 3 张北风，摸到第 4 张
   - 应显示暗杠选项

2. **敲牌后暗杠**：
   - 敲牌后手中有 3 张北风，摸到第 4 张
   - 应显示暗杠选项

3. **暗杠后自摸**：
   - 暗杠后摸牌自摸
   - 应正常胡牌

4. **多个暗杠选项**：
   - 手中有 3 张北风 + 3 张南风，摸到第 4 张北风
   - 应只显示北风暗杠选项
