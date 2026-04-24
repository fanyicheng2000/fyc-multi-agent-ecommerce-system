# 🚀 Java 多 Agent 电商系统 — 快速学习路线

> **时间投入**：8-12 小时（含代码实践）
> **难度等级**：⭐⭐⭐⭐（有 Java + Spring Boot 基础即可快速上手）
> **推荐学习顺序**：**按本文档从上到下依次学习**

---

## 📚 第一部分：核心概念预热（1-2 小时）

### Step 1.1：理解什么是 Multi-Agent 系统？

**问题**：为什么不用一个超级大的 AI 模型，而要分成多个小的 Agent？

**答案**：

```
单 Agent（普通做法）          Multi-Agent（本项目做法）
┌──────────────────┐         ┌─────────────────────┐
│   超级大 Agent    │         │  用户画像 Agent      │  ← 专注用户分析
│ 需要知道：        │         │  商品推荐 Agent      │  ← 专注产品召回
│ - 用户行为        │         │  营销文案 Agent      │  ← 专注文案生成
│ - 商品库存        │         │  库存决策 Agent      │  ← 专注库存校验
│ - 推荐算法        │         └─────────────────────┘
│ - 库存系统        │
│ - 营销文案生成    │         优点：
│ 缺点：            │         ✅ 上下文不膨胀（Token少）
│ ❌ 上下文膨胀     │         ✅ 可以并行执行（更快）
│ ❌ 推理变慢       │         ✅ 可独立升级（灵活）
└──────────────────┘         ✅ 容错能力强（一个失败不影响其他）
```

**学习资源**：阅读项目根目录下的 `README.md` 中的 "这个项目是什么？" 部分。

---

### Step 1.2：系统架构三视图

#### 视图 1：端到端流程（用户视角）

```
用户请求: 推荐5件商品
    ↓
系统执行
    ↓
返回: [iPhone, AirPods, 充电宝, 屏幕膜, 手机壳] + 个性化文案 + 实验分组
```

#### 视图 2：内部编排流程（Supervisor 视角）

```
Supervisor
├─ Phase 1（并行 ↔ 互不等待）
│  ├─ 用户画像 Agent → 分析用户是 VIP/新客/价格敏感
│  └─ 商品推荐 Agent → 从数百万商品中召回100件候选
├─ Phase 2（并行）
│  ├─ LLM 重排 Agent → 从100件精排到10件（按相关性）
│  └─ 库存决策 Agent → 检查10件中哪些有货
├─ Phase 3（串行 ← 依赖前两步结果）
│  └─ 营销文案 Agent → 为有货商品生成个性化文案
└─ A/B 测试引擎 → 记录用户点击，动态调整策略
```

#### 视图 3：Java 架构（代码结构视角）

```java
RecommendationController（REST API 入口）
    ↓
SupervisorOrchestrator（编排核心）
    ├─ UserProfileAgent     // ① 用户画像
    ├─ ProductRecAgent      // ② 商品推荐
    ├─ InventoryAgent       // ③ 库存决策
    └─ MarketingCopyAgent   // ④ 营销文案
    ↓
ABTestService             // 实验分组
```

---

### Step 1.3：核心概念 Checklist

学完本节后，你应该能回答这些问题：

- [ ] **什么是 Supervisor 模式？** — 一个中心控制器分发任务给多个 Agent，汇总结果
- [ ] **为什么 Phase1 和 Phase2 可以并行？** — 因为它们之间没有数据依赖
- [ ] **为什么 Phase3 必须串行？** — 因为它需要 Phase2 的结果（有货商品列表）
- [ ] **CompletableFuture 的作用是什么？** — 在 Java 中实现异步并行任务
- [ ] **RFM 模型是什么？** — Recency(最近) × Frequency(频率) × Monetary(消费)，用于用户分群

---

## 📂 第二部分：项目结构导航（0.5 小时）

### 目录树速览

