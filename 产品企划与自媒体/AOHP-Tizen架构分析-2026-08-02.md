# AOHP → Tizen：将 Agent 作为 OS 一等公民的架构迁移分析
**日期：** 2026-08-02
**作者：** K豆 🫘

---

## AOHP 的四个设计原则 → Tizen 的映射

### 原则 1：Agent 是一等公民，不是应用层插件

**AOHP 做了什么：** 在 Android AOSP 内核层为 Agent 开辟了独立的执行上下文——Agent 不是"一个 App 调用另一个 App"，而是和进程管理器、文件系统同级的基础设施。

**Tizen 现状：** Bixby 是一个 Tizen App。Vision AI 是芯片+中间件的能力。SmartThings 是另一个 App。三者互不打通。

**Tizen 应该怎么做：**

```
当前 Tizen 架构：
┌────────────────────────────────┐
│  Tizen App Layer               │
│  ┌──────┐ ┌──────┐ ┌────────┐ │
│  │Bixby │ │Smart │ │Netflix │ │  ← Agent 在 App 层
│  │ App  │ │Things│ │  App   │ │     无法跨 App 调度
│  └──────┘ └──────┘ └────────┘ │
├────────────────────────────────┤
│  Tizen Middleware              │
├────────────────────────────────┤
│  Linux Kernel + NQ4 NPU Driver │
└────────────────────────────────┘

AOHP 启发的 Tizen 未来架构：
┌────────────────────────────────┐
│  Tizen App Layer               │
│  ┌──────┐ ┌──────┐ ┌────────┐ │
│  │Bixby │ │Smart │ │Netflix │ │
│  │ UI   │ │Things│ │  UI    │ │
│  └──┬───┘ └──┬───┘ └───┬────┘ │
│     │        │         │       │
├─────┼────────┼─────────┼───────┤
│  ┌──┴────────┴─────────┴──────┐│
│  │   Tizen Agent Runtime      ││  ← NEW：OS 级 Agent 运行时
│  │   ┌────────┐ ┌──────────┐  ││
│  │   │Agent   │ │Agent     │  ││
│  │   │Scheduler│ │Memory    │  ││
│  │   └────────┘ └──────────┘  ││
│  │   ┌────────┐ ┌──────────┐  ││
│  │   │Knox    │ │App Intent│  ││
│  │   │Agent   │ │Bridge    │  ││
│  │   │Guard   │ └──────────┘  ││
│  │   └────────┘               ││
│  └────────────────────────────┘│
├────────────────────────────────┤
│  Linux Kernel + NQ4 NPU Driver │
└────────────────────────────────┘
```

**关键决策：** Tizen Agent Runtime 必须在 OS 中间件层，不在 App 层。这样：
- Agent 可以调度所有 App 的能力（不是"一个 App 调另一个 App"）
- Agent 有独立的 NPU 时间片（不被 App 线程抢占）
- Knox AgentGuard 在 OS 层做安全拦截（不是 App 层事后审计）

---

### 原则 2：OS 级个性化服务组合

**AOHP 做了什么：** Agent 可以动态发现和组合多个 App 的功能——"我要订外卖"→ Agent 自动发现地图 App 获取位置、外卖 App 下单、支付 App 付款、日历 App 添加提醒。不需要用户手动在不同 App 之间切换。

**Tizen 应该怎么做：** 这就是上次讨论的"App Entity + App Intent"模式，但更底层。

```
Tizen App Entity Schema（OS 层注册）：
┌────────────────────────────────────────────────────┐
│  Netflix App Entity                                │
│  ├── Entity Type: VideoContent                     │
│  ├── Actions: Play, Pause, Search, Resume          │
│  ├── Queries: "last watched", "genre:X", "kid-safe"│
│  └── Permissions: child_lock_required              │
│                                                     │
│  YouTube App Entity                                │
│  ├── Entity Type: VideoContent                     │
│  ├── Actions: Play, Pause, Search                  │
│  └── Queries: "subscribed channels", "trending"    │
│                                                     │
│  SmartThings App Entity                            │
│  ├── Entity Type: DeviceControl                    │
│  ├── Actions: On/Off, Dim, Temperature, Lock       │
│  └── Entities: lights[3], thermostat, doorlock     │
└────────────────────────────────────────────────────┘

Tizen Agent Runtime 自动组合：
用户："蜜豆要看汪汪队"
  → Agent 发现 Netflix + YouTube 都有"汪汪队"
  → 检查 Kid Profile → 自动选 Netflix（有儿童锁）
  → 同时调用 SmartThings → 灯光调暗 20%
  → 一个 OS 级调用完成，不需要 3 个 App 之间跳转
```

---

### 原则 3：OS 级高效 Agent 接口

**AOHP 的核心发现：** 当 Agent 调用走 App 层 API 时，每次跨 App 调用都有 IPC 开销 + UI 线程阻塞。OS 级 Agent 接口直接跳过这些，延迟降低一个数量级。

**Tizen 的机会：** 三星控制整个硬件栈——从 NPU 到 OS 到 App 框架。这意味可以做到比 AOHP 更激进的优化：

