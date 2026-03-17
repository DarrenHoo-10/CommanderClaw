# CommanderClaw MVP 开发计划

## MVP 目标
实现多设备 Agent 协同框架的核心功能：节点发现、Commander 选举、任务分发、King 指令系统

---

## Phase 1: 基础通信层（1-2天）

### 1.1 WebSocket 服务端
- [ ] 基础 WebSocket 服务器（Node.js）
- [ ] 客户端连接/断开处理
- [ ] 消息路由

### 1.2 节点发现
- [ ] 节点注册（deviceId, name, capabilities）
- [ ] 邻居列表同步
- [ ] 节点列表维护

### 1.3 心跳机制
- [ ] 定时心跳
- [ ] 心跳超时检测
- [ ] 节点状态更新（online/busy/offline）

---

## Phase 2: Commander 选举（1天）

### 2.1 选举机制
- [ ] 选举触发检测
- [ ] Bully 选举算法
- [ ] ELECTION / VOTE / ELECTED 消息处理

### 2.2 Commander 角色
- [ ] Commander 状态管理
- [ ] 任务队列管理
- [ ] 心跳广播

---

## Phase 3: 任务系统（1-2天）

### 3.1 任务分发
- [ ] 任务创建（King 发起）
- [ ] 任务分配（Commander 分配给 Soldier）
- [ ] 任务状态跟踪

### 3.2 任务执行
- [ ] 执行结果回传
- [ ] 进度更新
- [ ] 任务完成/失败处理

### 3.3 任务传承
- [ ] 任务快照
- [ ] 传承消息广播
- [ ] 任务恢复

---

## Phase 4: King 指令系统（1天）

### 4.1 指令接收
- [ ] 命令解析
- [ ] 指令分类（EXECUTE/ABORT/PAUSE/RESUME 等）

### 4.2 介入请求
- [ ] NEED_DECISION
- [ ] NEED_APPROVAL
- [ ] NEED_INFO
- [ ] STUCK / EMERGENCY

### 4.3 实时干预
- [ ] 暂停任务
- [ ] 终止任务
- [ ] 重新分配

---

## Phase 5: Web 控制台 MVP（1-2天）

### 5.1 页面开发
- [ ] Command 页面（对话 + 任务列表）
- [ ] Monitor 页面（节点 + 日志）
- [ ] Tab 切换

### 5.2 实时通信
- [ ] WebSocket 连接
- [ ] 状态同步
- [ ] 消息推送

---

## Phase 6: 部署与测试（1天）

### 6.1 本地测试
- [ ] 多节点联调
- [ ] 故障恢复测试
- [ ] 性能测试

### 6.2 文档
- [ ] 快速开始指南
- [ ] API 接口文档

---

## 交付物

| 组件 | 说明 |
|------|------|
| `coordination-server` | Node.js 协调服务 |
| `coordination-client` | 客户端 SDK |
| `web-console` | Web 控制台 |
| `SPEC.md` | 技术规范文档 |

---

## 预估工时
- **总计：7-9 天**
- 核心框架：4-5 天
- 控制台：1-2 天
- 测试部署：1-2 天