```
src/main/java/com/ecommerce/
├── agent/                          # 🤖 4个Agent核心实现
│   ├── BaseAgent.java              # ⭐ 所有Agent的基类（重试+降级+超时）
│   ├── UserProfileAgent.java       # ⭐ 用户画像Agent
│   ├── ProductRecAgent.java        # ⭐ 商品推荐Agent
│   ├── InventoryAgent.java         # ⭐ 库存决策Agent
│   └── MarketingCopyAgent.java     # ⭐ 营销文案Agent
│
├── orchestrator/
│   └── SupervisorOrchestrator.java # 🎯 编排核心（CompletableFuture并行）
│
├── model/                          # 📦 数据模型
│   ├── RecommendationRequest.java  # 用户请求
│   ├── RecommendationResponse.java # 推荐结果
│   ├── UserProfile.java            # 用户画像
│   ├── Product.java                # 商品数据
│   └── AgentResult.java            # Agent执行结果
│
├── service/
│   └── ABTestService.java          # A/B测试分桶引擎
│
├── config/
│   ├── LLMConfig.java              # LLM(大模型)配置
│   └── RecommendationController.java# REST API 端点
│
└── MultiAgentApplication.java      # 🚀 Spring Boot 启动类
```

### 3 分钟快速定位表

| 想要做的事 | 去哪个文件 | 学习优先级 |
|---------|---------|---------|
| 看推荐接口怎么调用的 | `RecommendationController` | 🔴 必看 |
| 理解 Supervisor 怎么协调的 | `SupervisorOrchestrator` | 🔴 必看 |
| 学习如何实现 Agent 基类 | `BaseAgent` | 🟡 应看 |
| 学习用户画像怎么算的 | `UserProfileAgent` | 🟡 应看 |
| 学习商品怎么精排的 | `ProductRecAgent` | 🟡 应看 |
| 了解 A/B 测试原理 | `ABTestService` | 🟢 可看 |

---

## 🎯 第三部分：深度学习路线（6-10 小时）

### 学习路线图（按推荐顺序）

```
Day 1 上午: 代码导读
├─ 10分钟: 看 MultiAgentApplication.java 和项目结构
├─ 20分钟: 看 RecommendationController，理解 API 怎么调用的
└─ 30分钟: 理解 RecommendationRequest / Response 数据结构

Day 1 下午: 编排逻辑
├─ 40分钟: 精读 SupervisorOrchestrator.java
│          ✅ 理解 CompletableFuture 怎么并行的
│          ✅ 理解库存过滤逻辑
│          ✅ 理解实验分组怎么流转的
└─ 20分钟: 对比 Python 版本（README 中有代码示例）

Day 2 上午: Agent 基类
├─ 30分钟: 精读 BaseAgent.java
│          ✅ 理解 runAsync() 公开接口设计
│          ✅ 理解指数退避重试机制
│          ✅ 理解超时控制和降级（Fallback）
└─ 30分钟: 画出调用链路图

Day 2 下午: 4 个 Agent 逐一深钻
├─ 20分钟: UserProfileAgent — 用户分群算法（RFM模型）
├─ 25分钟: ProductRecAgent — LLM 精排 Prompt 设计
├─ 20分钟: InventoryAgent — 库存查询和限购策略
└─ 20分钟: MarketingCopyAgent — 文案模板和合规校验

Day 3: 系统运行 & 实验
├─ 1小时: 本地启动项目，测试推荐接口
├─ 30分钟: ABTestService — 理解流量分桶逻辑
└─ 30分钟: 修改代码，做一个小实验（如改变 Agent 超时时间观察变化）
```

---

## 📖 详细学习指南

### 🟢 第 1 层：API 端点（必读）

**文件**：`config/RecommendationController.java`

**学习目标**：
- REST API 的请求入参和响应结构
- 如何从 HTTP 请求流转到 Supervisor

**关键代码示例**：
```java
@PostMapping("/recommend")
public ResponseEntity<RecommendationResponse> recommend(
    @RequestBody RecommendationRequest request) {
    // 1️⃣ 参数验证
    // 2️⃣ 调用 Supervisor
    // 3️⃣ 返回结果
}
```

**面试话术**：
> "请求进来后先经过 Controller 验证参数，然后交给 Supervisor 编排器，Supervisor 会根据请求并行执行多个 Agent..."

---

### 🟡 第 2 层：Supervisor 编排器（核心，必精读）

**文件**：`orchestrator/SupervisorOrchestrator.java`

**学习目标**：
- 理解三阶段编排（Phase 1 / 2 / 3）
- CompletableFuture 如何实现并行
- 库存过滤的具体逻辑
- 实验分组如何流转

**核心代码解读**：

