# OpenClaw 多设备协同框架 - 技术方案

## 1. 概述

### 1.1 设计目标

- **去中心化**：无固定 Commander，动态选举
- **WebSocket 优先**：轻量、低延迟
- **Relay 中继**：简单解决跨网络 NAT 穿透
- **任务继承**：Commander 失联后任务可传承
- **零门槛接入**：插件化设计，一键接入

### 1.2 核心特性

| 特性 | 实现 |
|------|------|
| 动态 Commander | 谁接任务谁当 Commander |
| 故障恢复 | 投票选举新 Commander |
| 任务传承 | 分布式状态同步 |
| 人类监督 | 全程可见可干预 |

---

## 2. 系统架构

### 2.1 整体架构图

```
┌──────────────────────────────────────────────────────────────────┐
│                         用户 (Commanding Officer)                 │
│                  Matrix / WebSocket / 任意 OpenClaw 通道         │
└─────────────────────────────┬────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                      Agent Network (Overlay)                      │
│                                                                   │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐      │
│  │ Device A│◄──►│ Device B│◄──►│ Device C│◄──►│ Device D│      │
│  │ (Soldier)│    │(Commander) │    │ (Soldier)│    │ (Soldier)│      │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘      │
│       │              │              │              │            │
│       └──────────────┴──────────────┴──────────────┘            │
│                         P2P / Matrix / WebSocket                  │
└─────────────────────────────┬────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                        MinIO / Shared FS                          │
│              (任务状态、配置、知识库同步)                          │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 组件设计

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenClaw Agent Node                       │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐    ┌─────────────────────────────┐   │
│  │   OpenClaw Core │    │  Coordination Plugin         │   │
│  │                 │    │  ┌─────────────────────────┐ │   │
│  │  - Skills       │    │  │   Election Module      │ │   │
│  │  - Tools        │    │  │   - Heartbeat          │ │   │
│  │  - Memory       │    │  │   - Voting             │ │   │
│  │  - Channels     │    │  │   - Leader Election    │ │   │
│  │                 │    │  ├─────────────────────────┤ │   │
│  │                 │    │  │   Task Commander         │ │   │
│  │                 │    │  │   - Distribution       │ │   │
│  │                 │    │  │   - State Sync         │ │   │
│  │                 │    │  │   - Inheritance         │ │   │
│  │                 │    │  ├─────────────────────────┤ │   │
│  │                 │    │  │   Discovery Module     │ │   │
│  │                 │    │  │   - Peer Registry      │ │   │
│  │                 │    │  │   - Health Check       │ │   │
│  └─────────────────┘    │  └─────────────────────────┘ │   │
│                         │  ┌─────────────────────────┐ │   │
│                         │  │   Transport Layer       │ │   │
│                         │  │   - Matrix Protocol     │ │   │
│                         │  │   - WebSocket           │ │   │
│                         │  │   - HTTP Fallback       │ │   │
│                         │  └─────────────────────────┘ │   │
│                         └─────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. 核心机制

### 3.1 节点发现与注册

#### 3.1.1 发现机制

| 方式 | 适用场景 | 复杂度 |
|------|---------|--------|
| 固定列表 | 小规模、已知设备 | 低 |
| mDNS/Bonjour | 局域网自动发现 | 中 |
| Matrix 房间 | 跨网络、互联网 | 中 |
| Consul/etcd | 生产环境 | 高 |

#### 3.1.2 注册流程

```
1. 节点启动 → 读取配置中的 seed nodes / Matrix 服务器
2. 连接传输层 → 建立 WebSocket / Matrix 会话
3. 注册自身信息 → DeviceID, Capabilities, Endpoint
4. 获取邻居列表 → 从已有节点拉取 peer list
5. 心跳广播 → 定期向所有邻居广播存活状态
```

#### 3.1.3 节点元数据

```typescript
interface NodeInfo {
  deviceId: string;          // 唯一标识
  name: string;              // 可读名称，如 "laptop-zhangsan"
  endpoint: string;          // WebSocket/Matrix 连接地址
  capabilities: string[];    // 支持的能力 ["coding", "browser", "file"]
  
