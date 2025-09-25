# V2bX-SSPanel集成技术实现细节

## 核心问题解决

### 1. Reality协议端口配置错误
**问题**: V2bX尝试连接30845端口而不是443端口
**原因**: 错误地将监听端口用作Reality目标端口
**解决**: 
```go
// 错误的实现
ServerPort: params["port"],  // 30845

// 正确的实现  
realityPort := params["serverPort"]  // 443
if realityPort == "" {
    realityPort = "443"
}
ServerPort: realityPort,
```

### 2. PHP数组格式数据传输
**问题**: V2bX发送JSON，SSPanel期望PHP数组格式
**解决**: 使用表单数据模拟PHP数组
```go
// 流量上报格式
formData[fmt.Sprintf("data[%d][user_id]", i)] = fmt.Sprintf("%d", userID)
formData[fmt.Sprintf("data[%d][u]", i)] = fmt.Sprintf("%d", upload)
formData[fmt.Sprintf("data[%d][d]", i)] = fmt.Sprintf("%d", download)

// 在线用户格式
formData[fmt.Sprintf("data[%d][user_id]", i)] = fmt.Sprintf("%d", userID)
formData[fmt.Sprintf("data[%d][ip]", i)] = ip
```

### 3. 节点类型检查遗漏
**问题**: 多处代码未包含VLESS节点类型(15,16)
**影响**: 节点显示离线，在线用户数显示N/A
**解决**: 在4个关键位置添加VLESS节点类型
```php
// 原始代码
if (!in_array($sort, [0, 7, 8, 10, 11, 12, 13, 14])) {

// 修复后
if (!in_array($sort, [0, 7, 8, 10, 11, 12, 13, 14, 15, 16])) {
```

## API适配详情

### 1. 节点信息API
```
V2board: GET /api/v1/server/UniProxy/config
SSPanel: GET /mod_mu/nodes/{id}/info?key={token}&node_id={id}

响应格式转换:
V2board: 直接JSON配置
SSPanel: {ret: 1, data: {server: "host;params"}}
```

### 2. 用户列表API  
```
V2board: GET /api/v1/server/UniProxy/user
SSPanel: GET /mod_mu/users?key={token}&node_id={id}

响应格式:
V2board: {users: [...]}
SSPanel: {ret: 1, data: [...]}
```

### 3. 流量上报API
```
V2board: POST /api/v1/server/UniProxy/push
SSPanel: POST /mod_mu/users/traffic

数据格式:
V2board: JSON body {uid: [upload, download]}
SSPanel: Form data[0][user_id]=uid&data[0][u]=upload&data[0][d]=download
```

### 4. 在线用户上报API
```
V2board: POST /api/v1/server/UniProxy/alive  
SSPanel: POST /mod_mu/users/aliveip

数据格式:
V2board: JSON {uid: ["ip1", "ip2"]}
SSPanel: Form data[0][user_id]=uid&data[0][ip]=ip1&data[1][user_id]=uid&data[1][ip]=ip2
```

## VLESS配置解析

### 配置字符串格式
```
server;port=30845&flow=xtls-rprx-vision&security=reality&dest=aws.amazon.com&serverPort=443&serverName=aws.amazon.com&privateKey=xxx&publicKey=xxx&shortId=0123456789abcdef
```

### 解析逻辑
```go
parts := strings.Split(serverConfig, ";")
host := parts[0]  // server
paramStr := parts[1]  // 参数字符串

// 解析参数
params := make(map[string]string)
paramPairs := strings.Split(paramStr, "&")
for _, pair := range paramPairs {
    kv := strings.Split(pair, "=")
    if len(kv) == 2 {
        params[kv[0]] = kv[1]
    }
}

// 构造VLESS节点
vlessNode := &VAllssNode{
    CommonNode: CommonNode{
        Host:       host,
        ServerPort: port,  // 监听端口
    },
    TlsSettings: TlsSettings{
        Dest:       params["dest"],        // aws.amazon.com
        ServerPort: params["serverPort"],  // 443
        ServerName: params["serverName"],  // aws.amazon.com
        // ...其他Reality参数
    },
}
```

## 心跳机制

### 工作原理
1. V2bX每60秒调用用户列表API
2. SSPanel在`/mod_mu/users`接口中更新心跳:
   ```php
   $node->node_heartbeat = time();
   $node->save();
   ```
3. 前端检查在线状态:
   ```php
   if ($node_heartbeat > time() - 300) {
       return true;  // 在线
   }
   ```

### 在线用户数记录
在线用户数通过流量上报API记录:
```php
// 在addTraffic方法中
$online_log = new NodeOnlineLog();
$online_log->node_id = $node_id;
$online_log->online_user = count($data);  // 有流量的用户数
$online_log->log_time = time();
$online_log->save();
```

前端显示:
```php
public function getOnlineUserCount() {
    $log = NodeOnlineLog::where('node_id', $id)
        ->where('log_time', '>', time() - 300)
        ->first();
    return $log ? $log->online_user : 0;
}
```

## 调试日志格式

### 流量上报日志
```
DEBUG: ReportUserTraffic called - Users: 2
DEBUG: User 1 - Upload: 1048576 bytes (1.00 MB), Download: 2097152 bytes (2.00 MB)
DEBUG: User 2 - Upload: 524288 bytes (0.50 MB), Download: 1048576 bytes (1.00 MB)
DEBUG: Total Traffic - Upload: 1572864 bytes (1.50 MB), Download: 3145728 bytes (3.00 MB)
DEBUG: ReportUserTraffic response - Status: 200, Body: {"ret":1,"data":"ok"}
```

### 在线用户上报日志
```
DEBUG: ReportNodeOnlineUsers called - Users: 1, Data: map[1:[183.229.199.66]]
DEBUG: ReportNodeOnlineUsers response - Status: 200, Body: {"ret":1,"data":"ok"}
```

## 编译配置

### 完整功能编译
```bash
$env:GOOS="linux"; $env:GOARCH="amd64"; go build -v -o ./V2bX-linux-amd64-complete -tags "xray sing hysteria2 with_quic with_grpc with_utls with_wireguard with_acme" -trimpath -ldflags "-s -w -buildid="
```

### 必需的构建标签
- `xray`: Xray核心支持
- `sing`: Sing-box核心支持  
- `hysteria2`: Hysteria2协议支持
- `with_quic`: QUIC协议支持
- `with_grpc`: gRPC传输支持
- `with_utls`: uTLS伪装支持
- `with_wireguard`: WireGuard支持
- `with_acme`: ACME证书支持

## 部署验证清单

1. ✅ V2bX成功连接SSPanel API
2. ✅ 获取节点配置无错误
3. ✅ 获取用户列表成功
4. ✅ 流量上报格式正确
5. ✅ 在线用户上报无PHP警告
6. ✅ 前端显示节点在线状态
7. ✅ 前端显示在线用户数
8. ✅ VLESS Reality连接正常
9. ✅ 签到功能无JavaScript错误
