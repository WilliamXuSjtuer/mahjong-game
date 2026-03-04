# Bug修复：解散房间后房间不重置

## 问题描述

当玩家投票解散游戏后，房间没有被正确清理：
1. 房间没有从 `gameRooms` Map 中删除
2. 房间状态没有重置（`gameRunning`、`dissolveRequest`、`dissolveVotes` 等）
3. 计时器没有被清除
4. 玩家可以继续在已解散的房间中操作

## 问题原因

`endGameForDissolve` 函数只做了以下操作：
1. 清除部分计时器
2. 设置 `gameRunning = false`
3. 广播 `game_dissolved` 事件

但没有：
1. 调用 `cleanup()` 清理资源
2. 从 `gameRooms` 中删除房间
3. 重置解散相关的状态

## 修复方案

修改 `endGameForDissolve` 函数，添加完整的清理逻辑：

```javascript
endGameForDissolve() {
    // 清除所有计时器
    // ... 现有代码 ...
    
    // 【新增】重置解散相关状态
    this.dissolveRequest = null;
    this.dissolveVotes = {};
    
    // 【新增】重置游戏状态
    this.gameRunning = false;
    this.isPaused = false;
    this.pausePlayer = null;
    this.matchStarted = false;
    this.currentRound = 0;
    this.matchScores = [0, 0, 0, 0];
    this.roundHistory = [];
    this.huangFanCount = 0;
    this.isHuangFanRound = false;
    
    // 【新增】重置玩家状态
    this.players.forEach(p => {
        p.hand = [];
        p.melds = [];
        p.discards = [];
        p.flowers = [];
        p.isTing = false;
        p.isQiao = false;
        p.ready = false;
        p.sankouCounts = [0, 0, 0, 0];
    });
    
    // 广播游戏解散
    this.broadcast('game_dissolved', {
        matchScores: this.matchScores,
        roundHistory: this.roundHistory,
        totalRounds: this.totalRounds,
        currentRound: this.currentRound
    });
    
    // 【新增】清理资源并删除房间
    this.cleanup();
    gameRooms.delete(this.code);
    console.log(`房间 ${this.code} 已解散并清理`);
}
```

## 修改的文件

- `server.js` - `endGameForDissolve` 函数

## 测试场景

1. 创建房间并开始游戏
2. 发起解散投票
3. 所有玩家同意解散
4. 验证：
   - 房间从列表中消失
   - 玩家返回大厅
   - 无法继续在已解散房间中操作

## 修复日期

2026-03-04
