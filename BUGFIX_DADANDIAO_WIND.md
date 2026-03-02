# Bug 修复：大单吊胡风向时无法敲

## 修复日期
2026-03-02

## 问题描述

玩家形成大单吊牌型时，如果听的是风向牌（东/南/西/北），无法正确检测到听牌，导致不能自动"敲"。

### 具体场景
- 玩家手牌：1 张西风
- 副露：4 组碰牌（12 张）
- 总牌数：13 张
- 听牌：再摸一张西风形成对子

**预期行为**：AI 打出牌后，检测到听牌，自动设置 `isQiao=true` 并显示"敲"

**实际行为**：没有检测到听牌，不显示"敲"

---

## 根本原因

在 `getTingTiles()` 函数中，风向牌的遍历逻辑有误：

### 错误代码
```javascript
const allTileTypes = ['wan', 'tiao', 'tong', 'wind'];

for (const type of allTileTypes) {
    const maxValue = type === 'wind' ? 4 : 9;
    for (let value = 1; value <= maxValue; value++) {
        const testTile = { type, value };  // ❌ 风向牌的 value 应该是字符串！
        // ...
    }
}
```

### 问题分析
- 风向牌的 `value` 是字符串：`'dong'`, `'nan'`, `'xi'`, `'bei'`
- 但代码中使用了数字 `1, 2, 3, 4` 作为 `value`
- 导致创建的测试牌 `{ type: 'wind', value: 1 }` 与实际风向牌不匹配
- `canHu()` 检测时无法正确比对，返回 `false`
- 最终导致听牌检测失败

---

## 修复方案

### 修复后的代码
```javascript
// 获取听牌列表
getTingTiles(hand, melds = []) {
    const tingTiles = [];
    
    // 数字牌：万、条、筒
    for (const type of ['wan', 'tiao', 'tong']) {
        for (let value = 1; value <= 9; value++) {
            const testTile = { type, value };
            const testHand = [...hand, testTile];
            if (this.canHu(testHand, melds)) {
                tingTiles.push({ type, value, tileName: getTileName(testTile) });
            }
        }
    }
    
    // 字牌：风向
    for (const windValue of WINDS) {
        const testTile = { type: 'wind', value: windValue };
        const testHand = [...hand, testTile];
        if (this.canHu(testHand, melds)) {
            tingTiles.push({ type: 'wind', value: windValue, tileName: getTileName(testTile) });
        }
    }
    
    // 箭牌：中发白
    for (const honorValue of ['zhong', 'fa', 'bai']) {
        const testTile = { type: 'honor', value: honorValue };
        const testHand = [...hand, testTile];
        if (this.canHu(testHand, melds)) {
            tingTiles.push({ type: 'honor', value: honorValue, tileName: getTileName(testTile) });
        }
    }
    
    return tingTiles;
}
```

### 修复要点
1. **分开处理**：将数字牌、风向牌、箭牌分开遍历
2. **正确使用 value**：
   - 数字牌：`value = 1-9`（数字）
   - 风向牌：`value = 'dong', 'nan', 'xi', 'bei'`（字符串）
   - 箭牌：`value = 'zhong', 'fa', 'bai'`（字符串）
3. **完整覆盖**：确保所有牌型都能被检测到

---

## 修改文件

- `server.js` (第 2162-2192 行)
  - 函数：`getTingTiles(hand, melds)`

---

## 测试场景

### 场景 1：大单吊胡风向
1. AI 形成大单吊牌型（4 组副露 + 1 张手牌）
2. 手牌为风向牌（如西风）
3. AI 打出手牌
4. **预期**：检测到听牌（胡西风对倒），自动敲牌
5. **控制台输出**：`AI xxx 听牌！自动敲牌，听牌：西风`

### 场景 2：大单吊胡箭牌
1. AI 形成大单吊牌型
2. 手牌为箭牌（如红中）
3. AI 打出手牌
4. **预期**：检测到听牌（胡红中对倒），自动敲牌
5. **控制台输出**：`AI xxx 听牌！自动敲牌，听牌：红中`

### 场景 3：大单吊胡数字牌
1. AI 形成大单吊牌型
2. 手牌为数字牌（如五万）
3. AI 打出手牌
4. **预期**：检测到听牌（胡五万对倒），自动敲牌
5. **控制台输出**：`AI xxx 听牌！自动敲牌，听牌：五万`

---

## 技术说明

### 风向牌数据结构
```javascript
const WINDS = ['dong', 'nan', 'xi', 'bei']; // 东南西北

// 创建风向牌
{ type: 'wind', value: 'xi', id: 'wind_xi_0' }
```

### 箭牌数据结构
```javascript
const HONORS = ['zhong', 'fa', 'bai']; // 中发白

// 创建箭牌
{ type: 'honor', value: 'zhong', id: 'honor_zhong_0' }
```

### 听牌检测原理
1. 遍历所有可能的牌
2. 将每张牌添加到手牌中（形成 14 张）
3. 调用 `canHu()` 检测是否能胡
4. 如果能胡，则该牌是听牌

---

## 影响范围

### 正面影响
- ✅ 修复大单吊胡风向牌时无法敲的 bug
- ✅ 修复大单吊胡箭牌时无法敲的 bug
- ✅ 听牌检测更加准确和完整
- ✅ AI 智能度提升

### 兼容性
- ✅ 不影响现有功能
- ✅ 听牌检测逻辑更加严谨
- ✅ 性能影响可忽略（AI 数量少）

---

## 相关文档

- [`server.js`](file:///f:/QiaoMa/shanghaimahjong/server.js#L2162-L2192) - 修复的听牌检测函数
- [`BUGFIX_SUMMARY.md`](file:///f:/QiaoMa/shanghaimahjong/BUGFIX_SUMMARY.md) - 之前的 bug 修复总结
- [`BAKOUSAN_IMPLEMENTATION.md`](file:///f:/QiaoMa/shanghaimahjong/BAKOUSAN_IMPLEMENTATION.md) - 包三口规则实现

---

## 更新日志

- 2026-03-02: 修复大单吊胡风向牌无法敲的 bug
  - ✅ 修复 `getTingTiles()` 函数中风向牌的遍历逻辑
  - ✅ 添加箭牌的听牌检测
  - ✅ 确保所有牌型都能被正确检测