```java
// ④ Phase 1：用户画像 + 商品召回 【并行】
CompletableFuture<AgentResult> profileFuture = userProfileAgent.runAsync(...);
CompletableFuture<AgentResult> recFuture = productRecAgent.runAsync(...);

AgentResult profileResult = profileFuture.join();  // 等两个都完成
AgentResult recResult = recFuture.join();

// 关键！join() 不会让 CPU 空转，会一直等到 Future 完成
// 总耗时 ≈ max(userProfileAgent耗时, productRecAgent耗时)
// 而不是两者相加！

// ④ Phase 2：LLM 重排 + 库存校验 【并行】
CompletableFuture<AgentResult> rerankFuture = productRecAgent.runAsync(...);
CompletableFuture<AgentResult> inventoryFuture = inventoryAgent.runAsync(...);

AgentResult rerankResult = rerankFuture.join();
AgentResult inventoryResult = inventoryFuture.join();

// ④ Phase 3：文案生成 【串行】
// 因为需要用前面的结果
AgentResult copyResult = marketingCopyAgent.runAsync(...).join();
```

**关键问题解答**：

Q: 为什么要用 CompletableFuture 而不是直接调用？
A:
```java
// ❌ 串行调用（慢）
userProfileAgent.run(...);  // 等待 3s
productRecAgent.run(...);   // 再等待 5s
// 总耗时 = 3 + 5 = 8s

// ✅ 并行调用（快）
CompletableFuture.allOf(
    userProfileAgent.runAsync(...),  // 立刻开始，3s 后完成
    productRecAgent.runAsync(...)    // 立刻开始，5s 后完成
).join();  // 总耗时 = max(3, 5) = 5s
```

**练习题**：
- [ ] 改代码，把 Phase1 改成串行，看推荐延迟如何变化
- [ ] 改代码，让某个 Agent 的超时时间特别短，观察降级逻辑

---

### 🟡 第 3 层：基类与容错设计

**文件**：`agent/BaseAgent.java`

**学习目标**：
- 模板方法模式的应用
- 重试机制（指数退避）
- 降级（Fallback）设计
- 超时控制

**核心模式**：
```java
// BaseAgent 定义了所有 Agent 的标准流程
public abstract class BaseAgent {

    // 公开方法 — 封装了计时、重试、降级
    public final AgentResult runSync(...) {
        try {
            return execute(...);  // 子类实现
        } catch (Exception e) {
            return fallback(...);  // 降级
        }
    }

    // 异步方法 — 返回 CompletableFuture
    public CompletableFuture<AgentResult> runAsync(...) {
        return CompletableFuture.supplyAsync(
            () -> this.runSync(...)
        );
    }
}
```

**降级设计的价值**：

```
场景：LLM 服务暂时不可用
┌─────────────────────────────────┐
│ 传统做法：直接抛异常             │
│ 结果：整个推荐请求失败 ❌         │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│ 本项目做法：降级返回默认结果      │
│ 比如 UserProfileAgent 失败       │
│ → 返回"默认用户画像"（新客）      │
│ → 系统继续执行，推荐请求成功 ✅   │
└─────────────────────────────────┘
```

---

### 🟡 第 4 层：4 个 Agent 的实现

#### Agent 1：UserProfileAgent（用户画像）

**文件**：`agent/UserProfileAgent.java`

**功能**：分析用户属性，分配到 5 个分群之一

```java
// 输入: userId
// 过程:
// ① 从 Redis 获取用户行为特征（点击次数、购买频率）
// ② 调用 LLM 进行用户分析
// ③ 输出 UserProfile 对象

// 输出: {
//   "userId": "user_001",
//   "segment": "price_sensitive",  // 价格敏感型
//   "rfmScore": 0.65,              // RFM 评分
//   "behaviors": [...]
// }
```

**学习重点**：
- RFM 模型三维度的计算
- 5 个分群的定义（新客、VIP、活跃、流失、价格敏感）
- 如何从原始行为数据映射到结构化画像

---

#### Agent 2：ProductRecAgent（商品推荐）

**文件**：`agent/ProductRecAgent.java`

**功能**：两阶段推荐（召回 → 精排）

```
多路召回策略
├─ 协同过滤（用户的购买相似度）
├─ 向量检索（商品语义相似度）
├─ 热度策略（最近热卖商品）
└─ 新品策略（最近上架）
    ↓ 去重合并到 100 件候选
LLM 精排
├─ Prompt: "用户是价格敏感型，偏好手机配件，以下100件商品请排序..."
└─ 输出: 按相关性排序的商品 ID 列表
    ↓
Top-N 商品（通常 10-20 件）
```

