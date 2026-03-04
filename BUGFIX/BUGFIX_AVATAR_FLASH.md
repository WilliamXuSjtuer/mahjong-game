# Bug 修复：自己头像一直金色闪烁

## 修复日期
2026-03-02

## 问题描述

玩家自己的头像一直显示金色脉冲闪烁效果，即使在不是自己出牌回合时也闪烁。

### 具体表现
- 玩家头像始终有金色边框
- 头像持续脉冲动画效果
- 无法区分是否轮到自己出牌

---

## 根本原因

在 HTML 代码中，玩家头像的 `div` 元素**硬编码**了 `current-turn` 类：

### 错误代码（第 2664 行）
```html
<div class="player-avatar current-turn" id="playerAvatar">
    😊
    <div class="sankou-indicator" id="playerSankou" style="display: none;">3</div>
</div>
```

### 问题分析
- `current-turn` 类会触发脉冲动画（`animation: pulse 1s infinite;`）
- 该类应该由 JavaScript 动态添加/移除
- 但在 HTML 中写死，导致始终显示动画效果

### 相关 CSS
```css
/* 第 438-440 行 */
.player-avatar.current-turn {
    animation: pulse 1s infinite;
}

/* 第 452-455 行 */
@keyframes pulse {
    0%, 100% { box-shadow: 0 0 0 0 rgba(212,175,55,0.7); }
    50% { box-shadow: 0 0 0 10px rgba(212,175,55,0); }
}

/* 第 468-471 行 - 出牌阶段的高亮样式 */
.player-avatar.current-turn-active {
    animation: avatarPulse 1.5s ease-in-out infinite;
    border: 3px solid var(--gold);
}
```

---

## 修复方案

移除 HTML 中硬编码的 `current-turn` 类，让 JavaScript 动态控制：

### 修复后的代码
```html
<!-- 第 2664 行 -->
<div class="player-avatar" id="playerAvatar">
    😊
    <div class="sankou-indicator" id="playerSankou" style="display: none;">3</div>
</div>
```

---

## 修改文件

- `index.html` (第 2664 行)
  - 移除 `player-avatar` 的 `current-turn` 类

---

## 修复效果

### 修复前
| 状态 | 头像效果 | 说明 |
|------|---------|------|
| 非出牌阶段 | ❌ 金色脉冲闪烁 | 错误（始终闪烁） |
| 出牌阶段 | ❌ 金色脉冲闪烁 | 无法区分 |

### 修复后
| 状态 | 头像效果 | 说明 |
|------|---------|------|
| 非出牌阶段 | ✅ 正常显示 | 无动画 |
| 出牌阶段 | ✅ 金色脉冲闪烁 | JavaScript 动态添加 `current-turn-active` 类 |

---

## JavaScript 控制逻辑

### 高亮函数（第 6855-6909 行）
```javascript
function updateCurrentPlayerHighlight() {
    if (!gameState) return;
    
    // 清除所有玩家的高亮效果
    document.querySelectorAll('.player-avatar').forEach(avatar => {
        avatar.classList.remove('current-turn-active');
    });
    
    // 只在出牌阶段高亮
    if (gameState.turnPhase !== 'discard') return;
    
    // ... 座位映射逻辑 ...
    
    if (relSeat === 0) {
        // 轮到底部玩家（自己）
        const playerAvatar = document.getElementById('playerAvatar');
        if (playerAvatar) playerAvatar.classList.add('current-turn-active');
        // ...
    } else {
        // 轮到对手
        const opponentAvatar = document.getElementById(`opponent${relSeat}Avatar`);
        if (opponentAvatar) opponentAvatar.classList.add('current-turn-active');
        // ...
    }
}
```

### 调用时机
- 每次游戏状态更新时调用 `_doUpdateGameUI()`
- `_doUpdateGameUI()` 调用 `updateCurrentPlayerHighlight()`
- 动态添加/移除 `current-turn-active` 类

---

## CSS 动画对比

### `current-turn` 类（旧版，已废弃）
```css
.player-avatar.current-turn {
    animation: pulse 1s infinite;
}

@keyframes pulse {
    0%, 100% { box-shadow: 0 0 0 0 rgba(212,175,55,0.7); }
    50% { box-shadow: 0 0 0 10px rgba(212,175,55,0); }
}
```

### `current-turn-active` 类（新版，动态控制）
```css
.player-avatar.current-turn-active {
    animation: avatarPulse 1.5s ease-in-out infinite;
    border: 3px solid var(--gold);
}

@keyframes avatarPulse {
    0%, 100% { 
        box-shadow: 0 0 0 0 rgba(212,175,55,0.8), 0 0 20px rgba(212,175,55,0.4);
        transform: scale(1);
    }
    50% { 
        box-shadow: 0 0 0 8px rgba(212,175,55,0), 0 0 30px rgba(212,175,55,0.6);
        transform: scale(1.05);
    }
}
```

---

## 测试场景

### 场景 1：非出牌阶段
1. 其他玩家出牌
2. **预期**：自己头像正常显示，无金色边框和动画
3. **实际（修复后）**：✅ 正常显示

### 场景 2：自己出牌阶段
1. 轮到自己出牌
2. **预期**：头像显示金色边框和脉冲动画
3. **实际（修复后）**：✅ 显示金色边框和脉冲动画

### 场景 3：出牌后
1. 自己打出牌后
2. **预期**：金色边框和动画消失
3. **实际（修复后）**：✅ 恢复正常显示

---

## 视觉效果对比

### 修复前
```
[始终闪烁的金色头像] 😊
玩家 642（西）
```

### 修复后
```
非出牌阶段：
[正常头像] 😊
玩家 642（西）

自己出牌阶段：
[✨ 金色脉冲头像 ✨] 😊
玩家 642（西）
```

---

## 相关文件

### 代码文件
- [`index.html`](file:///f:/QiaoMa/shanghaimahjong/index.html#L2664) - 修复的玩家头像 HTML
- [`index.html`](file:///f:/QiaoMa/shanghaimahjong/index.html#L438-L471) - 头像 CSS 样式
- [`index.html`](file:///f:/QiaoMa/shanghaimahjong/index.html#L6855-L6909) - JavaScript 高亮逻辑

### 文档文件
- [`BUGFIX_COMPLETE_SUMMARY.md`](file:///f:/QiaoMa/shanghaimahjong/BUGFIX_COMPLETE_SUMMARY.md) - 今日所有 bug 修复总结

---

## 更新日志

- 2026-03-02: 修复自己头像一直金色闪烁的 bug
  - ✅ 移除 HTML 中硬编码的 `current-turn` 类
  - ✅ 头像高亮由 JavaScript 动态控制
  - ✅ 只在出牌阶段显示金色脉冲效果

---

## 技术说明

### 类名说明
- `current-turn`：旧版类名，已在 HTML 中移除
- `current-turn-active`：新版类名，由 JavaScript 动态添加/移除

### 高亮时机
- **时机**：游戏状态为 `discard` 阶段（出牌阶段）
- **对象**：当前出牌玩家的头像和手牌区域
- **效果**：金色边框 + 脉冲动画 + 手牌高亮

### 为什么之前会一直闪烁？
- HTML 中硬编码了 `current-turn` 类
- CSS 中该类定义了无限循环的脉冲动画
- 没有 JavaScript 逻辑移除该类
- 导致始终显示动画效果

---

**修复完成** ✅