| 调用路径 | 当前 Tizen（App 层） | AOHP 启发（OS 层） | 三星独有优化 |
|---------|---------------------|-------------------|-------------|
| Agent → 视频播放 | Bixby App → Netflix API（IPC ×2） | Agent Runtime → Entity Bridge → Netflix（IPC ×1） | **Agent Runtime 直通 NQ4 NPU → 跳过 CPU → 零 IPC** |
| Agent → 设备控制 | Bixby → SmartThings Cloud → 设备（云端往返） | Agent Runtime → Matter Bridge → 本地设备 | **本地 NPU + Matter Thread = 完全离线可用** |
| Agent → 内容搜索 | 逐 App 查询（O(n)） | Agent Runtime 并行索引查询（O(1)） | **NQ4 NPU 语义索引 → 毫秒级跨App搜索** |

---

### 原则 4：OS 级安全信息流

**AOHP 做了什么：** Agent 之间的数据交换不是自由的——OS 层定义了信息流的权限边界。哪个 Agent 可以访问哪个 Entity 的数据，是在 OS 层声明和强制执行的。

**Tizen 应该怎么做：** 这直接对应 Knox AgentGuard 的设计：

```
Knox AgentGuard 安全架构：
┌────────────────────────────────────────┐
│  Agent Runtime Security Boundary       │
│                                         │
│  ┌──────────────────────────────────┐  │
│  │  Agent Identity Certificate      │  │  ← 每个 Agent 有独立身份
│  │  (Knox HSM 签发，不可伪造)        │  │
│  └──────────────────────────────────┘  │
│              │                          │
│  ┌───────────▼──────────────────────┐  │
│  │  Permission Manifest             │  │  ← Agent 能力白名单
│  │  ├── allowed_entities: [...]     │  │
│  │  ├── allowed_actions: [...]      │  │
│  │  ├── max_spend: $0               │  │
│  │  └── network: local_only         │  │
│  └──────────────────────────────────┘  │
│              │                          │
│  ┌───────────▼──────────────────────┐  │
│  │  Real-time Circuit Breaker       │  │  ← 行为异常检测+熔断
│  │  ├── rate_limit: 10 calls/sec    │  │
│  │  ├── anomaly_detection: ML model │  │
│  │  └── auto_revoke: 3 violations   │  │
│  └──────────────────────────────────┘  │
│              │                          │
│  ┌───────────▼──────────────────────┐  │
│  │  Audit Trail (Knox TEE 存储)     │  │  ← 不可篡改
│  │  AgentX @ 14:23:15 → Read        │  │
│  │  Netflix.watch_history           │  │
│  │  AgentX @ 14:23:16 → Dim         │  │
│  │  SmartThings.living_room_lights  │  │
│  └──────────────────────────────────┘  │
└────────────────────────────────────────┘
```

---

## 跟之前讨论的三种架构对比：AOHP 填补了什么

| 架构 | 定位 | 三星能用的 |
|------|------|-----------|
| Google Gemini Spark | 云端 Agent 运行时 | ❌ 不适用（全云端） |
| Apple App Entity+Siri | App 层 Agent 调度框架 | ✅ 引入 App Entity 模式到 Tizen |
| AgentOS（概念） | 多 Agent 去中心化协作 | ✅ 引入 A2A 协议到多设备 |
| **AOHP（新增）** | **OS 层 Agent 一等公民** | ✅ **Tizen Agent Runtime 的 OS 层架构参考** |

AOHP 填补了之前缺的一环：**Agent 不应该在应用层，应该在 OS 内核层。** 这是三星独有的优势——Google 控制不了 Android OEM 的内核层（各家各自改），Apple 的 OS 层不对外开放。只有三星这种"全栈自研 OS + 芯片 + 设备"的厂商，能实现 AOHP 级别的深度 Agent 集成。

---

## 短期可落地建议

| 阶段 | 做什么 | 参考 AOHP 哪部分 |
|------|--------|-----------------|
| **Phase 1（6个月）** | Tizen App Entity Schema + App Intent Bridge | 原则 2：服务组合 |
| **Phase 2（12个月）** | Tizen Agent Runtime（OS 中间件层原型） | 原则 1+3：一等公民 + 高效接口 |
| **Phase 3（18个月）** | Knox AgentGuard 集成（OS 层安全） | 原则 4：安全信息流 |
| **Phase 4（24个月）** | 跨设备 Agent Mesh（TV+手机+家电） | A2A 协议 + OS 层 Agent 身份 |

**核心建议：** Phase 1 可以在当前 Tizen 版本上做（App Entity 不需要动 OS 内核），快速验证价值。Phase 2 需要 Tizen 内核团队配合，但从 AOHP 的经验看，OS 层改造带来的性能提升（Token -51%）和安全优势（OS 级权限）是应用层无法替代的。

---

## 参考

- AOHP arXiv:2606.23449, "An Open-Source OS-Level Agent Harness for Personalized, Efficient and Secure Interaction", Zhao et al., THU-AIR, 2026
- GitHub: aohp-os/aohp
- Android 16 QPR2 AOSP base