**学习重点**：
- 召回策略的多样性（为什么要 4 路？）
- Prompt 工程（如何写出让 LLM 精排效果好的 Prompt）
- 排序算法

**面试话术**：
> "商品推荐分两步：第一步多路召回，用协同过滤、向量检索等不同策略各取一部分商品，这样能覆盖长尾商品，避免算法单调；第二步用 LLM 精排，把用户画像和商品属性结合，让模型理解'这个用户为什么会喜欢这件商品'，精排结果会更符合个性化需求。"

---

#### Agent 3：InventoryAgent（库存决策）

**文件**：`agent/InventoryAgent.java`

**功能**：查询实时库存，过滤缺货商品，输出限购策略

```java
// 输入: [P001, P002, P003, ...] （推荐商品列表）
// 过程:
// ① 查询 MySQL/WMS 实时库存
// ② 过滤缺货商品
// ③ 计算限购策略（库存紧张时每人只能买 1-2 件）

// 输出: {
//   "availableProducts": ["P001", "P003", ...],
//   "purchaseLimits": {"P001": 2, "P003": 1, ...},
//   "alerts": [{"productId": "P001", "stock": 5, "warning": "库存紧张"}]
// }
```

**学习重点**：
- 如何保证库存查询的实时性
- 库存不足时的限购策略设计
- 库存预警的临界值设定

---

#### Agent 4：MarketingCopyAgent（营销文案）

**文件**：`agent/MarketingCopyAgent.java`

**功能**：根据用户分群生成个性化文案

```
5 套文案模板 × 5 种用户分群
├─ 新客: "首单专属福利，{商品} 立减{折扣}元！"
├─ VIP: "尊享会员特权，{商品} 专属价，品质之选。"
├─ 价格敏感: "今日限时抢购！{商品} 历史最低价，仅剩{库存}件！"
├─ 活跃: "根据您的浏览偏好，为您精选 {商品}，好评率{评分}%"
└─ 流失风险: "好久不见！{商品} 为您专属保留，点击领取优惠券"

步骤:
① 根据 UserProfile 确定分群
② 选择对应模板
③ 调用 LLM 填入变量（折扣、库存等）
④ 广告法合规校验（检查是否有"最好"、"第一"等违禁词）
```

**学习重点**：
- 文案模板的多样性为什么重要（A/B 测试）
- Prompt 变量的设计
- 合规词库的维护

---

### 🟢 第 5 层：A/B 测试引擎

**文件**：`service/ABTestService.java`

**学习目标**：
- 理解流量分桶逻辑（为什么用 MD5 哈希）
- Thompson Sampling 算法原理
- 如何在生产环境中安全地做实验

**核心概念**：

```
问题：有 3 套推荐策略，不知道哪个好，怎么选？

方案 A：灰度发布（固定分桶）
├─ 60% 用户用策略 1
├─ 20% 用户用策略 2
├─ 20% 用户用策略 3
├─ 对比点击率，选最好的
└─ ❌ 问题：前期可能浪费了很多用户在不好的策略上

方案 B：Thompson Sampling（动态分桶）✅
├─ 初始各策略分配 33%
├─ 观察点击率：策略 1 点击率 80%，策略 2 点击率 60%，策略 3 点击率 50%
├─ 自动重新分配：策略 1 → 70%，策略 2 → 20%，策略 3 → 10%
├─ 继续观察...
└─ ✅ 优点：自动收敛到最优策略，用户体验损失最少
```

**实现细节**：

```java
// 流量分桶（确保同一用户每次都在同一桶）
int bucket = Integer.parseInt(
    MD5(userId).substring(0, 8),  // MD5 哈希的前 8 位
    16  // 16 进制转 10 进制
) % 100;  // 对 100 取模 → [0, 99]

if (bucket < 60) return "strategy_1";      // 60%
else if (bucket < 80) return "strategy_2";  // 20%
else return "strategy_3";                    // 20%

// 关键特性：
// ✅ 分布式安全：不需要中心服务器记录
// ✅ 一致性：同一用户永远落到同一桶
// ✅ 灵活性：改变阈值可以动态调整流量比例
```

---

## 💻 第四部分：实践动手（2-3 小时）

### 任务 1：本地启动项目

```bash
# 1. 配置环境变量
# 编辑 src/main/resources/application.yml
# 填入你的 LLM_API_KEY

# 2. 启动项目
mvn spring-boot:run

# 3. 测试推荐接口
curl -X POST http://localhost:8080/api/v1/recommend \
  -H "Content-Type: application/json" \
  -d '{
    "userId": "user_001",
    "numItems": 5
  }'
```

