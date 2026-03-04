# Bug修复：checkAnGangAvailable 中 player 为 undefined

## 问题描述

```
TypeError: Cannot read properties of undefined (reading 'hand')
    at MahjongRoom.checkAnGangAvailable (F:\QiaoMa\shanghaimahjong\server.js:2219:16)
    at MahjongRoom.startGame (F:\QiaoMa\shanghaimahjong\server.js:807:42)
```

## 问题原因

在 `startGame` 函数中：
```javascript
const dealer = this.players[dealerIndex];
const dealerAnGangActions = this.checkAnGangAvailable(dealer);
```

`dealerIndex` 的计算方式：
- 第一局：`Math.floor(Math.random() * 4)` - 可能返回 0-3
- 后续局：`this.lastWinnerIndex` - 可能是无效值

如果 `this.players` 数组长度小于4，或者 `dealerIndex` 无效，则 `this.players[dealerIndex]` 返回 `undefined`。

## 可能的场景

1. **竞态条件**：玩家在游戏开始前离开，导致玩家数不足4人
2. **lastWinnerIndex 无效**：上局赢家索引可能被错误设置
3. **fillWithAI 未完成**：AI填充可能在游戏开始前未完成

## 修复方案

### 1. 在 `startGame` 开始时检查玩家数量

```javascript
startGame() {
    if (this.gameRunning) return;
    
    // 【修复】确保玩家数量正确
    if (this.players.length < 4) {
        console.log(`玩家数量不足(${this.players.length})，填充AI`);
        this.fillWithAI();
    }
    
    // 确保 dealerIndex 有效
    if (dealerIndex >= this.players.length) {
        dealerIndex = 0;
    }
    
    // ...
}
```

### 2. 在访问 dealer 前检查有效性

```javascript
const dealer = this.players[dealerIndex];
if (!dealer) {
    console.error(`无效的庄家索引: ${dealerIndex}, 玩家数: ${this.players.length}`);
    return;
}
const dealerAnGangActions = this.checkAnGangAvailable(dealer);
```

## 修改的文件

- `server.js` - `startGame` 函数

## 测试场景

1. 只有1个真人玩家，3个位置空着
2. 点击准备
3. 验证：游戏正常开始，AI填充空位

## 修复日期

2026-03-04