  state: "online" | "busy" | "offline";
  lastHeartbeat: number;    // 时间戳
  role: "candidate" | "commander" | "soldier";
  load: number;              // 负载 (0-100)
}
```

### 3.2 动态 Commander 选举

#### 3.2.1 等级制度

| 等级 | 说明 | 选举权重 |
|------|------|---------|
| **S** | 超级节点 - 主力设备，能力最强 | 最高优先级 |
| **A** | 高级节点 - 常用设备，性能较好 | 高优先级 |
| **B** | 中级节点 - 普通设备 | 中优先级 |
| **C** | 初级节点 - 备用设备，性能有限 | 低优先级 |

**等级确定**：
- 手动配置（默认）
- 或根据硬件性能自动评估（CPU/内存）

#### 3.2.2 触发条件

- **任务触发**：用户向某节点发送任务，该节点自动成为 Commander
- **故障触发**：当前 Commander 失联，触发重新选举

#### 3.2.2 选举算法（等级优先 Bully）

```
┌─────────────────────────────────────────────────────────────┐
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 检测到 Commander失联 (心跳超时 N 次)                      │
│                                                             │
│  2. 所有候选节点发起选举                                     │
│     → 按等级排序：S > A > B > C                             │
│     → 等级最高的候选节点发起选举                             │
│     → 向所有在线节点发送 ELECTION                            │
│                                                             │
│  3. 收到 ELECTION 的节点投票                                │
│     → 投票给等级最高的候选人                                │
│     → 向候选人发送 VOTE                                     │
│                                                             │
│  4. 候选人统计票数                                          │
│     → 获得多数票的候选人成为 Commander                      │
│     → 向所有节点发送 ELECTED                                 │
│                                                             │
│  5. 新 Commander 上任                                       │
│     → 读取任务状态 → 继续执行 / 重新分配                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘

```

#### 3.2.3 选举消息格式

```json
{
  "type": "ELECTION",
  "from": "device-b",
  "timestamp": 1710652800000
}
```

```json
{
  "type": "ELECTED",
  "from": "device-b",
  "commanderId": "device-b",
  "term": 42,
  "timestamp": 1710652800100
}
```

### 3.3 心跳与故障检测

#### 3.3.1 心跳机制

```typescript
interface HeartbeatConfig {
  intervalMs: number;      // 心跳间隔，默认 5000ms
  timeoutMs: number;      // 超时时间，默认 15000ms (3 次)
  maxConsecutiveMissed: number;  // 最大连续丢失次数
}
```

#### 3.3.2 状态机

```
          ┌─────────────┐
          │   ONLINE    │
          └──────┬──────┘
                 │ 连续丢失心跳 > 3 次
                 ▼
          ┌─────────────┐
          │  SUSPECT    │ ◄────────────┐
          └──────┬──────┘              │ 收到心跳
                 │ 等待确认           │
                 │                    │
                 ▼                    │
          ┌─────────────┐              │
          │   DEAD      │──────────────┘
          └─────────────┘
                 │
                 │ 重新连接
                 ▼
          ┌─────────────┐
          │  RECOVERED  │
          └─────────────┘
```

### 3.4 任务分发与执行

#### 3.4.1 任务生命周期

```
┌─────────────────────────────────────────────────────────────┐
│                   Task Lifecycle                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  USER ──► [NEW] ──► [ASSIGNED] ──► [RUNNING] ──► [DONE]   │
│                │          │            │            │       │
│                │          │            │            │       │
│                ▼          ▼            ▼            ▼       │
│            Commander    Soldier       Soldier      Commander      │
│            分配任务   确认接收    执行中      汇总结果       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 3.4.2 任务结构

