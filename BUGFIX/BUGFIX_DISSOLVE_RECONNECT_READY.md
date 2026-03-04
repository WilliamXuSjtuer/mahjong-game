# Bug修复：提议解散后回到主页，再加入房间后准备状态未重置

## 问题描述

当玩家发起解散投票后返回主页，再加入房间时：
1. 解散投票状态残留
2. 玩家可能看到解散投票弹窗
3. 准备状态未正确重置

## 问题原因

当玩家在游戏进行中离开房间时：
1. 玩家被标记为离线状态
2. 解散投票自动投同意票
3. 但如果解散投票未完成，`dissolveRequest` 和 `dissolveVotes` 状态残留

当玩家重连时：
1. 离线状态被清除
2. 但解散投票状态没有清除
3. 导致玩家可能看到之前的解散投票弹窗

## 修复方案

### 方案1：玩家离开时取消解散请求

当发起解散的玩家离开房间时，取消解散请求：

```javascript
// 在 removePlayer 函数中，当玩家离线时
if (this.dissolveRequest && this.dissolveRequest.requesterId === socketId) {
    // 发起者离开，取消解散请求
    this.dissolveRequest = null;
    this.dissolveVotes = {};
    this.broadcast('dissolve_cancelled', { reason: '发起者已离线' });
}
```

### 方案2：重连时清除解散投票状态

当玩家重连时，清除解散投票状态：

```javascript
// 在 addPlayer 函数的重连逻辑中
// 清除解散投票状态（因为玩家已经离开过）
this.dissolveRequest = null;
this.dissolveVotes = {};
```

## 采用方案

采用方案1，因为：
1. 更符合逻辑：发起者离开，投票应该取消
2. 避免其他玩家被残留的投票状态影响

## 修改的文件

- `server.js` - `removePlayer` 函数

## 测试场景

1. 开始游戏
2. 玩家A发起解散投票
3. 玩家A返回主页
4. 验证：解散投票被取消，其他玩家不再看到投票弹窗
5. 玩家A重新加入房间
6. 验证：玩家正常重连，没有残留的解散投票状态

## 修复日期

2026-03-04
