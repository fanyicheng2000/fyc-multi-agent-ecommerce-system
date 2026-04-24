# 🎓 核心概念速查表

> 这是一份**高效查询手册**，用于快速找到关键概念的定义和例子。

---

## 📌 目录

- [Multi-Agent 系统](#multi-agent-系统)
- [Supervisor 模式](#supervisor-模式)
- [CompletableFuture 并行](#completablefuture-并行)
- [RFM 用户分群](#rfm-用户分群)
- [两阶段推荐](#两阶段推荐)
- [容错设计](#容错设计)
- [A/B 测试](#ab-测试)

---

## Multi-Agent 系统

### 定义

将一个复杂的问题分解为多个独立的**智能体（Agent）**，每个 Agent 专注于一个特定领域，通过协调执行来解决整体问题。

### 相比单 Agent 的优势

| 维度 | 单 Agent | Multi-Agent |
|------|---------|-----------|
| **上下文膨胀** | 工具数量多 → 推理准确率低 | 每个 Agent 工具少 → 准确率高 |
| **执行速度** | 串行执行各步骤 → 延迟长 | 并行执行 → 延迟短 |
| **系统复杂度** | 难以理解，难以维护 | 清晰分工，易于维护 |
| **容错能力** | 单点失败影响整体 | 可单独降级，互不影响 |

### 本项目的 4 个 Agent

```
用户画像 Agent      → 分析用户属性（VIP/新客/价格敏感）
商品推荐 Agent      → 从百万商品中精选 10 件
库存决策 Agent      → 过滤缺货，决定限购策略
营销文案 Agent      → 生成个性化推荐文案
        ↑
      Supervisor → 协调这 4 个 Agent 的执行
```

---

## Supervisor 模式

### 定义

由一个中央控制器（Supervisor）负责：
1. **分发任务**给各 Agent
2. **管理依赖**（哪些步骤先执行，哪些并行）
3. **聚合结果**（汇总各 Agent 的输出）

### 对比其他模式

```
Supervisor 模式                 Handoffs 模式
   ┌──────┐                    Agent A
   │ S    │                       ↓
   │ U    │                    Agent B
   │ P    │                       ↓
   │ E    │                    Agent C
   │ R    │
   │ V    │                优点：灵活、适合对话
   │ I    │                缺点：难以并行
   │ S    │
   │ O    │                本项目不用这个
   │ R    │
   └──────┘
    ↙ ↓ ↘

优点：清晰、可并行
缺点：需要编排逻辑
本项目采用这个 ✅
```

### 本项目的 3 Phase 设计

```
Phase 1（并行 ↔）
├─ UserProfileAgent（耗时 200ms）
└─ ProductRecAgent-召回（耗时 500ms）
    ↓（等两者都完成）
Phase 2（并行 ↔）
├─ ProductRecAgent-精排（耗时 800ms）
└─ InventoryAgent（耗时 300ms）
    ↓（等两者都完成）
Phase 3（串行 →）
└─ MarketingCopyAgent（耗时 600ms）

总耗时计算：
❌ 串行所有：200+500+800+300+600 = 2400ms
✅ 并行设计：max(200,500) + max(800,300) + 600 = 500+800+600 = 1900ms
节省：500ms（21% 提升）
```

---

## CompletableFuture 并行

### 基础语法

```java
// 创建异步任务 1
CompletableFuture<Result1> future1 = CompletableFuture.supplyAsync(() -> {
    return agent1.run();  // 异步执行，不阻塞主线程
});

// 创建异步任务 2
CompletableFuture<Result2> future2 = CompletableFuture.supplyAsync(() -> {
    return agent2.run();
});

// 等待两个任务都完成
Result1 result1 = future1.join();
Result2 result2 = future2.join();

// 关键：两个 run() 是同时开始的，而不是一个接一个
```

### 与串行的对比

```java
// ❌ 串行（慢）
long start = System.currentTimeMillis();
Result1 r1 = agent1.run();  // 阻塞 500ms
Result2 r2 = agent2.run();  // 阻塞 800ms
System.out.println("耗时: " + (System.currentTimeMillis() - start));  // 约 1300ms

// ✅ 并行（快）
long start = System.currentTimeMillis();
CompletableFuture<Result1> f1 = agent1.runAsync();  // 立刻返回，后台执行
CompletableFuture<Result2> f2 = agent2.runAsync();  // 立刻返回，后台执行
Result1 r1 = f1.join();  // 等待 f1
Result2 r2 = f2.join();  // 等待 f2
System.out.println("耗时: " + (System.currentTimeMillis() - start));  // 约 800ms（两者最大值）
```

### 常见问题

**Q: join() 会让 CPU 空转吗？**

A: 不会。join() 让当前线程进入等待状态，CPU 会处理其他任务。

**Q: 为什么不用 get()?**

A: 都可以，但 join() 不需要 try-catch（因为抛的是 unchecked exception）。

**Q: CompletableFuture 的线程来自哪里？**

A: 默认用 `ForkJoinPool.commonPool()`（共享线程池）。可以指定自己的 ExecutorService。

---

## RFM 用户分群

### RFM 三个维度

| 维度 | 英文 | 解释 | 示例 |
|------|------|------|------|
| **R** | Recency | 距离上次购买**多少天** | 最近 3 天买过 → 高分 |
| **F** | Frequency | **买过多少次**（周期内） | 7 天内买 5 次 → 高分 |
| **M** | Monetary | **花过多少钱** | 消费总额 1w+ → 高分 |

### 计算流程

```
原始数据（用户 u001）：
- 上次购买：3 天前
- 近 7 天购买次数：5 次
- 消费总额：1500 元

        ↓ 归一化（转换到 0-1 区间）

- R_norm = (30 - 3) / 30 = 0.9   （最近越好）
- F_norm = min(5 / 10, 1) = 0.5   （频率越高越好）
- M_norm = min(1500 / 10000, 1) = 0.15  （消费越多越好）

        ↓ 加权求和

RFM_score = 0.3 * 0.9 + 0.3 * 0.5 + 0.4 * 0.15
          = 0.27 + 0.15 + 0.06
          = 0.48 → 中等用户

        ↓ 映射到分群

VIP:            rfm_score > 0.8
活跃用户:       0.6 < rfm_score ≤ 0.8
普通用户:       0.3 < rfm_score ≤ 0.6
流失风险:       rfm_score ≤ 0.3
```

### 5 种用户分群的营销策略

```
分群名        特征                   推荐策略              文案例子
════════════════════════════════════════════════════════════════════
VIP          高 RFM 分数             精选商品 + 优先发货    "尊享会员特权，专属价..."
活跃用户     中高 RFM 分数           频繁推荐              "根据您的兴趣，为您精选..."
新客户       无购买历史              降低决策成本，小折扣   "首单专属福利，立减..."
价格敏感     频繁买但消费少          强调折扣、价格优势     "历史最低价，仅剩..."
流失风险     很久没买过              复购激励              "好久不见！专属优惠券..."
```

---

## 两阶段推荐

### Phase 1：多路召回（Recall）

目标：从百万商品中筛出**几百件候选**

```
策略 1：协同过滤
输入：用户ID
逻辑：找买过相同商品的其他用户，看他们还买了什么
输出：100 件商品

策略 2：向量检索
输入：用户ID + 最近浏览商品
逻辑：用 Milvus 等向量数据库，找语义相似的商品
输出：100 件商品

策略 3：热度策略
输入：无
逻辑：最近 7 天销量 TOP
输出：50 件商品

策略 4：新品策略
输入：无
逻辑：上架 30 天内、销量 TOP
输出：50 件商品

    ↓ 去重合并

候选池：约 200-300 件商品
```

**为什么要多路召回？**
- 单路算法有盲点（热门商品偏差、长尾遗漏）
- 多路组合能覆盖用户的多种需求（热销品 + 个性化 + 新品）

### Phase 2：LLM 精排（Rank）

目标：从候选中精选 10 件**最相关的**商品

```java
// Prompt 构造
String prompt = """
用户画像：%s
最近浏览：%s
用户分群：%s

以下 200 件商品，请按用户偏好度从高到低排序，只返回 ID 列表：
%s
""";

// 调用 LLM
String response = llm.call(prompt);
// 返回：["P001", "P003", "P005", "P008", ...]

// 只取 Top 10
List<String> top10 = response.split(",").limit(10).collect(toList());
```

**精排 vs 召回**

| | 召回 | 精排 |
|--|--|--|
| 目标 | 减少搜索空间 | 最优排序 |
| 方法 | 多个启发式算法 | 单个 LLM 模型 |
| 准确率要求 | 不高（只要相关就行） | 很高（顺序要对） |
| 速度要求 | 要快（否则候选太少） | 可以慢（处理数少） |

---

## 容错设计

### 三层保障

```
第 1 层：超时控制
┌──────────────────────┐
│ Agent 执行中...      │
│ 0s → 2s → 4s → 6s    │
│       │       ↓      │
│       │   超时 5s    │
│       │   立刻返回   │
└──────────────────────┘

第 2 层：指数退避重试
第 1 次失败 → 等 1s 重试
第 2 次失败 → 等 2s 重试
第 3 次失败 → 等 4s 重试

（为什么要等？高并发下，立刻重试会加重服务压力，反而更容易失败）

第 3 层：降级（Fallback）
3 次重试全失败 → 返回默认结果
UserProfileAgent 失败 → 返回"新客用户画像"
InventoryAgent 失败 → 返回"全部商品可售"
系统继续执行，推荐请求成功 ✅
```

### 代码实现（BaseAgent）

```java
public abstract class BaseAgent {
    private static final int MAX_RETRIES = 3;
    private static final long RETRY_DELAY = 1000;  // 1s
    private static final long TIMEOUT = 5000;      // 5s

    public AgentResult runSync(...) {
        for (int attempt = 0; attempt < MAX_RETRIES; attempt++) {
            try {
                // 执行 Agent，超过 5s 就抛 TimeoutException
                return executeWithTimeout(TIMEOUT);
            } catch (TimeoutException | IOException e) {
                if (attempt < MAX_RETRIES - 1) {
                    // 指数退避：1s, 2s, 4s
                    long delay = RETRY_DELAY * (1L << attempt);
                    Thread.sleep(delay);
                }
            }
        }
        // 3 次都失败，降级
        return fallback();
    }

    public AgentResult fallback() {
        // 返回"说得过去的"默认结果
        // 由子类具体实现
    }
}
```

---

## A/B 测试

### 流量分桶（Bucketing）

目标：保证**同一用户**在**多次请求**中落到**同一个实验组**

```java
// 方法：MD5 哈希取模
int bucket = Integer.parseInt(
    MD5(userId).substring(0, 8),  // MD5 值的前 8 位
    16  // 16 进制转 10 进制
) % 100;  // 对 100 取模，得到 [0, 99]

if (bucket < 60) return "control";           // 60%
else if (bucket < 80) return "treatment_v1";  // 20%
else return "treatment_v2";                    // 20%
```

**为什么用 MD5？**
- 分布式安全：不需要中心服务器记录用户桶位
- 一致性：同一 userId 每次 hash 结果都一样
- 灵活性：修改阈值可动态调整流量

### Thompson Sampling（动态调优）

问题：3 套策略，不知道哪个好。用哪个？

```
普通方案：固定分配 60% 控制组 + 20% 测试组 1 + 20% 测试组 2
问题：可能浪费了 60% 用户在不好的策略上 ❌

Thompson Sampling：自动收敛到最优策略
┌─────────────────────────────────────┐
│ 初始：各策略都是 1/3                 │
│                                     │
│ 1 小时后：控制组 CTR 80%            │
│          测试 1 CTR 60%              │
│          测试 2 CTR 50%              │
│                                     │
│ 自动调整：控制 70% / 测试1 20% / 测试2 10%  │
│                                     │
│ 继续观察...逐步收敛                  │
└─────────────────────────────────────┘

数学模型：Beta 分布
Alpha[i] = 该策略的点击次数
Beta[i] = 该策略的未点击次数
胜率[i] = Beta(Alpha[i], Beta[i]) 采样值
选胜率最高的策略
```

**好处**：
- 自动收敛到最优
- 用户体验损失最少（不是固定浪费 40% 用户）
- 适应动态变化（策略表现变差会自动降低其分配）

---

## 快速查询表

| 概念 | 用一句话说 | 代码位置 |
|------|---------|--------|
| Supervisor | 中央控制器，协调所有 Agent | SupervisorOrchestrator.java |
| CompletableFuture | Java 异步并行的标准工具 | 任何 runAsync() 调用 |
| BaseAgent | 所有 Agent 的基类，定义重试/超时/降级 | BaseAgent.java |
| RFM | 用三个指标判断用户价值 | UserProfileAgent.java |
| 多路召回 | 用不同算法各召回一部分商品 | ProductRecAgent.java |
| LLM 精排 | 用大模型排序，得到最相关的商品 | ProductRecAgent.java |
| 降级 | Agent 失败不中断整体，返回默认结果 | BaseAgent.fallback() |
| MD5 分桶 | 用哈希保证用户分桶的一致性 | ABTestService.assign() |
| Thompson Sampling | A/B 测试中自动给好的策略分配更多流量 | ABTestService.recordClick() |

---

**下次想起什么概念，来这儿速查，省时间！⏱️**