```typescript
interface Task {
  id: string;                    // 任务唯一ID
  type: "one-shot" | "persistent";
  spec: string;                  // 任务描述
  assignee?: string;             // 当前执行者 DeviceID
  status: "pending" | "assigned" | "running" | "completed" | "failed";
  result?: string;               // 执行结果
  logs: TaskLog[];               // 执行日志
  createdAt: number;
  updatedAt: number;
  createdBy: string;             // 创建者 (用户/设备ID)
}

interface TaskLog {
  timestamp: number;
  nodeId: string;
  action: "assigned" | "started" | "progress" | "completed" | "error";
  message: string;
}
```

#### 3.4.3 任务分发策略

| 策略 | 适用场景 | 规则 |
|------|---------|------|
| Random | 测试 | 随机选择 |
| LeastLoaded | 负载均衡 | 选择负载最低的节点 |
| CapabilityMatch | 任务匹配 | 选择具备所需能力的节点 |
| Locality | 低延迟 | 选择网络延迟最低的节点 |

### 3.5 任务传承（故障恢复）

#### 3.5.1 传承触发

当 Commander 检测到自己即将失联（或被用户/其他节点标记）：

```
1. Commander 检查所有 "running" 状态的任务
2. 对每个任务：
   a. 生成任务快照 (状态 + 上下文 + 日志)
   b. 广播 TASK_INHERIT 消息给所有 Soldier
   c. 指定继承者或让候选人自举
3. 关闭自己的 Commander 角色
4. 如果有 Soldier 响应，开始任务传承
```

#### 3.5.2 传承消息

```json
{
  "type": "TASK_INHERIT",
  "taskId": "task-123",
  "from": "device-b",
  "to": "device-c",
  "snapshot": {
    "status": "running",
    "context": {...},
    "progress": "已完成 60%，正在处理...",
    "logs": [...]
  },
  "timestamp": 1710652800000
}
```

### 3.6 人类监督与干预

### 3.6 人类监督与干预

#### 3.6.1 房间 (Room) 机制

所有协作都在**房间**中进行，类比 Matrix 房间：

```
┌─────────────────────────────────────────────────────────────────┐
│                     Room: "Project Alpha"                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  成员:                                                           │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐           │
│  │  King    │  │ Commander │  │ Soldier  │  │ Soldier  │           │
│  │ (用户)   │  │ Node-A  │  │ Node-B  │  │ Node-C  │           │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘           │
│                                                                  │
│  消息流:                                                         │
│                                                                  │
│  [King]    → 发布任务指令                                        │
│  [Commander] → 确认接收、分配任务                                  │
│  [Soldiers] → 执行、汇报进度                                      │
│  [Commander] → 协调、汇总结果                                      │
│  [King]    → 审核、干预、批准                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### 3.6.2 房间类型

| 房间类型 | 成员 | 用途 |
|---------|------|------|
| **Global Room** | 所有节点 + 用户 | 网络广播、节点发现 |
| **Task Room** | 当前 Commander + 执行节点 + 用户 | 特定任务协作 |
| **Private Room** | 用户 + 单节点 | 私密指令、调试 |

#### 3.6.3 国王指令系统

用户（King）在房间中拥有最高权限：

```typescript
interface KingCommand {
  type: "KING_COMMAND";
  command: 
    | "EXECUTE"      // 立即执行
    | "ABORT"        // 终止任务
    | "PAUSE"        // 暂停任务
    | "RESUME"       // 恢复任务
    | "REASSIGN"     // 重新分配任务
    | "INTERVENE"   // 介入干预
    | "OVERRIDE"    // 覆盖决策
    | "APPROVE"      // 批准继续
    | "INFO";        // 询问信息
  target?: string;   // 目标节点/任务 ID
  reason?: string;  // 指令原因
  timestamp: number;
}
```

#### 3.6.4 国王指令示例

```
房间: "Task Room - 登录页面开发"

[User/King]: 帮我写一个登录页面
[Commander]: 收到任务，正在分配...
  → @Soldier-B: 前端 React 登录组件
  → @Soldier-C: 后端登录 API

