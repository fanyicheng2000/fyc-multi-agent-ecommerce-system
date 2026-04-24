# 👨‍💻 代码逐行讲解（初学者友好版）

> 这篇文档逐行讲解项目的关键代码，帮你理解**每一行在干什么**。

---

## 目录

1. [API 入口](#1-api-入口)
2. [Supervisor 编排器](#2-supervisor-编排器)
3. [BaseAgent 基类](#3-baseagent-基类)
4. [4 个具体 Agent](#4-个具体-agent)
5. [A/B 测试服务](#5-ab-测试服务)

---

## 1. API 入口

**文件**：`config/RecommendationController.java`

### 代码框架

```java
@RestController
@RequestMapping("/api/v1")
public class RecommendationController {

    @PostMapping("/recommend")
    public ResponseEntity<RecommendationResponse> recommend(
            @RequestBody RecommendationRequest request) {
        // 核心逻辑
    }
}
```

### 逐行解读

```java
@RestController                              // ① 标记这是 REST API 控制器
                                             //    Spring 会自动处理 HTTP 请求/响应

@RequestMapping("/api/v1")                   // ② 定义 API 前缀
                                             //    所有方法的 URL 都以 /api/v1 开头

public class RecommendationController {     // ③ 控制器类

    // 注入依赖（由 Spring 容器管理）
    private final SupervisorOrchestrator supervisor;

    public RecommendationController(
        SupervisorOrchestrator supervisor    // ④ 构造函数注入
                                             //    Spring 会自动创建 Supervisor 实例
    ) {
        this.supervisor = supervisor;
    }

    @PostMapping("/recommend")               // ⑤ 定义 POST 端点
                                             //    完整 URL: POST /api/v1/recommend

    public ResponseEntity<RecommendationResponse> recommend(
        @RequestBody RecommendationRequest request  // ⑥ 自动从 JSON 解析请求体
    ) {
        try {
            // ⑦ 调用 Supervisor 的 recommend() 方法
            RecommendationResponse response =
                supervisor.recommend(request);

            // ⑧ 返回 HTTP 200 + response JSON
            return ResponseEntity.ok(response);

        } catch (Exception e) {
            // ⑨ 异常处理
            return ResponseEntity.status(500)
                .body(new RecommendationResponse());
        }
    }
}
```

### 请求流程示意

```
用户 curl 请求
    ↓
HTTP POST /api/v1/recommend
{
  "userId": "user_001",
  "numItems": 5
}
    ↓
Spring 识别到 @PostMapping("/recommend")
    ↓
@RequestBody 自动将 JSON 解析成 RecommendationRequest 对象
    ↓
调用 recommend(request) 方法
    ↓
supervisor.recommend(request)
    ↓
返回 RecommendationResponse 对象
    ↓
Spring 自动将对象序列化成 JSON
    ↓
HTTP 200 响应给用户
```

---

## 2. Supervisor 编排器

**文件**：`orchestrator/SupervisorOrchestrator.java`

### 核心流程

```java
public RecommendationResponse recommend(RecommendationRequest request) {
    // ① 生成请求 ID（用于日志追踪）
    String requestId = UUID.randomUUID().toString();
    long start = System.nanoTime();

    // ② 分配 A/B 实验组（在最开始就决定用哪套策略）
    String experimentGroup = abTestService
        .assign(request.getUserId())           // 根据 userId 分桶
        .getOrDefault("group", "control")      // 取 group 字段，默认值 "control"
        .toString();

    // ③ Phase 1：并行执行
    //    两个任务同时开始，不互相等待
    CompletableFuture<AgentResult> profileFuture =
        userProfileAgent.runAsync(                // 异步执行
            Map.of("userId", request.getUserId()) // 参数：userId
        );

    CompletableFuture<AgentResult> recFuture =
        productRecAgent.runAsync(
            Map.of("numItems", request.getNumItems() * 2)  // 召回要比最终需要的多
        );

    // ④ 等待两个异步任务都完成（join() 阻塞当前线程）
    AgentResult profileResult = profileFuture.join();
    AgentResult recResult = recFuture.join();

    // 关键！总耗时 = max(userProfileAgent 耗时, productRecAgent 耗时)
    // 而不是两者相加！

    // ⑤ 从 AgentResult 中提取数据
    UserProfile profile = (UserProfile)
        profileResult.getData().get("profile");  // 类型强制转换

    @SuppressWarnings("unchecked")                // ⑥ 压制 IDE 警告
    List<Product> rawProducts =
        (List<Product>) recResult.getData()
        .get("products");

    // ⑦ Phase 2：并行执行
    //    需要 Phase1 的结果，所以必须放在后面
    CompletableFuture<AgentResult> rerankFuture =
        productRecAgent.runAsync(
            Map.of(
                "userProfile", profile,          // 使用 Phase1 的结果
                "numItems", request.getNumItems()
            )
        );

    CompletableFuture<AgentResult> inventoryFuture =
        inventoryAgent.runAsync(
            Map.of("products", rawProducts)      // 使用 Phase1 的结果
        );

    // ⑧ 等待 Phase2 完成
    AgentResult rerankResult = rerankFuture.join();
    AgentResult inventoryResult = inventoryFuture.join();

    // ⑨ 提取 Phase2 的结果
    List<Product> rankedProducts =
        (List<Product>) rerankResult.getData()
        .get("products");

    List<String> availableIds =
        (List<String>) inventoryResult.getData()
        .get("available_products");

    // ⑩ 库存过滤：只保留有货的商品
    Set<String> availSet = new HashSet<>(availableIds);
    List<Product> finalProducts = rankedProducts
        .stream()
        .filter(p -> availSet.contains(p.getProductId()))
        .limit(request.getNumItems())            // 限制数量
        .collect(Collectors.toList());

    // ⑪ Phase 3：串行执行（需要前两步结果）
    AgentResult copyResult =
        marketingCopyAgent.runAsync(
            Map.of(
                "userProfile", profile,
                "products", finalProducts
            )
        ).join();

    // ⑫ 提取文案
    List<String> copies =
        (List<String>) copyResult.getData()
        .get("copies");

    // ⑬ 计算总延迟
    long durationNanos = System.nanoTime() - start;
    double durationMs = durationNanos / 1_000_000.0;  // 纳秒转毫秒

    // ⑭ 构建响应
    RecommendationResponse response =
        new RecommendationResponse()
            .setUserId(request.getUserId())
            .setProducts(finalProducts)
            .setMarketingCopies(copies)
            .setExperimentGroup(experimentGroup)
            .setTotalLatencyMs(durationMs);

    return response;
}
```

### 关键概念解释

#### CompletableFuture 的两种用法

```java
// 用法 1：supplyAsync（有返回值）
CompletableFuture<Result> future =
    CompletableFuture.supplyAsync(() -> {
        // 这里的代码在后台线程执行
        return agent.run();
    });

// 用法 2：join()（阻塞等待）
Result result = future.join();  // 阻塞直到 supplyAsync 完成
```

#### 为什么要 * 2？

```java
// 召回时申请 2 倍的商品数
numItems = request.getNumItems() * 2;  // 比如用户要 5 件，召回 10 件

// 原因：
// 1. 精排会过滤一些不相关的
// 2. 库存过滤会去掉缺货的
// 所以需要多召回一些，才能最后凑齐需要数量
```

---

## 3. BaseAgent 基类

**文件**：`agent/BaseAgent.java`

### 完整代码

```java
public abstract class BaseAgent {

    // ① 常量定义
    protected static final int MAX_RETRIES = 3;           // 最多重试 3 次
    protected static final long RETRY_DELAY = 1000;       // 初始延迟 1s
    protected static final long TIMEOUT = 5000;           // 超时 5s

    protected String name;                                 // Agent 名称（用于日志）

    // ② 同步执行方法（阻塞当前线程）
    public final AgentResult runSync(Map<String, Object> input) {
        try {
            // ③ 调用有超时的执行方法
            return executeWithTimeout(input);

        } catch (TimeoutException e) {
            // ④ 超时异常
            log.warn("{} timeout, triggering fallback", name);
            return fallback(input);

        } catch (Exception e) {
            // ⑤ 其他异常（包括重试失败）
            log.error("{} failed", name, e);
            return fallback(input);
        }
    }

    // ⑥ 异步执行方法（返回 Future）
    public final CompletableFuture<AgentResult> runAsync(
        Map<String, Object> input
    ) {
        return CompletableFuture.supplyAsync(() -> {
            // ⑦ 在后台线程池中执行 runSync
            return runSync(input);
        });
    }

    // ⑧ 私有方法：带重试的执行
    private AgentResult executeWithTimeout(
        Map<String, Object> input
    ) throws Exception {

        // ⑨ 指数退避重试：1s, 2s, 4s
        for (int attempt = 0; attempt < MAX_RETRIES; attempt++) {
            try {
                // ⑩ 执行具体逻辑（由子类实现）
                AgentResult result = execute(input);

                if (result.isSuccess()) {
                    return result;
                }

            } catch (TimeoutException e) {
                if (attempt == MAX_RETRIES - 1) {
                    throw e;  // 最后一次重试还是超时，抛异常
                }
                // ⑪ 指数退避：1 << attempt = 2^attempt
                long delay = RETRY_DELAY * (1L << attempt);
                Thread.sleep(delay);

            } catch (Exception e) {
                // ⑫ 其他异常也重试
                if (attempt == MAX_RETRIES - 1) {
                    throw e;
                }
                Thread.sleep(RETRY_DELAY * (1L << attempt));
            }
        }

        // ⑬ 不应该到这里（前面已经抛异常了）
        throw new RuntimeException(name + " failed after retries");
    }

    // ⑭ 抽象方法：由子类实现具体业务逻辑
    protected abstract AgentResult execute(
        Map<String, Object> input
    ) throws Exception;

    // ⑮ 降级方法：Agent 失败时返回默认结果
    protected abstract AgentResult fallback(
        Map<String, Object> input
    );
}
```

### 执行流程图

```
调用 runSync(input)
    ↓
try {
    executeWithTimeout(input)
        ↓
    第 1 次尝试
        ↓ 失败
    catch: 等 1s，重试
        ↓
    第 2 次尝试
        ↓ 失败
    catch: 等 2s，重试
        ↓
    第 3 次尝试
        ↓ 失败
    catch: 等 4s，重试
        ↓
    全部失败，抛异常
}
catch (Exception e) {
    return fallback(input)  ← 返回默认结果
}
```

---

## 4. 个具体 Agent

### 4.1 UserProfileAgent（用户画像）

```java
@Service
public class UserProfileAgent extends BaseAgent {

    @Override
    protected AgentResult execute(Map<String, Object> input) throws Exception {

        // ① 解析输入参数
        String userId = (String) input.get("userId");

        // ② 从 Redis 获取用户行为特征
        Map<String, Object> behaviors =
            featureStore.getUserFeatures(userId);
        //    返回: {
        //      "clicks_1h": 12,
        //      "purchases_7d": 3,
        //      "categories": ["手机", "耳机"]
        //    }

        // ③ 调用 LLM 分析用户
        String prompt = String.format(
            "用户行为数据: %s\n请分析用户分群和RFM得分，输出JSON",
            behaviors
        );

        String llmResponse = llm.invoke(prompt);
        //    返回: {
        //      "segment": "active",
        //      "rfmScore": 0.75
        //    }

        // ④ 解析 LLM 响应，构建 UserProfile
        UserProfile profile = new UserProfile()
            .setUserId(userId)
            .setSegment(llmResponse.getString("segment"))
            .setRfmScore(llmResponse.getDouble("rfmScore"));

        // ⑤ 包装成 AgentResult 返回
        AgentResult result = new AgentResult();
        result.setData(Map.of("profile", profile));
        result.setSuccess(true);

        return result;
    }

    @Override
    protected AgentResult fallback(Map<String, Object> input) {
        // ⑥ 降级：返回"新客"画像
        UserProfile defaultProfile = new UserProfile()
            .setSegment("new_customer")
            .setRfmScore(0.3);

        AgentResult result = new AgentResult();
        result.setData(Map.of("profile", defaultProfile));
        result.setSuccess(true);

        return result;
    }
}
```

### 4.2 ProductRecAgent（商品推荐）

```java
@Service
public class ProductRecAgent extends BaseAgent {

    @Override
    protected AgentResult execute(Map<String, Object> input) throws Exception {

        // ① 判断是召回还是精排
        UserProfile userProfile =
            (UserProfile) input.get("userProfile");
        int numItems = (Integer) input.get("numItems");

        if (userProfile == null) {
            // ② 阶段 1：多路召回（召回阶段）
            return recall(numItems);

        } else {
            // ③ 阶段 2：LLM 精排（精排阶段）
            return rerank(userProfile, numItems);
        }
    }

    private AgentResult recall(int numItems) throws Exception {
        // ④ 多路召回
        List<Product> candidates = new ArrayList<>();

        // 协同过滤
        candidates.addAll(collaborativeFilter());

        // 向量检索
        candidates.addAll(vectorSearch());

        // 热度策略
        candidates.addAll(hotProducts());

        // ⑤ 去重
        List<Product> unique = candidates.stream()
            .distinct()  // 去重（基于 productId）
            .limit(numItems * 2)  // 限制在 2 倍
            .collect(Collectors.toList());

        return AgentResult.success(
            Map.of("products", unique)
        );
    }

    private AgentResult rerank(
        UserProfile profile,
        int numItems
    ) throws Exception {
        // ⑥ 获取候选商品列表（前面步骤已提供）
        List<Product> candidates = getCandidates();

        // ⑦ 构造 LLM 精排 Prompt
        String prompt = String.format(
            "用户分群: %s\n" +
            "用户偏好: %s\n" +
            "以下 %d 件商品，请按相关性从高到低排序:\n" +
            "%s\n" +
            "只返回商品 ID 列表，用逗号分隔",
            profile.getSegment(),
            profile.getPreferences(),
            candidates.size(),
            candidates.stream()
                .map(p -> p.getProductId() + ":" + p.getName())
                .collect(Collectors.joining("\n"))
        );

        // ⑧ 调用 LLM
        String llmResponse = llm.invoke(prompt);
        //    返回: "P001,P003,P005,P008,..."

        // ⑨ 解析并排序
        List<String> rankedIds =
            Arrays.asList(llmResponse.split(","));

        Map<String, Integer> rankMap = new HashMap<>();
        for (int i = 0; i < rankedIds.size(); i++) {
            rankMap.put(rankedIds.get(i), i);
        }

        // ⑩ 按排名重排产品
        List<Product> ranked = candidates.stream()
            .sorted((a, b) -> {
                Integer rankA = rankMap.getOrDefault(
                    a.getProductId(),
                    Integer.MAX_VALUE
                );
                Integer rankB = rankMap.getOrDefault(
                    b.getProductId(),
                    Integer.MAX_VALUE
                );
                return rankA.compareTo(rankB);
            })
            .limit(numItems)
            .collect(Collectors.toList());

        return AgentResult.success(
            Map.of("products", ranked)
        );
    }
}
```

### 4.3 InventoryAgent（库存决策）

```java
@Service
public class InventoryAgent extends BaseAgent {

    @Override
    protected AgentResult execute(Map<String, Object> input)
        throws Exception {

        // ① 获取商品列表
        @SuppressWarnings("unchecked")
        List<Product> products =
            (List<Product>) input.get("products");

        // ② 查询数据库，获取库存
        List<String> available = new ArrayList<>();
        Map<String, Integer> limits = new HashMap<>();

        for (Product p : products) {
            Integer stock = database.getStock(
                p.getProductId()
            );

            if (stock > 0) {
                // ③ 有货商品加入列表
                available.add(p.getProductId());

                // ④ 根据库存计算限购数
                if (stock < 5) {
                    limits.put(p.getProductId(), 1);  // 紧张：限 1 件
                } else if (stock < 20) {
                    limits.put(p.getProductId(), 2);  // 中等：限 2 件
                } else {
                    limits.put(p.getProductId(), 5);  // 充足：限 5 件
                }
            }
        }

        // ⑤ 构建响应
        AgentResult result = new AgentResult();
        result.setData(Map.of(
            "available_products", available,
            "purchase_limits", limits
        ));
        result.setSuccess(true);

        return result;
    }

    @Override
    protected AgentResult fallback(Map<String, Object> input) {
        // ⑥ 降级：假设所有商品都有货
        @SuppressWarnings("unchecked")
        List<Product> products =
            (List<Product>) input.get("products");

        List<String> allIds = products.stream()
            .map(Product::getProductId)
            .collect(Collectors.toList());

        AgentResult result = new AgentResult();
        result.setData(Map.of(
            "available_products", allIds
        ));
        result.setSuccess(true);

        return result;
    }
}
```

### 4.4 MarketingCopyAgent（营销文案）

```java
@Service
public class MarketingCopyAgent extends BaseAgent {

    // ① 文案模板
    private static final Map<String, String> TEMPLATES =
        Map.ofEntries(
            Map.entry("new_customer",
                "首单专属福利，{product}立减{discount}元！"),
            Map.entry("vip",
                "尊享会员特权，{product}专属价，品质之选。"),
            Map.entry("price_sensitive",
                "今日限时抢购！{product}历史最低价，仅剩{stock}件！"),
            Map.entry("active",
                "根据您的浏览偏好，为您精选 {product}，好评率{rating}%"),
            Map.entry("churn_risk",
                "好久不见！{product}为您专属保留，点击领取优惠券")
        );

    // ② 广告法禁用词
    private static final Set<String> BANNED_WORDS =
        Set.of("最好", "第一", "最便宜", "绝对", "100%");

    @Override
    protected AgentResult execute(Map<String, Object> input)
        throws Exception {

        // ③ 获取用户画像和商品
        UserProfile profile =
            (UserProfile) input.get("userProfile");

        @SuppressWarnings("unchecked")
        List<Product> products =
            (List<Product>) input.get("products");

        // ④ 生成文案
        List<String> copies = new ArrayList<>();

        for (Product p : products) {
            // ⑤ 选择模板
            String template = TEMPLATES.get(
                profile.getSegment()
            );

            // ⑥ 填入变量
            String copy = template
                .replace("{product}", p.getName())
                .replace("{discount}", "50")
                .replace("{stock}", String.valueOf(p.getStock()))
                .replace("{rating}", String.valueOf(p.getRating()));

            // ⑦ 合规校验
            for (String bannedWord : BANNED_WORDS) {
                if (copy.contains(bannedWord)) {
                    // 如果包含违禁词，移除
                    copy = copy.replace(bannedWord, "");
                }
            }

            copies.add(copy);
        }

        // ⑧ 返回
        AgentResult result = new AgentResult();
        result.setData(Map.of("copies", copies));
        result.setSuccess(true);

        return result;
    }

    @Override
    protected AgentResult fallback(Map<String, Object> input) {
        // ⑨ 降级：返回简单文案
        @SuppressWarnings("unchecked")
        List<Product> products =
            (List<Product>) input.get("products");

        List<String> simpleCopies = products.stream()
            .map(p -> "精选商品: " + p.getName())
            .collect(Collectors.toList());

        AgentResult result = new AgentResult();
        result.setData(Map.of("copies", simpleCopies));
        result.setSuccess(true);

        return result;
    }
}
```

---

## 5. A/B 测试服务

**文件**：`service/ABTestService.java`

```java
@Service
public class ABTestService {

    public Map<String, Object> assign(String userId) {
        // ① 用 MD5 哈希 userId
        String md5Hex = DigestUtils.md5DigestAsHex(
            userId.getBytes()
        );

        // ② 取前 8 位，转成 10 进制数
        int bucket = Integer.parseInt(
            md5Hex.substring(0, 8),  // 前 8 位 16 进制
            16                       // 16 进制转 10 进制
        ) % 100;  // 对 100 取模，得 [0, 99]

        // ③ 根据 bucket 分配策略
        String group, strategy;

        if (bucket < 60) {
            // ④ 60% 流量分配给控制组
            group = "control";
            strategy = "collaborative_filter";

        } else if (bucket < 80) {
            // ⑤ 20% 流量分配给测试 1
            group = "treatment_v1";
            strategy = "llm_rerank";

        } else {
            // ⑥ 20% 流量分配给测试 2
            group = "treatment_v2";
            strategy = "vector_search";
        }

        // ⑦ 返回分配结果
        return Map.of(
            "group", group,
            "strategy", strategy,
            "bucket", bucket  // 用于调试
        );
    }

    public void recordClick(String userId, boolean clicked) {
        // ① 获取用户的实验组
        Map<String, Object> assignment =
            assign(userId);  // 重复计算（保证一致性）
        String group = (String) assignment.get("group");

        // ② 记录点击事件（送到数据库或 Redis）
        if (clicked) {
            database.increment(group + ":clicks");     // 点击数 +1
        } else {
            database.increment(group + ":impressions"); // 展示数 +1
        }

        // ③ 计算当前 CTR
        long clicks = database.get(group + ":clicks");
        long impressions = database.get(
            group + ":impressions"
        );
        double ctr = (double) clicks / impressions;

        // ④ 更新 Beta 分布参数（Thompson Sampling）
        // 简化版：直接记录点击数和展示数
        // 复杂版：根据 Beta 分布更新流量比例
    }
}
```

### MD5 分桶原理

```
输入：userId = "user_001"
    ↓
MD5 哈希：MD5("user_001") = "8a9b7c5d2e1f4g3h..."（16 进制）
    ↓
取前 8 位："8a9b7c5d"
    ↓
16 进制转 10 进制：0x8a9b7c5d = 2325298525（某个大数）
    ↓
对 100 取模：2325298525 % 100 = 25（[0, 99] 范围内）
    ↓
根据阈值分配：
    if (25 < 60) → 控制组 ✓
    else if (25 < 80) → 测试 1
    else → 测试 2

关键特性：
- 同一 userId 每次得到的 bucket 都是 25（确定性）
- 不同 userId 的 bucket 均匀分布（[0, 99]）
- 无需中心服务器记录（分布式安全）
```

---

## 记忆助手

| 想干什么 | 看哪段代码 |
|--------|--------|
| 理解请求怎么到达系统 | RecommendationController |
| 理解 Supervisor 怎么协调 | SupervisorOrchestrator.recommend() |
| 理解异步并行 | CompletableFuture.supplyAsync() + join() |
| 理解重试机制 | BaseAgent.executeWithTimeout() |
| 理解降级 | BaseAgent.fallback() |
| 理解用户分群 | UserProfileAgent.execute() |
| 理解两阶段推荐 | ProductRecAgent.recall() + rerank() |
| 理解库存过滤 | InventoryAgent.execute() |
| 理解文案生成 | MarketingCopyAgent.execute() |
| 理解 A/B 测试 | ABTestService.assign() |

---

**现在你已经理解每一行代码在干什么了！🎉**

下一步：用 IDE 的 Debug 功能，实际执行代码，单步跟踪，对照本文档理解执行流程。

