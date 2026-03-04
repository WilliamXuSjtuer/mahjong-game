# Bug修复：杠上炮（暗杠）未触发

## 问题描述

暗杠后，杠上炮标记没有正确传递到胡牌结算，导致杠上炮分数没有正确计算。

## 问题原因

在 `discardTile` 函数中，`gangShangPao` 标记被清除的时机不对：

```javascript
// 出牌时
const tile = player.hand.splice(tileIndex, 1)[0];
player.discards.push(tile);

this.gameState.lastDiscard = tile;
this.gameState.lastDiscardPlayer = player.seatIndex;
this.gameState.lastDrawnTile = null;
this.gameState.gangShangPao = false; // ← 问题：在这里就清除了

// 然后才检查其他玩家是否可以胡牌
this.checkActionsAfterDiscard(tile, player.seatIndex);
```

当 `checkActionsAfterDiscard` 检查到有玩家可以胡牌时，`gangShangPao` 已经被清除为 `false`。

## 正确逻辑

杠上炮的定义：杠牌后，杠者摸牌再打出，被其他玩家胡牌。

流程应该是：
1. 暗杠/明杠 → 设置 `gangShangPao = true`
2. 杠后摸牌
3. 杠者出牌 → 此时应该保存 `gangShangPao` 状态
4. 检查其他玩家是否可以胡牌 → 使用保存的状态
5. 如果有人胡牌 → 结算时清除标记

## 修复方案

### 方案1：延迟清除标记

在胡牌结算完成后才清除 `gangShangPao` 标记：

```javascript
// 出牌时不清除，改为在以下情况清除：
// 1. 没有人胡牌，进入下一轮
// 2. 胡牌结算完成
```

### 方案2：将标记保存在弃牌信息中

```javascript
this.gameState.lastDiscard = tile;
this.gameState.lastDiscardPlayer = player.seatIndex;
this.gameState.lastDiscardIsGangShangPao = this.gameState.gangShangPao; // 保存状态
this.gameState.gangShangPao = false; // 清除标记
```

然后在胡牌时使用 `lastDiscardIsGangShangPao`。

## 采用方案

采用方案1，因为更简单且不需要修改数据结构。

### 修改点

1. **`discardTile` 函数**：不在出牌时清除 `gangShangPao`
2. **`nextTurn` 函数**：在进入下一轮时清除 `gangShangPao`
3. **`endRound` 函数**：在胡牌结算后清除 `gangShangPao`

## 测试场景

1. 玩家A暗杠
2. 玩家A摸牌
3. 玩家A出牌
4. 玩家B胡牌
5. 验证：分数应该是正常分数 × 3

## 修复日期

2026-03-04
