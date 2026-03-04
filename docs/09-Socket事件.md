# Socket事件文档

本文档详细描述前后端通信的Socket.IO事件协议。

## 一、事件概述

项目使用 Socket.IO 进行前后端实时通信，所有事件分为两类：
- **客户端 → 服务器事件**: 客户端发送给服务器的请求
- **服务器 → 客户端事件**: 服务器广播给客户端的通知

## 二、连接配置

### 2.1 Socket.IO 配置

```javascript
// 服务器端配置
const io = socketIo(server, {
    cors: {
        origin: "*",
        methods: ["GET", "POST"]
    },
    pingTimeout: 30000,        // 30秒无响应视为断开
    pingInterval: 10000,       // 10秒心跳间隔
    maxHttpBufferSize: 1e6,    // 消息大小限制 1MB
    transports: ['websocket', 'polling'],
    perMessageDeflate: {       // 消息压缩
        threshold: 512
    }
});
```

### 2.2 客户端连接

```javascript
// 前端连接
const socket = io('服务器地址', {
    transports: ['websocket', 'polling']
});
```

---

## 三、客户端 → 服务器事件

### 3.1 房间管理事件

#### create_room - 创建房间

**触发时机**: 玩家点击"创建房间"

**数据格式**:
```javascript
{
    username: string,   // 玩家名称
    avatar: string,     // 头像emoji
    voice: string       // 语音类型: 'female01', 'female02', 'male', 'male02'
}
```

**服务器响应**: `room_created`

---

#### join_room - 加入房间

**触发时机**: 玩家输入房间号加入

**数据格式**:
```javascript
{
    roomCode: string,   // 房间号（6位）
    username: string,   // 玩家名称
    avatar: string,     // 头像emoji
    voice: string       // 语音类型
}
```

**服务器响应**: `room_joined` 或 `join_error`

---

#### leave_room - 离开房间

**触发时机**: 玩家点击"离开房间"

**数据格式**: 无

---

#### toggle_ready - 准备/取消准备

**触发时机**: 玩家点击"准备"按钮

**数据格式**:
```javascript
{
    ready: boolean   // true=准备, false=取消准备
}
```

---

#### join_as_viewer - 以观战者身份加入

**触发时机**: 游戏进行中选择观战

**数据格式**:
```javascript
{
    roomCode: string,
    username: string,
    avatar: string
}
```

---

### 3.2 游戏操作事件

#### draw_tile - 摸牌

**触发时机**: 轮到玩家摸牌时

**数据格式**: 无

**服务器响应**: `tile_drawn` 或 `action_error`

---

#### discard_tile - 出牌

**触发时机**: 玩家选择一张牌打出

**数据格式**:
```javascript
{
    tileId: string   // 要出的牌ID
}
```

**服务器响应**: `tile_discarded`（广播给所有人）

---

#### player_action - 执行动作

**触发时机**: 玩家选择碰/杠/吃/胡/过

**数据格式**:
```javascript
{
    actionType: string,      // 'hu', 'gang', 'peng', 'chi', 'pass', 
                             // 'hu_zimo', 'an_gang', 'jia_gang'
    extraData: {             // 可选
        selectedChiIndex: number,  // 吃牌选项索引
        anGangOptions: array,      // 暗杠选项
        jiaGangOptions: array      // 加杠选项
    }
}
```

---

#### confirm_qiao - 确认敲牌

**触发时机**: 听牌后确认敲牌

**数据格式**: 无

---

#### takeover_ai - 接管AI

**触发时机**: 被AI接管的玩家恢复控制

**数据格式**: 无

---

### 3.3 暂停/解散事件

#### pause_game - 暂停游戏

**触发时机**: 玩家请求暂停

**数据格式**: 无

---

#### cancel_pause - 取消暂停

**触发时机**: 玩家取消暂停

**数据格式**: 无

---

#### request_dissolve - 发起解散

**触发时机**: 玩家请求解散游戏

**数据格式**: 无

---

#### vote_dissolve - 投票解散

**触发时机**: 玩家投票同意/反对解散

**数据格式**:
```javascript
{
    agree: boolean   // true=同意, false=反对
}
```

---

### 3.4 测试房间事件

#### set_custom_deck - 设置自定义牌谱