[Soldier-B]: 开始编写前端...
[Soldier-C]: 开始编写 API...

[User/King]: @Commander 暂停一下，需求有变
[Commander]: 已暂停，等待新指令

[User/King]: 密码改成至少 8 位，要包含特殊字符
[Commander]: 收到，更新需求...
  → @Soldier-B: 前端校验规则更新
  → @Soldier-C: 后端校验规则更新

[User/King]: @Commander 继续
[Commander]: 恢复执行...
```

#### 3.6.5 Commander 向国王请求介入

当 Commander 遇到问题或需要确认时，可以请求国王介入：

```
┌─────────────────────────────────────────────────────────────┐
│                  Commander → King 请求机制                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  请求类型:                                                   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ NEED_DECISION        需要决策                        │   │
│  │   "有两个方案，请国王选择"                           │   │
│  │   Options: [A] [B]                                 │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ NEED_APPROVAL        需要批准                        │   │
│  │   "执行此操作需要国王批准"                          │   │
│  │   Action: 删除生产数据库                            │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ NEED_INFO           需要更多信息                    │   │
│  │   "请提供 XXX 信息"                                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ STUCK                 卡住了                         │   │
│  │   "任务遇到困难，请求帮助"                          │   │
│  │   Reason: API 超时 3 次                             │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ EMERGENCY            紧急情况                       │   │
│  │   "发现严重问题，需要立即处理"                      │   │
│  │   Issue: 内存使用 95%                               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 3.6.6 请求介入消息格式

```typescript
interface KingRequest {
  id: string;
  type: "NEED_DECISION" | "NEED_APPROVAL" | "NEED_INFO" | "STUCK" | "EMERGENCY";
  from: string;           // Commander ID
  taskId?: string;       // 相关任务
  message: string;       // 描述
  options?: string[];    // 选项 (NEED_DECISION)
  context?: any;         // 上下文信息
  timestamp: number;
}

interface KingResponse {
  requestId: string;
  action: "APPROVE" | "DENY" | "ANSWER" | "IGNORE";
  message?: string;
  timestamp: number;
}
```

#### 3.6.7 请求介入示例

```
[Commander]: ⚠️ 需要决策
   选项: 
   [A] 使用方案一，性能更好但成本高
   [B] 使用方案二，成本低但需要重构

[User/King]: 选择 A

[Commander]: 收到，执行方案一

---

[Commander]: ⚠️ 需要批准
   动作: 部署到生产环境
   影响: 1000+ 用户

[User/King]: 批准

[Commander]: 开始部署...

---

[Commander]: ⚠️ 卡住了
   问题: GitHub API 认证失败
   已重试: 3 次

[User/King]: 检查一下 token 是不是过期了

[Commander]: 收到，检查中...
```

#### 3.6.8 实时干预

用户可以随时直接干预：

| 操作 | 效果 |
|------|------|
| `@Node-X 停止` | 指定节点停止当前任务 |
| `@Commander 改用方案 B` | 强制改变决策 |
| `暂停所有任务` | 暂停整个房间的任务 |
| `终止任务 Y` | 强制终止指定任务 |
| `接管任务 Y` | 用户自己执行任务 |

---

## 4. 传输层设计

### 4.1 协议选择：WebSocket 优先

| 协议 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| **WebSocket** | 轻量、低延迟、简单 | 需要穿透/NAT | 默认首选 |
| WebSocket Relay | 简单中继、解决 NAT | 引入中继节点 | 跨网络场景 |
| HTTP | 穿透强 | 非实时 | 消息推送备用 |
| Matrix | 跨网络、IM 集成 | 协议重、延迟高 | 可选集成 |

