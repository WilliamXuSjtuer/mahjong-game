# Bug修复：解散游戏后还有暗杠标志

## 问题描述

解散游戏后，`gameState` 对象中的状态没有被重置，导致：
1. `gangShangPao` 标志残留
2. `pendingActions` 残留
3. 其他游戏状态残留

这可能导致下一局游戏开始时出现异常。

## 问题原因

在 `endGameForDissolve` 函数中，虽然重置了 `gameRunning`、`matchScores` 等状态，但没有重置 `gameState` 对象本身：

```javascript
// 重置游戏状态
this.gameRunning = false;
this.isPaused = false;
// ... 但没有重置 this.gameState
```

`gameState` 对象包含：
- `gangShangPao`
- `pendingActions`
- `pendingZimo`
- `waitingForQiao`
- 等等

## 修复方案

在 `endGameForDissolve` 函数中添加 `gameState` 对象的重置：

```javascript
// 【修复】重置 gameState 对象
this.gameState = null;
```

## 修改的文件

- `server.js` - `endGameForDissolve` 函数

## 测试场景

1. 开始游戏
2. 进行暗杠操作
3. 发起解散投票
4. 所有玩家同意解散
5. 验证：`gameState` 为 null，下一局游戏正常开始

## 修复日期

2026-03-04