**触发时机**: 测试房间设置牌谱

**数据格式**:
```javascript
{
    deckOrder: Tile[]   // 牌的顺序数组
}
```

---

## 四、服务器 → 客户端事件

### 4.1 房间状态事件

#### room_created - 房间创建成功

**数据格式**:
```javascript
{
    roomCode: string,      // 房间号
    isTestRoom: boolean    // 是否测试房间
}
```

---

#### room_joined - 加入房间成功

**数据格式**:
```javascript
{
    roomCode: string,
    isTestRoom: boolean
}
```

---

#### room_updated - 房间状态更新

**数据格式**:
```javascript
{
    room: {
        code: string,
        hostId: string,
        gameRunning: boolean,
        players: [{
            id: string,
            username: string,
            avatar: string,
            voice: string,
            seatIndex: number,
            wind: string,
            windName: string,
            ready: boolean,
            isHost: boolean,
            isBot: boolean
        }],
        viewers: [{
            username: string,
            avatar: string
        }]
    }
}
```

---

#### join_error - 加入失败

**数据格式**:
```javascript
{
    message: string   // 错误信息
}
```

---

#### game_in_progress_choice - 游戏进行中选择

**数据格式**:
```javascript
{
    roomCode: string,
    message: string,
    canJoin: boolean   // 是否可以加入游戏
}
```

---

### 4.2 游戏状态事件

#### game_started - 游戏开始

**数据格式**:
```javascript
{
    gameState: GameState,     // 完整游戏状态
    dealerIndex: number,      // 庄家索引
    yourSeat: number,         // 自己的座位
    currentRound: number,     // 当前局数
    totalRounds: number,      // 总局数
    matchScores: number[],    // 累计积分
    isReconnect: boolean      // 是否重连
}
```

---

#### game_state_update - 游戏状态更新

**数据格式**:
```javascript
{
    gameState: GameState   // 当前游戏状态
}
```

---

#### light_update - 轻量级状态更新

**数据格式**:
```javascript
{
    p: [{                    // 玩家简略信息
        s: number,           // seat
        h: number,           // handCount
        d: number,           // discardsCount
        m: number,           // meldsCount
        f: number,           // flowersCount
        o: boolean           // offline
    }],
    c: number,               // currentPlayerIndex
    t: string,               // turnPhase
    r: number                // deckRemaining
}
```

---

### 4.3 摸牌出牌事件

#### tile_drawn - 摸到牌

**数据格式**:
```javascript
{
    tile: Tile   // 摸到的牌
}
```

---

#### tile_discarded - 有人出牌

**数据格式**:
```javascript
{
    playerIndex: number,   // 出牌玩家索引
    tile: Tile,            // 打出的牌
    tileName: string,      // 牌名
    isAutoDiscard: boolean,// 是否自动出牌（可选）
    isAI: boolean          // 是否AI出牌（可选）
}
```

---

#### flower_drawn - 补花

**数据格式**:
```javascript
{
    playerIndex: number,   // 玩家索引
    playerId: string,      // 玩家ID
    flower: Tile,          // 花牌
    flowerName: string,    // 花牌名
    totalFlowers: number   // 总花牌数
}
```

---

#### ai_draw - AI摸牌

**数据格式**:
```javascript
{
    playerIndex: number,
    playerName: string,
    flowerCount: number
}
```

---

### 4.4 动作事件

#### action_available - 可执行动作

**数据格式**:
```javascript
{
    actions: string[],      // 可执行动作列表
    tile: Tile,             // 触发牌
    chiOptions: ChiOption[],// 吃牌选项（可选）
    anGangOptions: array,   // 暗杠选项（可选）
    jiaGangOptions: array   // 加杠选项（可选）
}
```

---

#### action_executed - 动作执行

**数据格式**:
```javascript
{
    playerIndex: number,   // 执行玩家索引
    action: string,        // 动作类型
    tile: Tile,            // 相关牌
    tileName: string       // 牌名
}
```

---

#### action_timeout - 动作超时

**数据格式**: 无

**说明**: 通知客户端隐藏动作按钮

---

#### action_error - 操作错误

**数据格式**:
```javascript
{
    message: string   // 错误信息
}
```

---

### 4.5 敲牌事件

#### ting_and_qiao_prompt - 听牌敲牌提示