```
┌─────────────────────────────────────────────────────────────┐
│              WebSocket-first Transport                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   局域网:  Node A ◄───► Node B ◄───► Node C               │
│              (直连，低延迟)                                  │
│                                                             │
│   跨网络:  Node A ◄──► Relay ◄──► Node B                   │
│              (中继服务器，解决 NAT)                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 消息格式

所有消息统一 JSON 格式：

```typescript
interface WSMessage {
  type: MessageType;
  from: string;           // 发送者 deviceId
  to?: string;           // 目标 deviceId (可选，广播时不填)
  payload: any;          // 消息内容
  timestamp: number;
  id: string;            // 消息唯一ID
}

type MessageType = 
  | "HELLO"              // 节点加入
  | "HELLO_ACK"          // 加入确认
  | "HEARTBEAT"          // 心跳
  | "PEER_LIST"          // 邻居列表同步
  | "ELECTION"           // 发起选举
  | "VOTE"               // 投票
  | "ELECTED"        // 选举结果
  | "TASK_NEW"           // 新任务
  | "TASK_ASSIGN"        // 任务分配
  | "TASK_UPDATE"        // 任务状态更新
  | "TASK_INHERIT"       // 任务传承
  | "TASK_RESULT"        // 任务结果
  | "BYE"                // 节点离开
```

### 4.3 连接管理

```typescript
interface ConnectionCommander {
  // 建立连接
  connect(peerUrl: string): Promise<WebSocket>;
  
  // 心跳
  startHeartbeat(intervalMs: number): void;
  
  // 重连
  reconnect(peerUrl: string, maxRetries: number): Promise<void>;
  
  // 消息路由
  route(message: WSMessage): void;
}
```

### 4.4 Relay 服务器（轻量中继）

```typescript
// 最小 relay 实现 (Go/Node.js)
// 只做消息转发，不存储状态

interface RelayServer {
  port: number;
  
  // 客户端注册
  register(clientId: string, ws: WebSocket): void;
  
  // 消息转发
  forward(message: WSMessage): void;
  
  // 客户端断开
  unregister(clientId: string): void;
}
```

### 4.5 NAT 穿透策略

```
连接建立流程:
                    
  1. 尝试直连 (局域网/公网 IP)
        │
        ▼ 失败
  2. 尝试 UPnP/NAT-PMP 端口映射
        │
        ▼ 失败  
  3. 连接 Relay 服务器中继
        │
        ▼ 成功
  4. 尝试 hole punching (STUN)
        │
        ▼ 成功
  5. 切换直连 (降低延迟)
```

### 4.6 配置示例

```json
{
  "transport": "websocket",
  "websocket": {
    "port": 8080,
    "relay": {
      "enabled": true,
      "url": "wss://relay.example.com:8081",
      "fallback": true
    },
    "peers": [
      "ws://192.168.1.100:8080",
      "ws://192.168.1.101:8080"
    ]
  }
}
```

---

## 5. Web 控制台

### 5.1 功能设计

```
┌─────────────────────────────────────────────────────────────────┐
│                    Agent Coordination Console                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   网络状态   │  │   任务管理   │  │   节点管理   │       │
│  │   Network    │  │    Tasks     │  │    Nodes     │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Dashboard                             │   │
│  │                                                          │   │
│  │  在线节点: 4    任务进行中: 2    Commander: Node-B        │   │
│  │                                                          │   │
│  │  [Node-A] ● online   负载: 30%    能力: coding,browser │   │
│  │  [Node-B] ● online   负载: 80%    能力: coding          │   │
│  │  [Node-C] ● busy     负载: 90%    能力: file            │   │
│  │  [Node-D] ○ offline  最后在线: 10分钟前                 │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 页面功能

| 页面 | 功能 |
|------|------|
| **Dashboard** | 网络拓扑、节点状态、任务概览 |
| **Nodes** | 节点列表、详情、负载、能力 |
| **Tasks** | 任务列表、详情、创建、监控 |
| **Logs** | 实时日志、搜索、过滤 |
| **Settings** | 节点配置、Relay 设置 |

### 5.3 技术实现

