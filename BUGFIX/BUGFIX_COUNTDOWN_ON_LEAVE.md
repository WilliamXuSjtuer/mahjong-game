# Bug修复：退出房间后依然在倒计时

## 问题描述

当用户退出房间或离开游戏时，客户端的倒计时计时器没有被正确清除，导致：
1. 退出房间后，倒计时仍在后台运行
2. 可能导致内存泄漏
3. 可能导致重新进入房间后出现多个倒计时

## 影响范围

- `leaveRoom()` 函数 - 离开房间时
- `leaveGame()` 函数 - 离开游戏时
- `returnToHome()` 函数 - 返回主页时

## 涉及的计时器

1. `discardCountdownTimer` - 出牌倒计时
2. `pauseTimerInterval` - 暂停计时器
3. `pingInterval` - 心跳计时器

## 修复方案

### 1. 创建统一的计时器清理函数

在 `index.html` 中添加一个统一的计时器清理函数：

```javascript
// 清除所有客户端计时器
function clearAllClientTimers() {
    // 清除出牌倒计时
    if (discardCountdownTimer) {
        clearInterval(discardCountdownTimer);
        discardCountdownTimer = null;
    }
    
    // 清除暂停计时器
    if (pauseTimerInterval) {
        clearInterval(pauseTimerInterval);
        pauseTimerInterval = null;
    }
    
    // 清除心跳计时器
    if (pingInterval) {
        clearInterval(pingInterval);
        pingInterval = null;
    }
    
    // 隐藏倒计时显示元素
    const countdownEl = document.getElementById('discardCountdown');
    if (countdownEl) {
        countdownEl.style.display = 'none';
    }
    
    // 隐藏下一局倒计时区域
    const nextRoundCountdownEl = document.getElementById('nextRoundCountdown');
    if (nextRoundCountdownEl) {
        nextRoundCountdownEl.style.display = 'none';
    }
    
    console.log('所有客户端计时器已清除');
}
```

### 2. 修改 `leaveRoom` 函数

```javascript
// 离开房间
function leaveRoom() {
    // 清除所有计时器
    clearAllClientTimers();
    
    socket.emit('leave_room');
    currentRoom = null;
    isReady = false;
    isTestRoom = false;
    currentDeckOrder = [];
    hideDeckEditor();
    hidePauseModals();
    sessionStorage.removeItem('mahjong_room_code');
    sessionStorage.removeItem('mahjong_username');
    document.getElementById('roomScreen').classList.remove('active');
    document.getElementById('lobbyScreen').classList.add('active');
}
```

### 3. 修改 `leaveGame` 函数

```javascript
// 离开游戏
function leaveGame() {
    if (confirm('确定要退出游戏吗？')) {
        // 清除所有计时器
        clearAllClientTimers();
        
        socket.emit('leave_room');
        document.getElementById('gameScreen').classList.remove('active');
        document.getElementById('lobbyScreen').classList.add('active');
        document.getElementById('chatArea').style.display = 'none';
        document.body.classList.remove('in-game');
        currentRoom = null;
        gameState = null;
        gameRunning = false;
        isViewer = false;
    }
}
```

### 4. 修改 `returnToHome` 函数

```javascript
// 返回主页
function returnToHome() {
    // 清除所有计时器
    clearAllClientTimers();
    hidePauseModals();
    
    const dissolveResultModalEl = document.getElementById('dissolveResultModal');
    if (dissolveResultModalEl) dissolveResultModalEl.classList.remove('active');
    
    const gameScreenEl = document.getElementById('gameScreen');
    if (gameScreenEl) gameScreenEl.classList.remove('active');
    
    const roomScreenEl = document.getElementById('roomScreen');
    if (roomScreenEl) roomScreenEl.classList.remove('active');
    
    const lobbyScreenEl = document.getElementById('lobbyScreen');
    if (lobbyScreenEl) lobbyScreenEl.classList.add('active');
    
    const chatAreaEl = document.getElementById('chatArea');
    if (chatAreaEl) chatAreaEl.style.display = 'none';
    
    document.body.classList.remove('in-game');
    currentRoom = null;
    gameState = null;
    gameRunning = false;
    isViewer = false;
}
```

## 测试验证

1. 进入房间，开始游戏
2. 等待出牌倒计时出现
3. 点击"离开"按钮退出房间
4. 验证倒计时已停止且不再显示
5. 重新进入房间，验证没有重复的倒计时

## 修复日期

2026-03-04