**预期结果**：
```json
{
  "userId": "user_001",
  "products": [
    {"productId": "P001", "name": "iPhone 16 Pro", "score": 0.95},
    {"productId": "P003", "name": "AirPods Pro", "score": 0.88}
  ],
  "marketingCopies": [
    {"productId": "P001", "copy": "根据您的兴趣..."}
  ],
  "experimentGroup": "control",
  "totalLatencyMs": 1523.4
}
```

---

### 任务 2：理解代码执行流

**目标**：从 API 调用开始，追踪每个方法的执行

1. 打开 IDE 的调试器（Debug）
2. 在 `RecommendationController.recommend()` 方法设置断点
3. 发起 curl 请求
4. 单步执行，观察：
   - [ ] 请求参数如何解析
   - [ ] Supervisor 如何分发到各 Agent
   - [ ] CompletableFuture 的 join() 点在哪
   - [ ] 库存过滤的具体逻辑

---

### 任务 3：做一个改进实验

**选题 1：提高系统延迟**

```java
// 在 ProductRecAgent 中加入延迟模拟
Thread.sleep(2000);  // 模拟 LLM 推理时间变长

// 问题：整个推荐延迟会变多少？
// 答：Phase1 和 Phase2 各有一个 ProductRecAgent，所以总延迟会增加 2s
```

**选题 2：测试降级机制**

```java
// 在 InventoryAgent 中抛出异常
if (random.nextDouble() < 0.5) {
    throw new RuntimeException("库存服务暂时不可用");
}

// 问题：推荐请求会失败吗？
// 答：不会！因为 BaseAgent 会自动重试 3 次，如果全失败会降级
```

**选题 3：验证并行收益**

```java
// 在 SupervisorOrchestrator 中改成串行
AgentResult profileResult = userProfileAgent.runAsync(...).join();
AgentResult recResult = productRecAgent.runAsync(...).join();

// 记录总延迟，对比并行版本
System.out.println("总延迟:" + (System.nanoTime() - start) / 1_000_000 + "ms");
```

---

## 🎓 第五部分：面试准备

### 面试常问问题 & 标准答法

#### Q1：为什么用多 Agent 而不是单个大 Agent？

```
标准答法（40 秒）：
"为了解决单 Agent 在规模化时的两个核心问题：

1. 上下文膨胀：一个 Agent 要掌握 50+ 个工具会导致推理准确率下降
   对比：多 Agent 每个只关注 5-10 个工具，专注度高

2. 性能瓶颈：串行执行各个步骤会导致延迟长
   对比：多 Agent 并行执行，延迟 ≈ 最慢步骤的耗时

3. 可靠性：某个工具失败会影响整个系统
   对比：多 Agent 可以独立降级，不影响其他流程

本项目的实施例：用户画像和商品推荐两个 Agent 并行，
虽然各自耗时 3s 和 5s，但并行后总耗时只有 5s，节省了 3s。"
```

---

#### Q2：介绍一下 Supervisor 模式

```
标准答法（50 秒）：
"Supervisor 模式是 Multi-Agent 系统中最常见的编排方式。

原理：中央控制器（Supervisor）负责任务分发和结果聚合，
各 Agent 按照 Supervisor 的指令执行任务。

优势：
- 流程清晰：Supervisor 定义好每个步骤的依赖关系
- 并行执行：没有依赖的步骤可以同时进行
- 集中管理：所有 Agent 的结果都在 Supervisor 汇总，便于处理异常

对比 Handoffs 模式（Agent 间直接传递）：
- Handoffs 更适合对话场景，流程动态
- Supervisor 更适合流程固定、需要并行的场景（如本项目）

本项目的实施：
Phase 1（并行）：用户画像 + 商品推荐
Phase 2（并行）：LLM 重排 + 库存校验
Phase 3（串行）：营销文案生成"
```

---

#### Q3：Supervisor 的三个 Phase 为什么这样划分？

