# Bug修复：动作优先级即时结算

## 问题描述

当同时存在多个待处理动作时（如胡、杠、碰、吃），系统会等待所有玩家都做出选择后才结算。但实际上应该根据优先级立即结算：

**错误行为**：
- 玩家A可以选择碰，玩家B可以选择吃
- 玩家A选择碰后，系统等待玩家B也做出选择
- 等待玩家B选择后才执行碰

**正确行为**：
- 玩家A选择碰后，由于碰的优先级高于吃
- 立即执行碰，取消玩家B的选择机会

## 优先级规则

```
胡 > 杠 > 碰 > 吃 > 过
 4    3    2    1   0
```

## 修复方案

### 修改 `playerAction` 函数

当玩家选择了一个动作后：
1. 检查这个动作的优先级
2. 检查是否还有其他玩家可能选择更高优先级的动作
3. 如果当前动作优先级最高且没有其他玩家可以选择更高优先级的动作，立即执行
4. 否则继续等待其他玩家响应

### 核心逻辑

```javascript
// 优先级：胡 > 杠 > 碰 > 吃 > pass
const priority = { hu: 4, gang: 3, peng: 2, chi: 1, pass: 0 };

// 当玩家选择了一个动作后
const currentPriority = priority[actionType] || 0;

// 检查是否还有其他玩家可以选择更高优先级的动作
const hasHigherPriorityOption = this.gameState.pendingActions.some(a => {
    if (a.resolved) return false; // 已经响应的不算
    if (a.playerId === socketId) return false; // 自己不算
    
    // 检查该玩家的可选动作中是否有更高优先级的
    return a.actions.some(act => (priority[act] || 0) > currentPriority);
});

if (!hasHigherPriorityOption) {
    // 没有更高优先级的选项，立即执行当前动作
    clearTimeout(this.gameState.actionTimeout);
    
    // 取消其他玩家的待处理动作
    this.gameState.pendingActions.forEach(a => {
        if (!a.resolved) {
            a.resolved = true;
            a.action = 'pass';
        }
    });
    
    // 执行当前动作
    pendingAction.resolved = true;
    pendingAction.action = actionType;
    this.resolveActions();
}
```

## 特殊情况处理

### 1. 多人同时可以胡
- 如果多人同时可以胡，需要等待所有人都响应
- 因为胡的优先级相同，需要根据规则决定（通常点炮者付分）

### 2. 多人同时可以碰/杠
- 如果多人同时可以碰/杠，需要等待响应
- 根据座位顺序决定谁执行

### 3. 一人胡，其他人碰/吃
- 胡优先级最高，选择胡后立即执行
- 取消其他人的选择机会

## 修改的文件

- `server.js` - `playerAction` 函数

## 测试场景

1. **碰 vs 吃**：玩家A碰，玩家B吃 → A选择碰后立即执行
2. **胡 vs 碰**：玩家A胡，玩家B碰 → A选择胡后立即执行
3. **杠 vs 吃**：玩家A杠，玩家B吃 → A选择杠后立即执行
4. **多人胡**：玩家A胡，玩家B胡 → 需要等待两人都响应

## 修复日期

2026-03-04
