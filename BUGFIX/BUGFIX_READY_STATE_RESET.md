# Bug修复：准备后退出房间，再次进入时准备状态未重置

## 问题描述

当玩家在准备房间中点击"准备"后离开房间，再次进入同一房间时：
1. 客户端的准备按钮状态可能不正确
2. 准备状态显示与服务器端不同步

## 问题原因

1. 客户端 `isReady` 变量在 `leaveRoom()` 中被重置为 `false`
2. 但在 `room_joined` 事件处理中没有重置 `isReady`
3. 准备按钮的UI状态没有在加入房间时重置

## 修复方案

### 1. 在 `room_joined` 事件中重置准备状态

```javascript
socket.on('room_joined', (data) => {
    console.log('收到 room_joined 事件，roomCode:', data.roomCode);
    currentRoom = data.roomCode;
    isTestRoom = data.isTestRoom || false;
    isReady = false;  // 【修复】重置准备状态
    // 重置准备按钮UI
    const readyBtn = document.getElementById('readyBtn');
    if (readyBtn) {
        readyBtn.innerHTML = '<i class="fas fa-check"></i> 准备';
        readyBtn.classList.remove('secondary');
    }
    sessionStorage.setItem('mahjong_room_code', data.roomCode);
    sessionStorage.setItem('mahjong_username', username);
    showRoomScreen();
});
```

### 2. 在 `showRoomScreen` 函数中重置准备按钮

```javascript
function showRoomScreen() {
    console.log('showRoomScreen 被调用');
    document.getElementById('lobbyScreen').classList.remove('active');
    document.getElementById('roomScreen').classList.add('active');
    document.getElementById('roomCodeDisplay').textContent = currentRoom;
    
    // 【修复】重置准备状态
    isReady = false;
    const readyBtn = document.getElementById('readyBtn');
    if (readyBtn) {
        readyBtn.innerHTML = '<i class="fas fa-check"></i> 准备';
        readyBtn.classList.remove('secondary');
    }
    
    // 测试房间显示牌谱编辑器
    if (isTestRoom) {
        showDeckEditor();
    } else {
        hideDeckEditor();
    }
}
```

## 修改的文件

- `index.html` - `room_joined` 事件处理 和 `showRoomScreen` 函数

## 测试场景

1. 玩家A加入房间
2. 玩家A点击"准备"
3. 玩家A离开房间
4. 玩家A再次加入同一房间
5. 验证：准备按钮显示"准备"，状态为未准备

## 修复日期

2026-03-04
