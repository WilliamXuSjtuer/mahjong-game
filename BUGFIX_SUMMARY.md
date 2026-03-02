# Bug 修复总结

## 修复日期
2026-03-02

## Bug 列表

### Bug 1: 小红大单吊已听牌但没有显示敲

#### 问题描述
- AI 玩家（小红）已经形成大单吊牌型（手牌只剩 1 张，4 组副露 + 1 对将）
- 应该自动听牌并显示"敲"状态，但没有显示

#### 根本原因
在 `server.js` 的 `aiDiscard` 函数中，听牌检测逻辑有误：
```javascript
// 原代码（错误）
if (!aiPlayer.isTing && !aiPlayer.isQiao && this.canHu(aiPlayer.hand, aiPlayer.melds)) {
    // ...
}
```

问题：
- `canHu()` 函数检测的是**已经胡牌**的情况（需要 14 张牌）
- 但 AI 刚打完牌后只有 13 张牌，所以 `canHu()` 始终返回 `false`
- 导致听牌检测失败

#### 修复方案
使用 `getTingTiles()` 函数进行听牌检测，检查是否再摸任意一张牌就能胡：
```javascript
// 修复后代码
if (!aiPlayer.isTing && !aiPlayer.isQiao) {
    const tingTiles = this.getTingTiles(aiPlayer.hand, aiPlayer.melds);
    if (tingTiles.length > 0) {
        aiPlayer.isTing = true;
        aiPlayer.isQiao = true;
        console.log(`AI ${aiPlayer.username} 听牌！自动敲牌，听牌：${tingTiles.map(t => t.tileName).join(', ')}`);
        
        this.broadcast('player_qiao', {
            playerIndex: aiPlayer.seatIndex,
            username: aiPlayer.username,
            voice: aiPlayer.voice || 'female01'
        });
    }
}
```

#### 修改文件
- `server.js` (第 1999-2018 行)

---

### Bug 2: 包三口关系没有在 UI 上显示

#### 问题描述
- 小红和上家已经形成包三口关系（累计 3 次吃/碰/明杠）
- 但在游戏界面上没有任何提示或标记

#### 修复方案

##### 1. 添加 CSS 样式
在 `index.html` 中添加包三口指示器样式：
```css
.sankou-indicator {
    position: absolute;
    top: -5px;
    right: -5px;
    background: linear-gradient(135deg, #e74c3c, #c0392b);
    color: white;
    border-radius: 50%;
    width: 24px;
    height: 24px;
    font-size: 0.7rem;
    font-weight: bold;
    display: flex;
    align-items: center;
    justify-content: center;
    border: 2px solid white;
    box-shadow: 0 2px 8px rgba(231,76,60,0.5);
    animation: sankouPulse 2s infinite;
    z-index: 10;
}
```

##### 2. 修改 HTML 结构
在所有玩家的头像容器中添加包三口指示器元素：
```html
<div class="player-avatar" id="playerAvatar">
    😊
    <div class="sankou-indicator" id="playerSankou" style="display: none;">3</div>
</div>
```

应用到：
- 玩家自己的头像 (`playerAvatar`)
- 对手 1 的头像 (`opponent1Avatar`)
- 对手 2 的头像 (`opponent2Avatar`)
- 对手 3 的头像 (`opponent3Avatar`)

##### 3. 添加 JavaScript 逻辑
创建 `updateSankouIndicators()` 函数：
```javascript
function updateSankouIndicators() {
    if (!gameState || !gameState.players) return;
    
    const currentPlayer = gameState.players.find(p => p.seatIndex === mySeatIndex);
    if (!currentPlayer) return;
    
    // 更新所有玩家的包三口指示器
    for (let i = 0; i < 4; i++) {
        const player = gameState.players[i];
        if (!player) continue;
        
        // 获取该玩家对当前玩家的三口计数
        const sankouCount = player.sankouCounts ? player.sankouCounts[mySeatIndex] : 0;
        
        // 确定指示器元素 ID
        let indicatorId;
        if (i === mySeatIndex) {
            indicatorId = 'playerSankou';
        } else {
            const relSeat = (i - mySeatIndex + 4) % 4;
            if (relSeat === 1) indicatorId = 'opponent1Sankou';
            else if (relSeat === 2) indicatorId = 'opponent2Sankou';
            else if (relSeat === 3) indicatorId = 'opponent3Sankou';
            else continue;
        }
        
        // 更新指示器显示
        const indicatorEl = document.getElementById(indicatorId);
        if (indicatorEl) {
            if (sankouCount >= 3) {
                indicatorEl.style.display = 'flex';
                indicatorEl.textContent = sankouCount;
            } else {
                indicatorEl.style.display = 'none';
            }
        }
    }
}
```

##### 4. 服务器端数据支持
在 `getPlayerGameState()` 和 `getViewerGameState()` 函数中添加三口计数数据：
```javascript
players: this.players.map(p => ({
    // ... 其他属性
    sankouCounts: p.sankouCounts || [0, 0, 0, 0]  // 【新增】包三口计数
}))
```

##### 5. 集成到 UI 更新流程
在 `_doUpdateGameUI()` 函数中调用：
```javascript
// 【新增】更新包三口指示器
updateSankouIndicators();

updateActionButtons();
updateDiscardArea();
// ...
```

#### 修改文件
- `index.html`:
  - CSS 样式（第 471-503 行）
  - HTML 结构（第 2593-2670 行）
  - JavaScript 函数（第 6092-6134 行）
  - UI 更新调用（第 4960-4966 行）
- `server.js`:
  - `getPlayerGameState()` (第 783 行)
  - `getViewerGameState()` (第 820 行)

---

## 效果展示

### Bug 1 修复效果
- AI 听牌后会自动显示"敲"状态
- 控制台会输出听牌信息：`AI 小红 听牌！自动敲牌，听牌：东风，西风`

### Bug 2 修复效果
- 当玩家 A 与玩家 B 有三口关系（≥3 口）时
- 玩家 B 的头像右上角会出现红色圆形徽章
- 徽章显示累计口数（如 "3"、"4" 等）
- 徽章带有脉冲动画效果，非常醒目

## 测试建议

### 测试场景 1：大单吊听牌
1. AI 玩家形成大单吊牌型（4 组副露 + 1 对将 + 1 张单牌）
2. AI 打出单牌后
3. **预期**：AI 头像旁显示"敲"状态，控制台输出听牌信息

### 测试场景 2：包三口显示
1. 玩家 A 吃/碰/明杠 玩家 B 的牌累计 3 次
2. 观察玩家 B 的头像
3. **预期**：玩家 B 头像右上角出现红色徽章，显示数字"3"

## 注意事项

1. **包三口计数是单向的**：A 吃 B 的牌，只计 A 对 B 的口数
2. **暗杠不计入**：暗杠是自己摸的牌，不计算在三口内
3. **听牌检测性能**：`getTingTiles()` 会遍历所有牌型，但 AI 数量少，性能影响可忽略

## 相关文件

- `server.js` - 服务器端逻辑
- `index.html` - 客户端 UI
- `BAKOUSAN_IMPLEMENTATION.md` - 包三口规则实现文档
- `BAKOUSAN_TEST.md` - 包三口规则测试指南

## 更新日志

- 2026-03-02: 修复两个关键 bug
  - ✅ 修复 AI 听牌检测逻辑
  - ✅ 添加包三口 UI 显示功能