**数据格式**:
```javascript
{
    message: string,       // 提示信息
    tingTiles: TingTile[]  // 听牌列表
}
```

---

#### player_qiao - 玩家敲牌

**数据格式**:
```javascript
{
    playerIndex: number,   // 玩家索引
    username: string,      // 玩家名称
    voice: string          // 语音类型
}
```

---

#### qiao_timeout_auto_confirm - 敲牌超时自动确认

**数据格式**: 无

---

### 4.6 回合/比赛事件

#### round_ended - 一局结束

**数据格式**:
```javascript
{
    roundResult: RoundResult,   // 本局结果
    currentRound: number,       // 当前局数
    totalRounds: number,        // 总局数
    matchScores: number[],      // 累计积分
    countdownSeconds: number    // 倒计时秒数
}
```

---

#### match_ended - 比赛结束

**数据格式**:
```javascript
{
    ranking: [{              // 排名
        username: string,
        seatIndex: number,
        totalScore: number,
        isBot: boolean
    }],
    matchScores: number[],   // 最终积分
    roundHistory: RoundResult[],  // 历史记录
    totalRounds: number      // 总局数
}
```

---

#### countdown_update - 倒计时更新

**数据格式**:
```javascript
{
    seconds: number,        // 剩余秒数
    readyStatus: ReadyStatus[]  // 准备状态
}
```

---

#### discard_countdown - 出牌倒计时

**数据格式**:
```javascript
{
    seconds: number   // 倒计时秒数
}
```

---

### 4.7 玩家状态事件

#### your_turn - 轮到你行动

**数据格式**:
```javascript
{
    phase: string,     // 'draw' 或 'discard'
    message: string    // 提示信息
}
```

---

#### auto_discard - 自动出牌

**数据格式**:
```javascript
{
    tile: Tile,
    message: string
}
```

---

#### player_offline - 玩家离线

**数据格式**:
```javascript
{
    username: string,
    seatIndex: number
}
```

---

#### player_reconnected - 玩家重连

**数据格式**:
```javascript
{
    username: string,
    seatIndex: number
}
```

---

#### player_takeover - 玩家接管

**数据格式**:
```javascript
{
    username: string,
    seatIndex: number
}
```

---

### 4.8 暂停/解散事件

#### game_paused - 游戏暂停

**数据格式**:
```javascript
{
    pausedPlayer: string,   // 暂停玩家名称
    pauseTime: number       // 暂停时间戳
}
```

---

#### pause_cancelled - 取消暂停

**数据格式**:
```javascript
{
    cancelledPlayer: string,   // 取消玩家名称
    resumeCountdown: number    // 恢复倒计时
}
```

---

#### pause_resume_countdown - 恢复倒计时

**数据格式**:
```javascript
{
    seconds: number
}
```

---

#### game_resumed - 游戏恢复

**数据格式**: 无

---

#### dissolve_requested - 解散请求

**数据格式**:
```javascript
{
    requester: string   // 发起者名称
}
```

---

#### dissolve_vote_update - 解散投票更新

**数据格式**:
```javascript
{
    voter: string,         // 投票者名称
    agree: boolean,        // 是否同意
    votes: number,         // 同意票数
    totalPlayers: number   // 总玩家数
}
```

---

#### dissolve_rejected - 解散被拒绝

**数据格式**: 无

---

#### game_dissolved - 游戏解散

**数据格式**:
```javascript
{
    matchScores: number[],
    roundHistory: RoundResult[],
    totalRounds: number,
    currentRound: number
}
```

---

### 4.9 观战者事件

#### viewer_joined - 观战者加入

**数据格式**:
```javascript
{
    roomCode: string,
    message: string
}
```

---

### 4.10 其他事件

#### takeover_success - 接管成功

**数据格式**:
```javascript
{
    message: string,
    seatIndex: number
}
```

---

#### ready_status_update - 准备状态更新

**数据格式**:
```javascript
{
    readyStatus: ReadyStatus[],
    countdown: number
}
```

---

#### ai_takeover_status - AI接管状态

**数据格式**:
```javascript
{
    readyStatus: ReadyStatus[]
}
```

---

#### room_auto_dissolve_warning - 自动解散警告

**数据格式**:
```javascript
{
    countdown: number,
    message: string
}
```

