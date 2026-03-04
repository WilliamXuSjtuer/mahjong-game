# Bug修复：AI不触发杠上炮

## 问题描述

当AI玩家杠牌后出牌，其他玩家胡牌时，杠上炮没有正确触发。

## 问题原因

在 `aiDiscard` 函数中，AI出牌时错误地立即清除了 `gangShangPao` 标记：

```javascript
// AI出牌
aiDiscard(aiPlayer) {
    // ... 出牌逻辑 ...
    
    this.gameState.lastDrawnTile = null;
    this.gameState.gangShangPao = false; // ← 问题：立即清除了标记
    
    // 然后才检查其他玩家是否可以胡牌
    this.checkActionsAfterDiscard(discardTile, aiPlayer.seatIndex);
}
```

这与真人玩家出牌时的bug相同，在检查胡牌之前就已经清除了杠上炮标记。

## 修复方案

移除 `aiDiscard` 中清除 `gangShangPao` 标记的代码，让标记在以下情况清除：
1. 进入下一轮时（`nextTurn`）
2. 胡牌结算后（`endRound`）

## 修改的文件

- `server.js` - `aiDiscard` 函数

## 测试场景

1. AI玩家杠牌
2. AI玩家摸牌
3. AI玩家出牌
4. 其他玩家胡牌
5. 验证：分数应该是正常分数 × 3

## 修复日期

2026-03-04
