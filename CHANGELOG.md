# 更新日志

## 2025-10-05 功能改进

### 🎯 主要更新

#### 1. 下载完成后自动停止上传
**问题**: 种子下载完成后会继续上传，消耗带宽和资源
**解决方案**: 
- 在 `server/engine.js` 中，当种子验证完成（finished 事件）时，自动断开所有 peer 连接
- 日志会显示断开的 peer 数量

```javascript
// 下载完成后自动停止上传，断开所有 peer 连接
if (engine.swarm && engine.swarm.wires) {
  console.log('stopping upload for ' + engine.infoHash + ' (' + engine.swarm.wires.length + ' peers)');
  engine.swarm.wires.slice().forEach(function (wire) {
    wire.destroy();
  });
  engine.swarm.pause();
}
```

#### 2. 真正的暂停功能
**问题**: 之前点击暂停按钮只是暂停 peer discovery，已建立的连接仍在上传
**解决方案**:
- 在 `server/socket.js` 的 WebSocket pause 事件中添加断开所有 peer 连接的逻辑
- 在 `server/index.js` 的 REST API pause 端点中添加相同逻辑
- 现在暂停会真正停止所有上传和下载活动

**影响的文件**:
- `server/socket.js` - WebSocket pause 事件处理
- `server/index.js` - POST `/torrents/:infoHash/pause` API

#### 3. 智能 InfoHash 自动添加
**问题**: 访问不存在的 infoHash 会返回 404 Not Found
**新功能**: 
- 当访问种子文件时，如果该 infoHash 尚未添加到下载列表，系统会自动添加
- 支持直接通过 URL 访问种子，无需先手动添加

**新增中间件**: `findOrAddTorrent`
- 自动识别 40 位或 32 位 infoHash
- 等待种子元数据加载完成（最长 30 秒）
- 提供详细的错误信息

**使用新中间件的路由**:
- `GET /torrents/:infoHash` - 获取种子详情
- `GET /torrents/:infoHash/files` - 获取 M3U 播放列表
- `GET /torrents/:infoHash/files/:path` - 流式传输文件
- `GET /torrents/:infoHash/archive` - 下载 ZIP 归档
- `GET /torrents/:infoHash/stats` - 获取统计信息

**使用示例**:
```bash
# 直接通过 infoHash 访问文件（会自动添加下载）
curl http://localhost:9000/torrents/YOUR_INFO_HASH/files

# 直接流式播放（会自动添加下载）
curl http://localhost:9000/torrents/YOUR_INFO_HASH/files/video.mp4
```

---

### 📝 修改的文件

1. **server/engine.js**
   - 添加下载完成后自动停止上传的逻辑

2. **server/socket.js**
   - 改进 pause 事件处理，真正断开所有连接

3. **server/index.js**
   - 添加 `findOrAddTorrent` 中间件
   - 改进 REST API pause 端点
   - 将文件访问相关路由改为自动添加模式

---

### 🧪 测试建议

#### 测试 1: 下载完成自动停止
1. 添加一个小种子
2. 等待下载完成
3. 检查控制台日志，应该看到 "stopping upload for..." 消息
4. 检查统计信息，peer 数量应该变为 0

#### 测试 2: 暂停功能
1. 添加种子并开始下载
2. 点击暂停按钮
3. 检查控制台日志，应该看到 "disconnecting X peers" 消息
4. 检查网络流量，上传和下载应该完全停止

#### 测试 3: 自动添加功能
1. 获取一个有效的 infoHash（40位或32位）
2. 直接访问 `http://localhost:9000/torrents/{infoHash}/files`
3. 应该看到种子自动添加到下载列表
4. 稍等片刻后应该返回文件列表（M3U格式）

---

### ⚠️ 注意事项

1. **自动添加功能**
   - 仅对 GET 请求有效（读取操作）
   - POST/DELETE 操作仍需要种子已存在
   - 超时时间设为 30 秒，大种子可能需要更长时间获取元数据

2. **停止上传**
   - 断开连接后，种子仍在列表中
   - 如需重新连接，点击恢复按钮
   - 完全删除种子请使用删除功能

3. **兼容性**
   - 所有修改向后兼容
   - 不影响现有 API 使用方式
   - 前端 UI 无需修改

---

### 🔄 未来改进建议

1. 添加配置选项控制是否自动停止上传
2. 允许用户设置上传速度限制而不是完全停止
3. 添加 UI 指示器显示种子的自动添加状态
4. 支持自定义等待超时时间