---

#### room_auto_dissolve_cancelled - 取消自动解散

**数据格式**:
```javascript
{
    message: string
}
```

---

#### custom_deck_saved - 牌谱保存成功

**数据格式**:
```javascript
{
    message: string,
    deckSize: number
}
```

---

#### custom_deck_error - 牌谱错误

**数据格式**:
```javascript
{
    message: string
}
```

---

## 五、事件流程图

### 5.1 创建房间流程

```
客户端                              服务器
   │                                  │
   │──── create_room ────────────────→│
   │                                  │ 生成房间号
   │                                  │ 创建房间
   │                                  │ 添加玩家
   │←─── room_created ───────────────│
   │                                  │
```

### 5.2 加入房间流程

```
客户端                              服务器
   │                                  │
   │──── join_room ──────────────────→│
   │                                  │ 检查房间
   │                                  │ 检查重连
   │                                  │ 添加玩家
   │←─── room_joined ────────────────│
   │                                  │
   │                        (广播) ←──│ room_updated
   │                                  │
```

### 5.3 游戏流程

```
客户端                              服务器
   │                                  │
   │──── toggle_ready ───────────────→│
   │                                  │
   │                        (广播) ←──│ game_started
   │                                  │
   │←─── your_turn ──────────────────│
   │                                  │
   │──── draw_tile ──────────────────→│
   │←─── tile_drawn ─────────────────│
   │                                  │
   │──── discard_tile ───────────────→│
   │                        (广播) ←──│ tile_discarded
   │                                  │
   │←─── action_available ───────────│ (如果有动作)
   │──── player_action ──────────────→│
   │                        (广播) ←──│ action_executed
   │                                  │
   │                        ... 循环 ...│
   │                                  │
   │                        (广播) ←──│ round_ended
   │                                  │
```

### 5.4 动作处理流程

```
客户端                              服务器
   │                                  │
   │←─── action_available ───────────│
   │     {actions: ['hu', 'gang']}   │
   │                                  │
   │──── player_action ──────────────→│
   │     {actionType: 'hu'}          │
   │                                  │ 解析动作
   │                                  │ 执行胡牌
   │                        (广播) ←──│ round_ended
   │                                  │
```

---

## 六、错误处理

### 6.1 常见错误

| 错误类型 | 事件 | 说明 |
|----------|------|------|
| 房间不存在 | join_error | 房间号错误 |
| 房间已满 | join_error | 4人已满 |
| 不是你的回合 | action_error | 回合验证失败 |
| 没有这张牌 | action_error | 出牌验证失败 |
| 无效动作 | action_error | 动作验证失败 |

### 6.2 断线重连

```
客户端断线                         服务器
   │                                  │
   │─────────────────────────────────→│ 检测断线
   │                                  │ 标记玩家offline
   │                        (广播) ←──│ player_offline
   │                                  │
   │──────── 重连 ──────────────────→│
   │──── join_room ──────────────────→│
   │                                  │ 检测重连
   │                                  │ 恢复状态
   │←─── game_started ───────────────│ (游戏进行中)
   │     {isReconnect: true}          │
   │                        (广播) ←──│ player_reconnected
   │                                  │
```

---

## 七、最佳实践

### 7.1 客户端事件监听

```javascript
// 监听所有事件（调试用）
socket.onAny((event, ...args) => {
    console.log(`收到事件: ${event}`, args);
});

// 监听连接状态
socket.on('connect', () => {
    console.log('已连接');
});

socket.on('disconnect', () => {
    console.log('已断开');
});

socket.on('connect_error', (error) => {
    console.error('连接错误:', error);
});
```

### 7.2 事件发送

```javascript
// 发送事件
socket.emit('join_room', {
    roomCode: 'ABC123',
    username: '玩家1',
    avatar: '👤',
    voice: 'female01'
});

// 带回调的事件
socket.emit('draw_tile', (response) => {
    if (response.error) {
        console.error('摸牌失败:', response.error);
    }
});
```

### 7.3 房间广播

```javascript
// 服务器端广播
room.broadcast('event_name', data);

// 广播给除某人外的所有人
socket.broadcast.emit('event_name', data);

// 广播给特定房间
io.to(roomCode).emit('event_name', data);
```

