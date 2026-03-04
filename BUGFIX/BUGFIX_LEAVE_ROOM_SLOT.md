# Bug修复：等待房间中离开时位置不会空出

## 问题描述

当玩家在等待房间中（游戏未开始）离开房间时，玩家的位置不会空出，而是被标记为"离线"状态。这导致：

1. 其他玩家无法加入该位置
2. 房间显示该玩家仍在，但状态为离线
3. 游戏无法正常开始（因为离线玩家无法准备）

## 问题原因

在 `removePlayer` 函数中，对于真人玩家的处理逻辑：

```javascript
// 真人玩家断线，标记为离线状态（无论游戏是否在进行，都保留玩家信息以便重连）
if (!player.isBot) {
    player.offline = true;
    // ...
    return;
}
```

这段代码无论游戏是否开始，都将真人玩家标记为离线，而不是真正移除。

## 正确逻辑

- **游戏未开始时**：真人玩家离开应该真正移除，空出位置
- **游戏进行中**：真人玩家离开应该标记为离线，以便重连

## 修复方案

修改 `removePlayer` 函数，增加游戏状态判断：

```javascript
// 移除玩家
removePlayer(socketId) {
    const playerIndex = this.players.findIndex(p => p.id === socketId);
    if (playerIndex !== -1) {
        const player = this.players[playerIndex];
        playerSockets.delete(socketId);
        
        // 游戏进行中：真人玩家断线，标记为离线状态以便重连
        if (!player.isBot && this.gameRunning) {
            player.offline = true;
            player.offlineTime = Date.now();
            player.socket = null;
            // ... 其他离线处理逻辑
            return;
        }
        
        // 游戏未开始或AI玩家：真正移除
        this.players.splice(playerIndex, 1);
        // ... 其他移除逻辑
    }
}
```

## 修改的文件

- `server.js` - `removePlayer` 函数

## 测试场景

1. **等待房间中离开**：
   - 玩家A创建房间
   - 玩家B加入房间
   - 玩家B离开房间
   - 验证：玩家B的位置已空出，其他玩家可以加入

2. **游戏中离开**：
   - 游戏进行中
   - 玩家B断开连接
   - 验证：玩家B被标记为离线，可以重连

## 修复日期

2026-03-04