```
┌─────────────────────────────────────────────────────────────┐
│                    Web Console Architecture                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                   React/Vue App                     │   │
│   │   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ │   │
│   │   │Dashboard│ │ Nodes   │ │ Tasks   │ │ Settings│ │   │
│   │   └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ │   │
│   └────────┼───────────┼───────────┼───────────┼──────┘   │
│            │           │           │           │          │
│   ┌────────▼───────────▼───────────▼───────────▼──────┐   │
│   │                  State Management                   │   │
│   │              (Zustand / Pinia / Context)           │   │
│   └────────────────────────┬───────────────────────────┘   │
│                            │                                 │
│   ┌────────────────────────▼───────────────────────────┐   │
│   │              WebSocket Client                        │   │
│   │         (连接任一节点，获取全网状态)                  │   │
│   └────────────────────────┬───────────────────────────┘   │
│                            │                                 │
└────────────────────────────┼─────────────────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
      ┌─────────────┐               ┌─────────────┐
      │  Node-A     │               │  Node-B    │
      │ (Commander)   │               │  (Soldier)  │
      └─────────────┘               └─────────────┘
```

### 5.4 API 设计

```typescript
// REST API (由任一节点提供)
GET  /api/nodes              // 获取所有节点
GET  /api/nodes/:id          // 获取节点详情
GET  /api/tasks              // 获取所有任务
POST /api/tasks              // 创建新任务
GET  /api/tasks/:id          // 获取任务详情
DELETE /api/tasks/:id        // 终止任务

// WebSocket API
WS   /ws                     // 实时状态更新

// 消息格式
{
  "type": "STATE_UPDATE",
  "payload": {
    "nodes": [...],
    "tasks": [...],
    "commander": "node-b"
  }
}
```

### 5.5 页面路由

```
/                     → Dashboard
/nodes                → 节点列表
/nodes/:id            → 节点详情
/tasks                → 任务列表
/tasks/new            → 创建任务
/tasks/:id            → 任务详情
/logs                 → 日志查看
/settings              → 配置管理
```

### 5.6 启动方式

```bash
# 方式1: 独立服务
cd web-console
npm run dev -- --port 3000

# 方式2: 集成到 Node 进程
# 作为子服务嵌入 coordination plugin

# 方式3: 嵌入式 UI
# 直接在 OpenClaw Canvas 中渲染
```

### 5.7 UI 组件库

推荐使用：
- **shadcn/ui** - 现代、简洁、可定制
- **Tailwind CSS** - 样式
- **Recharts** - 图表
- **React Flow** - 网络拓扑图

### 5.8 实时更新

```typescript
// WebSocket 订阅状态变化
const ws = new WebSocket('ws://localhost:8080/ws');

ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);
  
  switch (msg.type) {
    case 'NODE_UPDATE':
      updateNode(msg.payload);
      break;
    case 'TASK_UPDATE':
      updateTask(msg.payload);
      break;
    case 'ELECTED':
      showNotification(`新 Commander: ${msg.payload.commanderId}`);
      break;
  }
};
```

---

## 6. 安全设计

### 6.1 认证与授权

