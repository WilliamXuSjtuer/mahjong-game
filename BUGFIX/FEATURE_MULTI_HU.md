# 功能添加：一炮多响

## 功能描述

一炮多响：一张牌打出后，两家或三家同时胡牌，所有胡牌者都算胡，由点炮者一人全额赔付。下一局的庄家变为根据顺位第一个胡牌的人。

## 规则说明

1. **触发条件**：一张牌打出后，多个玩家同时选择"胡"
2. **分数计算**：点炮者向每个胡牌者分别支付全额分数
3. **庄家确定**：下一局庄家为顺位第一个胡牌的人（按座位顺序）

## 实现方案

### 1. 修改 `resolveActions` 函数

当有多个玩家选择"胡"时，收集所有胡牌玩家，而不是只执行一个：

```javascript
resolveActions() {
    // 收集所有选择"胡"的玩家
    const huActions = this.gameState.pendingActions.filter(a => a.action === 'hu');
    
    if (huActions.length > 1) {
        // 一炮多响
        this.executeMultiHu(huActions);
    } else if (huActions.length === 1) {
        // 单人胡牌
        this.executeAction(huActions[0]);
    } else {
        // 原有逻辑...
    }
}
```

### 2. 新增 `executeMultiHu` 函数

处理多人胡牌的结算：

```javascript
executeMultiHu(huActions) {
    const tile = huActions[0].tile;
    const loserIndex = this.gameState.lastDiscardPlayer;
    const isGangShangPao = this.gameState.gangShangPao;
    
    // 按座位顺序排序（确定第一个胡牌的人，作为下一局庄家）
    huActions.sort((a, b) => a.playerIndex - b.playerIndex);
    
    // 收集所有胡牌玩家信息
    const winners = huActions.map(action => {
        const player = this.players[action.playerIndex];
        player.hand.push(tile);
        return {
            playerIndex: action.playerIndex,
            player: player
        };
    });
    
    // 调用多人胡牌结算
    this.endRoundMultiHu(winners, loserIndex, isGangShangPao);
}
```

### 3. 新增 `endRoundMultiHu` 函数

处理多人胡牌的分数结算：

```javascript
endRoundMultiHu(winners, loserIndex, isGangShangPao) {
    // 点炮者向每个胡牌者支付全额分数
    const loser = this.players[loserIndex];
    
    winners.forEach((winner, index) => {
        const isFirstWinner = (index === 0);
        const nextDealer = isFirstWinner ? winner.playerIndex : this.lastWinnerIndex;
        
        // 计算分数...
        // 结算逻辑
    });
    
    // 下一局庄家为第一个胡牌的人
    this.lastWinnerIndex = winners[0].playerIndex;
}
```

## 修改的文件

- `server.js` - `resolveActions`、`executeAction`、新增 `executeMultiHu` 和 `endRoundMultiHu`

## 测试场景

1. 玩家A出牌
2. 玩家B和玩家C都可以胡
3. 两人都选择胡
4. 验证：
   - 玩家A向B和C各支付全额分数
   - 下一局庄家为B（座位顺序靠前）

## 修复日期

2026-03-04
