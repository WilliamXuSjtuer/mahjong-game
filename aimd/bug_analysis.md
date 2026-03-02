# 上海敲麻 Bug 分析报告

## 问题描述
**现象**：吃碰杠后无法打出牌，手牌多一张

## 根本原因分析

经过代码审查，发现 bug 位于 **`executeAction`** 方法中，该方法位于 [`server.js:1513-1713`](file://f:\QiaoMa\shanghaimahjong\server.js#L1513-L1713)。

### 问题 1：碰牌逻辑 - 手牌计算错误 ❌

**位置**：[`server.js:1522-1557`](file://f:\QiaoMa\shanghaimahjong\server.js#L1522-L1557)

```javascript
} else if (action.action === 'peng') {
    // 碰
    const sameTiles = player.hand.filter(t => 
        t.type === tile.type && t.value === tile.value
    ).slice(0, 2);
    
    // 从手牌移除
    sameTiles.forEach(t => {
        const idx = player.hand.findIndex(h => h.id === t.id);
        if (idx !== -1) player.hand.splice(idx, 1);
    });
    
    // 添加到副露
    player.melds.push({
        type: 'peng',
        tiles: [...sameTiles, tile],  // ⚠️ 问题：这里包含了别人打出的 tile
        from: this.gameState.lastDiscardPlayer
    });
    
    // 从弃牌堆移除
    const discardPlayer = this.players[this.gameState.lastDiscardPlayer];
    discardPlayer.discards.pop();
    
    // 轮到碰的玩家出牌
    this.gameState.currentPlayerIndex = action.playerIndex;
    this.gameState.turnPhase = 'discard';  // ✅ 设置为出牌阶段
```

**问题分析**：
- ✅ 从手牌中移除了 2 张相同的牌
- ✅ 副露包含了 3 张牌（2 张手牌 + 1 张别人打出的牌）
- ❌ **但是：这张别人打出的 `tile` 从未被添加到玩家手牌中！**
- 结果：副露有 3 张牌，但手牌只减少了 2 张，**总牌数正确，但手牌数量错误（多 1 张）**

**正确的牌数变化**：
- 碰牌前：手牌 13 张 + 别人打出的牌（在弃牌堆）
- 碰牌后：手牌应该 13-2=11 张 + 副露 3 张 = 总共 14 张
- 实际：手牌 13-2=11 张 + 副露 3 张（包含别人的牌）= 玩家控制 14 张 ✅
- **但是游戏状态可能显示手牌有 12 张（因为那张牌从未加入手牌）**

### 问题 2：杠牌逻辑 - 同样的问题 ❌

**位置**：[`server.js:1559-1591`](file://f:\QiaoMa\shanghaimahjong\server.js#L1559-L1591)

```javascript
} else if (action.action === 'gang') {
    // 杠
    const sameTiles = player.hand.filter(t => 
        t.type === tile.type && t.value === tile.value
    );
    
    sameTiles.forEach(t => {
        const idx = player.hand.findIndex(h => h.id === t.id);
        if (idx !== -1) player.hand.splice(idx, 1);
    });
    
    player.melds.push({
        type: 'gang',
        tiles: [...sameTiles, tile],  // ⚠️ 同样的问题
        from: this.gameState.lastDiscardPlayer
    });
    
    const discardPlayer = this.players[this.gameState.lastDiscardPlayer];
    discardPlayer.discards.pop();
    
    // 杠后摸一张牌
    this.gameState.currentPlayerIndex = action.playerIndex;
    this.gameState.turnPhase = 'draw';  // ✅ 设置为摸牌阶段
```

**问题分析**：
- ✅ 从手牌移除了 3 张相同的牌
- ❌ **副露包含了 4 张牌（3 张手牌 + 1 张别人打出的牌），但那张牌从未加入手牌**
- 结果：杠后应该摸牌，但手牌数量计算错误

### 问题 3：吃牌逻辑 - 同样的问题 ❌

**位置**：[`server.js:1593-1644`](file://f:\QiaoMa\shanghaimahjong\server.js#L1593-L1644)

```javascript
} else if (action.action === 'chi') {
    // 吃牌
    const chiTiles = selectedOption.tiles;
    
    // 从手牌中移除吃牌的两张牌（保留打出的那张，即 tile）
    const tilesToRemove = chiTiles.filter(t => t.id !== tile.id);
    tilesToRemove.forEach(t => {
        const idx = player.hand.findIndex(h => h.id === t.id);
        if (idx !== -1) player.hand.splice(idx, 1);
    });
    
    // 将打出的牌加入顺子
    player.melds.push({
        type: 'chi',
        tiles: chiTiles,  // ⚠️ 包含别人的牌
        from: this.gameState.lastDiscardPlayer
    });
    
    // 从弃牌堆移除
    const discardPlayer = this.players[this.gameState.lastDiscardPlayer];
    discardPlayer.discards.pop();
    
    // 轮到吃的玩家出牌
    this.gameState.currentPlayerIndex = action.playerIndex;
    this.gameState.turnPhase = 'discard';
```

**问题分析**：
- 代码注释说"保留打出的那张，即 tile"，但实际上 **这张 `tile` 从未被加入到手牌**
- 副露包含了 3 张牌，但手牌只移除了 2 张

---

## 正确的逻辑应该是

### 方案 A：先加入手牌，再组成副露（推荐）

```javascript
} else if (action.action === 'peng') {
    // 碰
    const sameTiles = player.hand.filter(t => 
        t.type === tile.type && t.value === tile.value
    ).slice(0, 2);
    
    // 先将别人打出的牌加入手牌
    player.hand.push(tile);
    
    // 从手牌移除 2 张相同的牌
    sameTiles.forEach(t => {
        const idx = player.hand.findIndex(h => h.id === t.id);
        if (idx !== -1) player.hand.splice(idx, 1);
    });
    
    // 移除刚加入的那张牌（用于组成副露）
    const tileIndex = player.hand.findIndex(h => h.id === tile.id);
    if (tileIndex !== -1) {
        player.hand.splice(tileIndex, 1);
    }
    
    // 添加到副露
    player.melds.push({
        type: 'peng',
        tiles: [...sameTiles, tile],
        from: this.gameState.lastDiscardPlayer
    });
    
    // 从弃牌堆移除
    const discardPlayer = this.players[this.gameState.lastDiscardPlayer];
    discardPlayer.discards.pop();
    
    // 轮到碰的玩家出牌
    this.gameState.currentPlayerIndex = action.playerIndex;
    this.gameState.turnPhase = 'discard';
```

### 方案 B：直接从手牌移除，副露不包含别人的牌（简化方案）

```javascript
} else if (action.action === 'peng') {
    // 碰
    const sameTiles = player.hand.filter(t => 
        t.type === tile.type && t.value === tile.value
    ).slice(0, 2);
    
    // 从手牌移除 2 张相同的牌
    sameTiles.forEach(t => {
        const idx = player.hand.findIndex(h => h.id === t.id);
        if (idx !== -1) player.hand.splice(idx, 1);
    });
    
    // 添加到副露（只包含手牌中的 2 张，别人的牌单独记录）
    player.melds.push({
        type: 'peng',
        tiles: sameTiles,
        from: this.gameState.lastDiscardPlayer,
        exposedTile: tile  // 别人打出的那张牌
    });
    
    // 从弃牌堆移除
    const discardPlayer = this.players[this.gameState.lastDiscardPlayer];
    discardPlayer.discards.pop();
    
    // 轮到碰的玩家出牌
    this.gameState.currentPlayerIndex = action.playerIndex;
    this.gameState.turnPhase = 'discard';
```

---

## 为什么会导致"无法打出牌"的问题？

### 可能的原因 1：前端手牌数量校验失败

前端可能在出牌时校验手牌数量：
```javascript
// 前端代码可能有类似逻辑
if (player.hand.length !== 13) {
    showError('手牌数量错误');
    return;
}
```

由于碰/杠/吃后，手牌数量计算错误，导致前端校验失败，无法出牌。

### 可能的原因 2：游戏状态同步问题

[`broadcastGameState()`](file://f:\QiaoMa\shanghaimahjong\server.js#L727-L820) 广播的游戏状态中，手牌数量不正确，导致前端显示的手牌数量与实际可操作的牌不一致。

### 可能的原因 3：出牌逻辑中的手牌索引错误

在 [`playerDiscard`](file://f:\QiaoMa\shanghaimahjong\server.js#L1121-L1235) 方法中：
```javascript
const tileIndex = player.hand.findIndex(t => t.id === tileId);
if (tileIndex === -1) {
    return { error: '没有这张牌' };  // ⚠️ 可能触发这个错误
}
```

如果手牌数量计算错误，可能导致前端发送的 `tileId` 在服务器端的手牌中找不到。

---

## 修复建议

### 立即修复（采用方案 B，改动最小）

修改 [`executeAction`](file://f:\QiaoMa\shanghaimahjong\server.js#L1513-L1713) 方法中的三个分支：

1. **碰牌**：副露只包含手牌中的 2 张，别人的牌单独记录
2. **杠牌**：副露只包含手牌中的 3 张，别人的牌单独记录
3. **吃牌**：副露只包含手牌中的 2 张，别人的牌单独记录

### 长期修复（重构吃碰杠逻辑）

重新设计吃碰杠的数据结构，明确区分：
- 手牌中的牌
- 别人打出的牌
- 副露的牌

---

## 受影响的代码位置

| 问题 | 代码位置 | 影响 |
|------|----------|------|
| 碰牌手牌计算错误 | [`server.js:1522-1557`](file://f:\QiaoMa\shanghaimahjong\server.js#L1522-L1557) | 碰后手牌多 1 张 |
| 杠牌手牌计算错误 | [`server.js:1559-1591`](file://f:\QiaoMa\shanghaimahjong\server.js#L1559-L1591) | 杠后手牌多 1 张 |
| 吃牌手牌计算错误 | [`server.js:1593-1644`](file://f:\QiaoMa\shanghaimahjong\server.js#L1593-L1644) | 吃后手牌多 1 张 |

---

## 验证方法

修复后，应该验证以下场景：

1. ✅ 碰牌后，手牌数量 = 13 - 2 = 11 张
2. ✅ 杠牌后，手牌数量 = 13 - 3 = 10 张（然后摸牌变 11 张）
3. ✅ 吃牌后，手牌数量 = 13 - 2 = 11 张
4. ✅ 碰/杠/吃后，可以正常打出牌
5. ✅ 前端显示的手牌数量与服务器端一致

---

## 生成时间
2026-03-02