```
┌─────────────────────────────────────────────────────────────┐
│                     Security Model                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 节点认证                                                 │
│     ├── 首次加入需预共享密钥 / Token                        │
│     └── 每个节点持有唯一 Device Token                        │
│                                                             │
│  2. 通信加密                                                 │
│     ├── Matrix: TLS + m.room.encryption                    │
│     ├── WebSocket: WSS (TLS)                               │
│     └── 消息内容可选端到端加密                               │
│                                                             │
│  3. 权限控制                                                 │
│     ├── Commander: 任务分配、全局状态读写                      │
│     ├── Soldier: 本地状态读写、任务执行                       │
│     └── 用户: 任务发起、结果查看、干预                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 凭证管理

类似 HiClaw 的设计：

- **节点不持有真实 API Key**
- 所有外部调用通过网关
- 凭证集中管理，按需分配

---

## 7. 插件设计 (OpenClaw Integration)

### 7.1 插件 manifest

```json
{
  "id": "agent-coordination",
  "name": "Agent Coordination Network",
  "description": "Multi-device coordination with dynamic commander election",
  "version": "1.0.0",
  "kind": "coordination",
  "configSchema": {
    "type": "object",
    "properties": {
      "transport": {
        "type": "string",
        "enum": ["matrix", "websocket", "hybrid"],
        "default": "matrix"
      },
      "matrixServer": {
        "type": "string"
      },
      "deviceId": {
        "type": "string"
      },
      "deviceName": {
        "type": "string"
      },
      "capabilities": {
        "type": "array",
        "items": { "type": "string" }
      },
      "seedNodes": {
        "type": "array",
        "items": { "type": "string" }
      }
    },
    "required": ["deviceId"]
  }
}
```

### 6.2 配置示例

```json
{
  "plugins": {
    "entries": ["agent-coordination"],
    "config": {
      "agent-coordination": {
        "transport": "websocket",
        "deviceId": "device-laptop-zhangsan",
        "deviceName": "张三的笔记本",
        "capabilities": ["coding", "browser", "file"],
        "websocket": {
          "port": 8080,
          "relay": {
            "enabled": true,
            "url": "wss://relay.example.com:8081"
          },
          "peers": [
            "ws://192.168.1.100:8080",
            "wss://relay.example.com:8081"
          ]
        }
      }
    }
  }
}
```

### 6.3 开发者接入流程

```bash
# 1. 安装插件
npm install @openclaw/agent-coordination

# 2. 配置
# 编辑 openclaw.json 添加插件配置

# 3. 启动
openclaw restart

# 4. 自动加入网络
# 连接到 Matrix 房间，开始发现邻居
```

---

## 8. 部署方案

### 8.1 开发测试（本地）

```bash
# 启动多个 OpenClaw 实例
openclaw --config node-a.json  # 设备 A
openclaw --config node-b.json  # 设备 B
openclaw --config node-c.json  # 设备 C
```

### 8.2 生产部署

```
┌─────────────────────────────────────────────────────────────┐
│                  Production Deployment                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [用户]                                                     │
│    │                                                        │
│    ▼                                                        │
│  ┌─────────────────┐    ┌─────────────────┐                │
│  │ Matrix Server   │    │  MinIO          │                │
│  │ (Tuwunel)       │    │  (Shared FS)    │                │
│  └────────┬────────┘    └────────┬────────┘                │
│           │                      │                         │
│           │  WebSocket / HTTP    │                         │
│           ▼                      ▼                         │
│  ┌─────────────────────────────────────────────┐           │
│  │         Agent Network (任意位置)            │           │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐    │           │
│  │  │ Node A  │  │ Node B  │  │ Node C  │    │           │
│  │  │ (阿里云) │  │ (AWS)   │  │ (本地)  │    │           │
│  │  └─────────┘  └─────────┘  └─────────┘    │           │
│  └─────────────────────────────────────────────┘           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 9. 路线图

### 9.1 Phase 1: 核心框架
- [ ] WebSocket 传输层（直连 + Relay）
- [ ] 节点发现与注册
- [ ] 基本心跳机制
- [ ] 简单任务分发

### 9.2 Phase 2: 选举与故障
- [ ] Bully 选举算法
- [ ] 任务传承机制
- [ ] 状态同步

### 9.3 Phase 3: 生产就绪
- [ ] NAT 穿透 (STUN/TURN)
- [ ] 安全加固 (TLS + 认证)
- [ ] 监控与日志
- [ ] 性能优化

### 9.4 Phase 4: 生态
- [ ] Matrix 协议可选集成
- [ ] 可视化控制台
- [ ] 插件市场

---

## 10. 参考

- HiClaw 架构: https://github.com/alibaba/hiclaw
- Matrix 协议: https://matrix.org/
- Raft 论文: https://raft.github.io/
- Bully 算法: https://en.wikipedia.org/wiki/Bully_algorithm