```
标准答法（40 秒）：
"根据数据依赖关系：

Phase 1 能并行？✅
- UserProfileAgent 和 ProductRecAgent 互不依赖
- 都只需要 userId 和基础商品库
- 可以同时开始

Phase 2 能并行？✅
- ProductRecAgent（重排）需要 Phase1 的用户画像 ✓
- InventoryAgent 需要 Phase1 的商品列表 ✓
- 但两者之间不依赖 ✓
- 所以可以同时进行

Phase 3 必须串行？✅
- MarketingCopyAgent 需要：
  ① Phase2 的精排商品列表（必须）
  ② Phase2 的库存过滤结果（必须）
  ③ 用户画像（可以用 Phase1 的结果）
- 所以只能在 Phase2 完成后执行

这样的划分能把总延迟从 8s（串行所有步骤）降低到 5s（最慢步骤）。"
```

---

### 代码相关常问

#### Q4：CompletableFuture.join() 和 get() 有什么区别？

```
标准答法：
"都能等待异步任务完成，但有关键区别：

| | join() | get() |
|--|--|--|
| 抛异常 | 抛 CompletionException（unchecked） | 抛 ExecutionException（checked） |
| 是否可以检查异常 | 编译器不强制 | 编译器强制 try-catch |
| 实际应用 | 适合流式编程（不关心异常处理） | 适合需要明确异常处理的场景 |

本项目用 join() 是因为：
异常会被 BaseAgent 的降级机制处理，不需要逐个 join() 点处理异常。"
```

---

#### Q5：Agent 的重试机制是怎样的？

```
标准答法：
"三层保障：

1. 超时控制：
   每个 Agent 有独立的超时时间（如 5s）
   超过时间就立刻返回失败，不阻塞整体

2. 指数退避重试：
   第 1 次失败 → 等 1s 重试
   第 2 次失败 → 等 2s 重试
   第 3 次失败 → 等 4s 重试
   （为什么？快速重试会在高并发下雪上加霜）

3. 降级（Fallback）：
   如果 3 次都失败，返回默认结果
   如 UserProfileAgent 失败 → 返回'新客'画像
   整个推荐流程继续执行，不中断

好处：
- 单个 Agent 故障不影响整体
- P99 延迟得到保障"
```

---

## 📊 学习进度 Checklist

按照以下 Checklist 自检学习进度：

### 第一天
- [ ] 理解 Multi-Agent 相比单 Agent 的三大优势
- [ ] 理解 Supervisor 编排的三个 Phase
- [ ] 熟悉项目目录结构
- [ ] 能够画出端到端的流程图

### 第二天
- [ ] 精读 SupervisorOrchestrator，理解 CompletableFuture 并行
- [ ] 精读 BaseAgent，理解重试 + 降级设计
- [ ] 理解 4 个 Agent 各自的功能
- [ ] 能够用自己的话解释"为什么 Phase1 和 Phase2 能并行"

### 第三天
- [ ] 本地启动项目，测试推荐接口
- [ ] 用 Debug 模式追踪代码执行
- [ ] 做至少 1 个改进实验
- [ ] 能够完整讲述"从 API 调用到推荐返回"的全过程

### 进阶
- [ ] 理解 ABTestService 的流量分桶逻辑
- [ ] 对比 Python 版本，理解三语言的区别
- [ ] 能够回答所有面试常问问题
- [ ] 能够独立修改 Prompt，调优推荐效果

---

## 🔗 参考资源

| 资源 | 说明 | 用途 |
|------|------|------|
| 根目录 `README.md` | 项目完整文档 | 整体理解 |
| 根目录 `docs/architecture.md` | 架构详解 | 深度学习 |
| 根目录 `docs/interview-guide.md` | 面试完全指南 | 面试准备 |
| 本文档 | 快速学习路线 | 入门指引 |

---

## ❓ 常见问题

### Q：我没有 Spring Boot 基础，能学吗？

A：可以。本项目的核心（Multi-Agent 编排、并行执行、容错设计）与框架无关。
只需知道：
- `@Service` = Spring 托管的服务类
- `@PostMapping` = REST API 端点
- `CompletableFuture` = Java 异步编程的标准库

### Q：学完后能干什么？

A：
1. **面试**：完全理解一个企业级多 Agent 系统
2. **项目**：可以改进你们公司的推荐系统（加入多 Agent 编排）
3. **职业发展**：掌握 AI Agent 技术，成为稀缺人才

### Q：代码量大吗？

A：总共约 1500 行 Java 代码，逻辑清晰。比其他项目（如电商后台）少很多。

### Q：怎么快速测试修改？

A：
1. 修改代码
2. `mvn clean compile` 编译
3. `mvn spring-boot:run` 重新启动
4. 发送 curl 请求测试

---

**祝学习顺利！🎉 有问题欢迎提 Issue 或讨论。**

