# 十七、推荐系统实操项目详解

> 以真实项目为背景，详细讲解知识点在实际项目中的应用，包含完整的技术方案和代码实现。

---

## 17.1 项目一：电商首页推荐系统

### 17.1.1 项目背景

```
【业务背景】
├── 平台：某电商平台，日活用户5000万
├── 场景：首页信息流推荐
├── 目标：提升用户点击率和GMV
└── 痛点：旧系统延迟高、扩展性差、效果不稳定

【系统目标】
├── 性能目标
│   ├── P99延迟 < 30ms
│   ├── QPS > 5万
│   └── 可用性 > 99.99%
│
├── 业务目标
│   ├── CTR提升 > 5%
│   ├── GMV提升 > 3%
│   └── 人均停留时长提升 > 10%
│
└── 技术目标
    ├── 支持实时特征更新
    ├── 支持模型热更新
    └── 支持A/B实验
```

### 17.1.2 系统架构设计

```
【整体架构】

┌─────────────────────────────────────────────────────────────────┐
│                         用户请求                                │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API网关层                                  │
│  ├── 请求鉴权、限流、路由                                       │
│  └── 监控埋点、日志记录                                         │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      推荐服务层                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ 召回服务  │→│ 粗排服务  │→│ 精排服务  │→│ 重排服务  │        │
│  │ P99<15ms │  │ P99<10ms │  │ P99<20ms │  │ P99<5ms  │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      支撑服务层                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ 特征服务  │  │ 模型服务  │  │ 配置服务  │  │ 实验服务  │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      数据存储层                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │  Redis   │  │  MySQL   │  │   ES     │  │  Kafka   │        │
│  │ 特征缓存 │  │ 业务数据 │  │ 检索索引 │  │ 消息队列 │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
└─────────────────────────────────────────────────────────────────┘
```

### 17.1.3 召回服务实现

```java
/**
 * 召回服务 - 多路召回调度器
 * 
 * 知识点应用：
 * 1. 多路召回架构
 * 2. 并行调度优化
 * 3. 结果合并去重
 */
@Service
public class RecallService {
    
    @Resource
    private List<RecallChannel> recallChannels;  // 多路召回通道
    
    @Resource
    private ThreadPoolExecutor recallExecutor;   // 并行线程池
    
    /**
     * 多路召回入口
     * 
     * 性能要求：P99 < 15ms
     * 实现策略：并行调度 + 结果合并
     */
    public RecallResult recall(RecallRequest request) {
        long startTime = System.currentTimeMillis();
        
        // 1. 构建召回上下文
        RecallContext context = buildContext(request);
        
        // 2. 并行调度各路召回
        List<CompletableFuture<ChannelResult>> futures = new ArrayList<>();
        for (RecallChannel channel : recallChannels) {
            futures.add(CompletableFuture.supplyAsync(
                () -> channel.recall(context), 
                recallExecutor
            ));
        }
        
        // 3. 等待所有召回完成（设置超时）
        List<ChannelResult> results = futures.stream()
            .map(f -> {
                try {
                    return f.get(100, TimeUnit.MILLISECONDS);  // 单路超时100ms
                } catch (Exception e) {
                    log.warn("召回通道超时: {}", e.getMessage());
                    return ChannelResult.empty();
                }
            })
            .collect(Collectors.toList());
        
        // 4. 结果合并去重
        List<RecallItem> mergedItems = mergeAndDeduplicate(results);
        
        // 5. 截断到指定数量
        List<RecallItem> finalItems = mergedItems.stream()
            .limit(request.getRecallSize())
            .collect(Collectors.toList());
        
        // 6. 记录监控
        long cost = System.currentTimeMillis() - startTime;
        MetricsUtil.recordLatency("recall_latency", cost);
        
        return new RecallResult(finalItems, cost);
    }
    
    /**
     * 结果合并去重
     * 
     * 策略：
     * 1. 按优先级排序
     * 2. 去重（保留分数最高的）
     * 3. 配额控制（每路最多占比）
     */
    private List<RecallItem> mergeAndDeduplicate(List<ChannelResult> results) {
        // 按itemId分组，保留分数最高的
        Map<String, RecallItem> itemMap = new HashMap<>();
        
        for (ChannelResult result : results) {
            for (RecallItem item : result.getItems()) {
                String itemId = item.getItemId();
                if (!itemMap.containsKey(itemId) || 
                    item.getScore() > itemMap.get(itemId).getScore()) {
                    itemMap.put(itemId, item);
                }
            }
        }
        
        // 按分数排序
        return itemMap.values().stream()
            .sorted(Comparator.comparingDouble(RecallItem::getScore).reversed())
            .collect(Collectors.toList());
    }
}

/**
 * Item-CF召回通道
 * 
 * 知识点应用：
 * 1. 协同过滤召回
 * 2. 倒排索引查询
 * 3. 热门降权
 */
@Component
public class ItemCFRecallChannel implements RecallChannel {
    
    @Resource
    private RedisTemplate<String, String> redisTemplate;
    
    // 倒排索引Key前缀
    private static final String ITEM_CF_INDEX_PREFIX = "item_cf:";
    
    @Override
    public ChannelResult recall(RecallContext context) {
        // 1. 获取用户最近点击的商品
        List<String> recentItems = context.getUserRecentItems();
        if (CollectionUtils.isEmpty(recentItems)) {
            return ChannelResult.empty();
        }
        
        // 2. 查询倒排索引，获取相似商品
        Map<String, Double> candidateScores = new HashMap<>();
        
        for (String itemId : recentItems) {
            // 从Redis获取相似商品列表
            String indexKey = ITEM_CF_INDEX_PREFIX + itemId;
            List<String> similarItems = redisTemplate.opsForList()
                .range(indexKey, 0, 100);  // 每个商品取Top100相似
            
            if (similarItems != null) {
                for (String similarItem : similarItems) {
                    // 解析分数
                    String[] parts = similarItem.split(":");
                    String candidateId = parts[0];
                    double score = Double.parseDouble(parts[1]);
                    
                    // 累加分数
                    candidateScores.merge(candidateId, score, Double::sum);
                }
            }
        }
        
        // 3. 热门降权
        candidateScores = applyPopularityPenalty(candidateScores);
        
        // 4. 排序返回
        List<RecallItem> items = candidateScores.entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .limit(500)  // 最多返回500个
            .map(e -> new RecallItem(e.getKey(), e.getValue(), "item_cf"))
            .collect(Collectors.toList());
        
        return new ChannelResult(items, "item_cf");
    }
    
    /**
     * 热门降权
     * 
     * 公式：score *= 1 / log(1 + popularity)
     * 目的：避免热门商品霸占召回结果
     */
    private Map<String, Double> applyPopularityPenalty(Map<String, Double> scores) {
        Map<String, Double> result = new HashMap<>();
        
        for (Map.Entry<String, Double> entry : scores.entrySet()) {
            String itemId = entry.getKey();
            double score = entry.getValue();
            
            // 获取商品热度
            int popularity = getItemPopularity(itemId);
            
            // 降权
            double penalty = 1.0 / Math.log(1 + popularity);
            result.put(itemId, score * penalty);
        }
        
        return result;
    }
}

/**
 * 向量召回通道
 * 
 * 知识点应用：
 * 1. 双塔DSSM模型
 * 2. ANN向量检索
 * 3. 实时向量更新
 */
@Component
public class VectorRecallChannel implements RecallChannel {
    
    @Resource
    private FaissIndex faissIndex;  // Faiss索引
    
    @Resource
    private FeatureService featureService;
    
    @Override
    public ChannelResult recall(RecallContext context) {
        // 1. 获取用户向量
        float[] userVector = getUserVector(context.getUserId());
        if (userVector == null) {
            return ChannelResult.empty();
        }
        
        // 2. ANN检索Top-K相似商品
        int topK = 500;
        SearchResult result = faissIndex.search(userVector, topK);
        
        // 3. 转换为召回结果
        List<RecallItem> items = new ArrayList<>();
        for (int i = 0; i < result.getIds().length; i++) {
            String itemId = String.valueOf(result.getIds()[i]);
            float score = result.getScores()[i];
            items.add(new RecallItem(itemId, score, "vector"));
        }
        
        return new ChannelResult(items, "vector");
    }
    
    /**
     * 获取用户向量
     * 
     * 策略：
     * 1. 实时计算：基于用户最近行为
     * 2. 缓存：缓存用户向量，减少计算
     */
    private float[] getUserVector(String userId) {
        // 1. 尝试从缓存获取
        String cacheKey = "user_vector:" + userId;
        float[] cachedVector = redisTemplate.opsForValue().get(cacheKey);
        if (cachedVector != null) {
            return cachedVector;
        }
        
        // 2. 实时计算用户向量
        // 获取用户最近点击的商品向量，取平均
        List<String> recentItems = featureService.getUserRecentItems(userId, 50);
        if (CollectionUtils.isEmpty(recentItems)) {
            return null;
        }
        
        List<float[]> itemVectors = new ArrayList<>();
        for (String itemId : recentItems) {
            float[] itemVector = featureService.getItemVector(itemId);
            if (itemVector != null) {
                itemVectors.add(itemVector);
            }
        }
        
        if (itemVectors.isEmpty()) {
            return null;
        }
        
        // 计算平均向量
        float[] userVector = new float[128];  // 向量维度128
        for (float[] vec : itemVectors) {
            for (int i = 0; i < vec.length; i++) {
                userVector[i] += vec[i];
            }
        }
        for (int i = 0; i < userVector.length; i++) {
            userVector[i] /= itemVectors.size();
        }
        
        // 3. 写入缓存（TTL 10分钟）
        redisTemplate.opsForValue().set(cacheKey, userVector, 10, TimeUnit.MINUTES);
        
        return userVector;
    }
}
```

### 17.1.4 精排服务实现

```java
/**
 * 精排服务 - 多目标排序
 * 
 * 知识点应用：
 * 1. 特征获取优化
 * 2. 模型批量推理
 * 3. 多目标融合
 */
@Service
public class RankingService {
    
    @Resource
    private FeatureService featureService;
    
    @Resource
    private ModelService modelService;
    
    /**
     * 精排入口
     * 
     * 性能要求：P99 < 20ms
     * 实现策略：批量特征获取 + 批量模型推理
     */
    public RankingResult rank(RankingRequest request) {
        long startTime = System.currentTimeMillis();
        
        List<RankingItem> items = request.getItems();
        if (CollectionUtils.isEmpty(items)) {
            return new RankingResult(Collections.emptyList(), 0);
        }
        
        // 1. 批量获取特征（关键优化点）
        FeatureBundle features = featureService.batchGetFeatures(
            request.getUserId(),
            items.stream().map(RankingItem::getItemId).collect(Collectors.toList())
        );
        
        // 2. 批量模型推理
        List<ModelOutput> outputs = modelService.batchPredict(features);
        
        // 3. 多目标融合
        List<RankingItem> rankedItems = new ArrayList<>();
        for (int i = 0; i < items.size(); i++) {
            RankingItem item = items.get(i);
            ModelOutput output = outputs.get(i);
            
            // 多目标融合分数
            double finalScore = fuseScores(output);
            
            item.setCtrScore(output.getCtrScore());
            item.setCvrScore(output.getCvrScore());
            item.setFinalScore(finalScore);
            
            rankedItems.add(item);
        }
        
        // 4. 排序
        rankedItems.sort(Comparator.comparingDouble(RankingItem::getFinalScore).reversed());
        
        // 5. 截断
        int topN = Math.min(request.getTopN(), rankedItems.size());
        rankedItems = rankedItems.subList(0, topN);
        
        long cost = System.currentTimeMillis() - startTime;
        MetricsUtil.recordLatency("ranking_latency", cost);
        
        return new RankingResult(rankedItems, cost);
    }
    
    /**
     * 多目标融合
     * 
     * 公式：score = pCTR^α × pCVR^β × price^γ
     * 
     * 参数调优：
     * α = 0.7（点击权重）
     * β = 0.3（转化权重）
     * γ = 0.1（价格权重）
     */
    private double fuseScores(ModelOutput output) {
        double ctrScore = output.getCtrScore();
        double cvrScore = output.getCvrScore();
        double price = output.getPrice();
        
        // 防止分数为0
        ctrScore = Math.max(ctrScore, 1e-6);
        cvrScore = Math.max(cvrScore, 1e-6);
        
        // 多目标融合
        double score = Math.pow(ctrScore, 0.7) 
                     * Math.pow(cvrScore, 0.3) 
                     * Math.pow(Math.log(1 + price), 0.1);
        
        return score;
    }
}

/**
 * 特征服务 - 批量特征获取
 * 
 * 知识点应用：
 * 1. Pipeline批量获取
 * 2. 特征缓存策略
 * 3. 降级处理
 */
@Service
public class FeatureService {
    
    @Resource
    private RedisClusterClient redisClient;
    
    /**
     * 批量获取特征
     * 
     * 性能要求：P99 < 5ms
     * 实现策略：Redis Pipeline + 本地缓存
     */
    public FeatureBundle batchGetFeatures(String userId, List<String> itemIds) {
        FeatureBundle bundle = new FeatureBundle();
        
        // 1. 获取用户特征（单个用户）
        Map<String, String> userFeatures = getUserFeatures(userId);
        bundle.setUserFeatures(userFeatures);
        
        // 2. 批量获取商品特征（Pipeline优化）
        Map<String, Map<String, String>> itemFeatures = batchGetItemFeatures(itemIds);
        bundle.setItemFeatures(itemFeatures);
        
        // 3. 获取上下文特征
        Map<String, String> contextFeatures = getContextFeatures();
        bundle.setContextFeatures(contextFeatures);
        
        return bundle;
    }
    
    /**
     * 批量获取商品特征 - Pipeline优化
     * 
     * 优化前：逐个查询，N次网络IO
     * 优化后：Pipeline批量查询，1次网络IO
     */
    private Map<String, Map<String, String>> batchGetItemFeatures(List<String> itemIds) {
        Map<String, Map<String, String>> result = new HashMap<>();
        
        // 使用Pipeline批量查询
        List<Response<String>> responses = new ArrayList<>();
        
        try (Jedis jedis = redisClient.getJedis()) {
            Pipeline pipeline = jedis.pipelined();
            
            for (String itemId : itemIds) {
                String key = "item_feature:" + itemId;
                responses.add(pipeline.get(key));
            }
            
            pipeline.sync();  // 执行批量查询
            
            // 解析结果
            for (int i = 0; i < itemIds.size(); i++) {
                String itemId = itemIds.get(i);
                String value = responses.get(i).get();
                
                if (value != null) {
                    Map<String, String> features = parseFeatures(value);
                    result.put(itemId, features);
                } else {
                    // 特征缺失，使用默认值
                    result.put(itemId, getDefaultFeatures());
                }
            }
        }
        
        return result;
    }
    
    /**
     * 获取用户特征
     * 
     * 特征类型：
     * 1. 静态特征：年龄、性别、城市等
     * 2. 统计特征：历史CTR、历史CVR等
     * 3. 行为特征：最近点击类目、最近搜索词等
     */
    private Map<String, String> getUserFeatures(String userId) {
        Map<String, String> features = new HashMap<>();
        
        // 1. 尝试从本地缓存获取
        Map<String, String> cachedFeatures = localCache.getIfPresent(userId);
        if (cachedFeatures != null) {
            return cachedFeatures;
        }
        
        // 2. 从Redis获取
        String key = "user_feature:" + userId;
        String value = redisClient.get(key);
        
        if (value != null) {
            features = parseFeatures(value);
        } else {
            // 用户特征缺失，使用默认值
            features = getDefaultUserFeatures();
        }
        
        // 3. 写入本地缓存（TTL 1分钟）
        localCache.put(userId, features);
        
        return features;
    }
}

/**
 * 模型服务 - TensorFlow Serving封装
 * 
 * 知识点应用：
 * 1. 模型批量推理
 * 2. 模型热更新
 * 3. 降级处理
 */
@Service
public class ModelService {
    
    @Resource
    private TensorFlowServingClient tfsClient;
    
    // 模型版本缓存
    private volatile String currentModelVersion;
    
    /**
     * 批量模型推理
     * 
     * 性能要求：P99 < 10ms
     * 实现策略：批量推理 + 模型缓存
     */
    public List<ModelOutput> batchPredict(FeatureBundle features) {
        // 1. 构建模型输入
        List<Map<String, Float>> instances = buildInstances(features);
        
        // 2. 批量推理
        PredictResponse response = tfsClient.predict(
            "ranking_model",           // 模型名称
            currentModelVersion,       // 模型版本
            instances                  // 批量输入
        );
        
        // 3. 解析输出
        List<ModelOutput> outputs = parseOutputs(response);
        
        return outputs;
    }
    
    /**
     * 构建模型输入
     * 
     * 特征处理：
     * 1. 连续特征：归一化
     * 2. 离散特征：Embedding查找
     * 3. 序列特征：Padding + Mask
     */
    private List<Map<String, Float>> buildInstances(FeatureBundle features) {
        List<Map<String, Float>> instances = new ArrayList<>();
        
        Map<String, String> userFeatures = features.getUserFeatures();
        Map<String, Map<String, String>> itemFeatures = features.getItemFeatures();
        Map<String, String> contextFeatures = features.getContextFeatures();
        
        for (String itemId : itemFeatures.keySet()) {
            Map<String, Float> instance = new HashMap<>();
            
            // 用户特征
            instance.put("user_age", normalizeAge(userFeatures.get("age")));
            instance.put("user_gender", encodeGender(userFeatures.get("gender")));
            instance.put("user_ctr", Float.parseFloat(userFeatures.getOrDefault("ctr", "0.05")));
            
            // 商品特征
            Map<String, String> itemFeat = itemFeatures.get(itemId);
            instance.put("item_price", normalizePrice(itemFeat.get("price")));
            instance.put("item_ctr", Float.parseFloat(itemFeat.getOrDefault("ctr", "0.03")));
            instance.put("item_category", encodeCategory(itemFeat.get("category")));
            
            // 上下文特征
            instance.put("hour_of_day", Float.parseFloat(contextFeatures.get("hour")));
            instance.put("day_of_week", Float.parseFloat(contextFeatures.get("weekday")));
            
            instances.add(instance);
        }
        
        return instances;
    }
    
    /**
     * 模型热更新
     * 
     * 流程：
     * 1. 监听模型版本变更
     * 2. 预热新模型
     * 3. 切换流量
     */
    @Scheduled(fixedDelay = 60000)  // 每分钟检查一次
    public void checkModelUpdate() {
        String latestVersion = getLatestModelVersion();
        
        if (!latestVersion.equals(currentModelVersion)) {
            log.info("发现新模型版本: {}, 当前版本: {}", latestVersion, currentModelVersion);
            
            // 预热新模型
            warmupModel(latestVersion);
            
            // 切换版本
            currentModelVersion = latestVersion;
            
            log.info("模型版本切换完成: {}", currentModelVersion);
        }
    }
}
```

### 17.1.5 性能优化实践

```
【优化过程】

问题1：P99延迟过高（从25ms涨到45ms）

排查过程：
├── 1. 分析调用链
│   └── 发现特征获取耗时占比最高（60%）
│
├── 2. 分析特征获取
│   ├── Redis查询次数多（每个商品单独查询）
│   └── 网络IO次数多
│
└── 3. 定位根因
    └── 未使用Pipeline批量查询

优化方案：
├── 使用Redis Pipeline批量查询
├── 增加本地缓存
└── 特征预取

优化效果：
├── 特征获取耗时：15ms → 3ms（降低80%）
└── 整体P99延迟：45ms → 22ms（降低51%）

---

问题2：高峰期QPS上不去

排查过程：
├── 1. 分析资源使用
│   ├── CPU使用率：40%
│   ├── 内存使用率：50%
│   └── 网络IO：正常
│
├── 2. 分析线程状态
│   └── 大量线程等待数据库响应
│
└── 3. 定位根因
    └── 同步调用导致线程阻塞

优化方案：
├── 异步化改造
│   ├── 召回并行化
│   └── 特征获取并行化
│
├── 增加线程池
│   └── 核心线程数：50 → 200
│
└── 增加连接池
    └── Redis连接数：20 → 100

优化效果：
├── 最大QPS：3万 → 6万（提升100%）
└── CPU使用率：40% → 75%（资源利用率提升）

---

问题3：内存占用过高

排查过程：
├── 1. 分析内存使用
│   ├── 堆内存：8GB
│   ├── 对象分布：特征对象占比最高
│   └── GC频率：Young GC每秒5次
│
├── 2. 分析对象创建
│   └── 每次请求创建大量临时对象
│
└── 3. 定位根因
    └── 特征对象未复用

优化方案：
├── 对象池化
│   └── 特征对象复用
│
├── 减少对象创建
│   └── 使用基本类型替代包装类
│
└── 调整JVM参数
    └── 新生代比例：1:1:8 → 1:1:4

优化效果：
├── 堆内存：8GB → 5GB（降低37.5%）
├── Young GC频率：5次/秒 → 2次/秒
└── GC停顿时间：50ms → 20ms
```

### 17.1.6 最终效果

```
【性能指标】

┌─────────────────┬──────────┬──────────┬──────────┐
│ 指标            │ 优化前   │ 优化后   │ 提升     │
├─────────────────┼──────────┼──────────┼──────────┤
│ P99延迟         │ 45ms     │ 22ms     │ -51%     │
│ 最大QPS         │ 3万      │ 6万      │ +100%    │
│ 可用性          │ 99.95%   │ 99.99%   │ +0.04%   │
│ CPU使用率       │ 40%      │ 75%      │ +35%     │
│ 内存使用        │ 8GB      │ 5GB      │ -37.5%   │
└─────────────────┴──────────┴──────────┴──────────┘

【业务指标】

┌─────────────────┬──────────┬──────────┬──────────┐
│ 指标            │ 优化前   │ 优化后   │ 提升     │
├─────────────────┼──────────┼──────────┼──────────┤
│ CTR             │ 3.2%     │ 3.5%     │ +9.4%    │
│ CVR             │ 2.1%     │ 2.3%     │ +9.5%    │
│ GMV             │ 1.8亿/天 │ 2.0亿/天 │ +11.1%   │
│ 人均停留时长    │ 12分钟   │ 14分钟   │ +16.7%   │
└─────────────────┴──────────┴──────────┴──────────┘
```

---

## 17.2 项目二：实时特征平台

### 17.2.1 项目背景

```
【业务背景】
├── 问题：特征更新延迟高（小时级）
├── 影响：推荐效果不及时
└── 目标：特征更新延迟降低到秒级

【系统目标】
├── 实时性：特征更新延迟 < 5秒
├── 吞吐量：支持100万QPS写入
├── 可靠性：数据不丢失
└── 一致性：最终一致性
```

### 17.2.2 系统架构

```
【实时特征平台架构】

┌─────────────────────────────────────────────────────────────────┐
│                      数据采集层                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ 用户行为 │  │ 商品变更 │  │ 订单事件 │  │ 搜索日志 │        │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘        │
└───────┼─────────────┼─────────────┼─────────────┼───────────────┘
        │             │             │             │
        └─────────────┴─────────────┴─────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      消息队列层                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Kafka Cluster                        │   │
│  │  Topic: user_behavior, item_change, order_event...      │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      实时计算层                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Flink Cluster                         │   │
│  │  ├── 实时特征计算                                        │   │
│  │  ├── 窗口聚合                                            │   │
│  │  └── 特征拼接                                            │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      存储层                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                     │
│  │  Redis   │  │  HBase   │  │  ES      │                     │
│  │ 热数据   │  │ 冷数据   │  │ 检索     │                     │
│  └──────────┘  └──────────┘  └──────────┘                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      服务层                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   特征服务                               │   │
│  │  ├── 特征查询接口                                        │   │
│  │  ├── 批量查询优化                                        │   │
│  │  └── 降级处理                                            │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 17.2.3 核心代码实现

```java
/**
 * 实时特征计算 - Flink作业
 * 
 * 知识点应用：
 * 1. 实时流处理
 * 2. 窗口聚合
 * 3. 状态管理
 */
public class RealtimeFeatureJob {
    
    public static void main(String[] args) throws Exception {
        // 1. 创建执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment
            .getExecutionEnvironment();
        env.setParallelism(100);
        
        // 2. Kafka数据源
        Properties kafkaProps = new Properties();
        kafkaProps.setProperty("bootstrap.servers", "kafka:9092");
        kafkaProps.setProperty("group.id", "feature-compute");
        
        FlinkKafkaConsumer<UserBehavior> source = new FlinkKafkaConsumer<>(
            "user_behavior",
            new UserBehaviorDeserializer(),
            kafkaProps
        );
        
        DataStream<UserBehavior> behaviorStream = env.addSource(source);
        
        // 3. 实时特征计算
        // 3.1 用户实时CTR（滑动窗口）
        DataStream<UserFeature> userCtrStream = behaviorStream
            .keyBy(UserBehavior::getUserId)
            .window(SlidingEventTimeWindows.of(
                Time.hours(1),    // 窗口大小1小时
                Time.minutes(5)   // 滑动步长5分钟
            ))
            .process(new UserCtrCalculator());
        
        // 3.2 商品实时CTR（滑动窗口）
        DataStream<ItemFeature> itemCtrStream = behaviorStream
            .keyBy(UserBehavior::getItemId)
            .window(SlidingEventTimeWindows.of(
                Time.hours(24),   // 窗口大小24小时
                Time.minutes(10)  // 滑动步长10分钟
            ))
            .process(new ItemCtrCalculator());
        
        // 4. 写入Redis
        RedisSink<UserFeature> userRedisSink = new RedisSink<>(
            new FlinkJedisPoolConfig.Builder()
                .setHost("redis")
                .setPort(6379)
                .build(),
            new UserFeatureRedisSink()
        );
        
        userCtrStream.addSink(userRedisSink);
        
        // 5. 执行作业
        env.execute("Realtime Feature Compute");
    }
}

/**
 * 用户CTR计算器
 * 
 * 计算逻辑：
 * CTR = 点击数 / 曝光数
 */
public class UserCtrCalculator extends ProcessWindowFunction<
    UserBehavior, UserFeature, String, TimeWindow> {
    
    @Override
    public void process(
        String userId,
        Context context,
        Iterable<UserBehavior> behaviors,
        Collector<UserFeature> out
    ) {
        long clickCount = 0;
        long exposeCount = 0;
        
        for (UserBehavior behavior : behaviors) {
            exposeCount++;
            if (behavior.isClick()) {
                clickCount++;
            }
        }
        
        // 计算CTR
        double ctr = exposeCount > 0 ? (double) clickCount / exposeCount : 0.0;
        
        // 平滑处理（防止极端值）
        ctr = smooth(ctr, exposeCount);
        
        UserFeature feature = new UserFeature();
        feature.setUserId(userId);
        feature.setCtr(ctr);
        feature.setClickCount(clickCount);
        feature.setExposeCount(exposeCount);
        feature.setTimestamp(System.currentTimeMillis());
        
        out.collect(feature);
    }
    
    /**
     * 平滑处理
     * 
     * 公式：ctr = (click + α) / (expose + α + β)
     * 
     * 目的：防止曝光数少时CTR波动大
     */
    private double smooth(double ctr, long exposeCount) {
        double alpha = 1.0;  // 贝叶斯平滑参数
        double beta = 20.0;
        
        return (ctr * exposeCount + alpha) / (exposeCount + alpha + beta);
    }
}

/**
 * 特征服务 - 实时特征查询
 * 
 * 知识点应用：
 * 1. 多级缓存
 * 2. 批量查询优化
 * 3. 降级处理
 */
@Service
public class RealtimeFeatureService {
    
    @Resource
    private RedisClusterClient redisClient;
    
    // 本地缓存
    private final Cache<String, Map<String, String>> localCache = 
        Caffeine.newBuilder()
            .maximumSize(100000)
            .expireAfterWrite(1, TimeUnit.MINUTES)
            .build();
    
    /**
     * 获取用户实时特征
     * 
     * 特征类型：
     * 1. 实时CTR（最近1小时）
     * 2. 实时CVR（最近24小时）
     * 3. 最近点击类目
     * 4. 最近搜索词
     */
    public Map<String, String> getUserRealtimeFeatures(String userId) {
        // 1. 尝试本地缓存
        Map<String, String> cached = localCache.getIfPresent(userId);
        if (cached != null) {
            return cached;
        }
        
        // 2. 从Redis获取
        Map<String, String> features = new HashMap<>();
        
        try {
            // 实时CTR
            String ctrKey = "user:ctr:" + userId;
            String ctr = redisClient.get(ctrKey);
            features.put("realtime_ctr", ctr != null ? ctr : "0.05");
            
            // 实时CVR
            String cvrKey = "user:cvr:" + userId;
            String cvr = redisClient.get(cvrKey);
            features.put("realtime_cvr", cvr != null ? cvr : "0.01");
            
            // 最近点击类目
            String categoryKey = "user:recent_category:" + userId;
            String categories = redisClient.get(categoryKey);
            features.put("recent_categories", categories != null ? categories : "");
            
            // 最近搜索词
            String searchKey = "user:recent_search:" + userId;
            String searches = redisClient.get(searchKey);
            features.put("recent_searches", searches != null ? searches : "");
            
        } catch (Exception e) {
            log.error("获取用户实时特征失败: {}", userId, e);
            // 降级：返回默认特征
            return getDefaultUserFeatures();
        }
        
        // 3. 写入本地缓存
        localCache.put(userId, features);
        
        return features;
    }
    
    /**
     * 批量获取商品实时特征
     * 
     * 优化：Pipeline批量查询
     */
    public Map<String, Map<String, String>> batchGetItemFeatures(List<String> itemIds) {
        Map<String, Map<String, String>> result = new HashMap<>();
        
        if (CollectionUtils.isEmpty(itemIds)) {
            return result;
        }
        
        try (Jedis jedis = redisClient.getJedis()) {
            Pipeline pipeline = jedis.pipelined();
            
            // 批量查询
            Map<String, Response<String>> responses = new HashMap<>();
            for (String itemId : itemIds) {
                String key = "item:realtime:" + itemId;
                responses.put(itemId, pipeline.get(key));
            }
            
            pipeline.sync();
            
            // 解析结果
            for (Map.Entry<String, Response<String>> entry : responses.entrySet()) {
                String itemId = entry.getKey();
                String value = entry.getValue().get();
                
                if (value != null) {
                    result.put(itemId, parseFeatures(value));
                } else {
                    result.put(itemId, getDefaultItemFeatures());
                }
            }
            
        } catch (Exception e) {
            log.error("批量获取商品特征失败", e);
            // 降级：返回默认特征
            for (String itemId : itemIds) {
                result.put(itemId, getDefaultItemFeatures());
            }
        }
        
        return result;
    }
}
```

### 17.2.4 性能优化

```
【优化过程】

问题1：特征更新延迟高（从秒级变成分钟级）

排查过程：
├── 1. 分析Flink作业
│   ├── 背压严重
│   └── 窗口计算慢
│
├── 2. 分析窗口计算
│   └── 窗口内数据量大
│
└── 3. 定位根因
    └── 窗口大小设置不合理

优化方案：
├── 调整窗口大小
│   ├── 用户CTR：1小时 → 30分钟
│   └── 商品CTR：24小时 → 6小时
│
├── 增加并行度
│   └── 并行度：50 → 200
│
└── 优化状态后端
    └── 使用RocksDB状态后端

优化效果：
├── 特征更新延迟：3分钟 → 3秒（降低99%）
└── 背压：消除

---

问题2：Redis写入压力大

排查过程：
├── 1. 分析写入模式
│   └── 每个特征单独写入
│
├── 2. 分析写入频率
│   └── 每秒10万次写入
│
└── 3. 定位根因
    └── 未使用Pipeline批量写入

优化方案：
├── Pipeline批量写入
│   └── 每100条批量写入一次
│
├── 本地聚合
│   └── 先在Flink本地聚合，再写入Redis
│
└── 增加Redis分片
    └── 分片数：16 → 64

优化效果：
├── Redis写入QPS：10万 → 2万（降低80%）
└── Redis CPU使用率：80% → 30%
```

### 17.2.5 最终效果

```
【性能指标】

┌─────────────────┬──────────┬──────────┬──────────┐
│ 指标            │ 优化前   │ 优化后   │ 提升     │
├─────────────────┼──────────┼──────────┼──────────┤
│ 特征更新延迟    │ 3分钟    │ 3秒      │ -99%     │
│ 写入吞吐量      │ 10万QPS  │ 100万QPS │ +900%    │
│ 查询延迟        │ 5ms      │ 1ms      │ -80%     │
│ 数据可靠性      │ 99.9%    │ 99.99%   │ +0.09%   │
└─────────────────┴──────────┴──────────┴──────────┘

【业务效果】

┌─────────────────┬──────────┬──────────┬──────────┐
│ 指标            │ 优化前   │ 优化后   │ 提升     │
├─────────────────┼──────────┼──────────┼──────────┤
│ CTR             │ 3.0%     │ 3.3%     │ +10%     │
│ 新品曝光率      │ 15%      │ 25%      │ +67%     │
│ 实时性感知      │ 低       │ 高       │ -        │
└─────────────────┴──────────┴──────────┴──────────┘
```

---

## 17.3 项目三：A/B实验平台

### 17.3.1 项目背景

```
【业务背景】
├── 问题：算法迭代效率低
│   ├── 实验周期长（2周）
│   ├── 实验分析慢
│   └── 决策依赖人工
│
├── 影响：算法迭代速度慢
│
└── 目标：实验周期缩短到3天

【系统目标】
├── 实验效率：实验周期 < 3天
├── 实验能力：支持100+并行实验
├── 分析能力：自动分析 + 报告生成
└── 决策能力：自动决策建议
```

### 17.3.2 系统架构

```
【A/B实验平台架构】

┌─────────────────────────────────────────────────────────────────┐
│                      实验管理层                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                     │
│  │ 实验配置 │  │ 流量分配 │  │ 实验监控 │                     │
│  └──────────┘  └──────────┘  └──────────┘                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      分流层                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   分流服务                               │   │
│  │  ├── 用户分流                                            │   │
│  │  ├── 请求分流                                            │   │
│  │  └── 正交分流                                            │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      实验执行层                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │ 实验组A  │  │ 实验组B  │  │ 实验组C  │  │ 对照组   │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      数据分析层                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                     │
│  │ 指标计算 │  │ 显著检验 │  │ 报告生成 │                     │
│  └──────────┘  └──────────┘  └──────────┘                     │
└─────────────────────────────────────────────────────────────────┘
```

### 17.3.3 核心代码实现

```java
/**
 * 分流服务
 * 
 * 知识点应用：
 * 1. 分层正交实验
 * 2. 一致性哈希
 * 3. 流量分配
 */
@Service
public class ExperimentService {
    
    @Resource
    private ExperimentConfigService configService;
    
    /**
     * 获取用户实验分组
     * 
     * 分流策略：
     * 1. 分层正交：不同层实验正交
     * 2. 一致性：同一用户始终分到同一组
     * 3. 均匀性：各组流量均匀
     */
    public ExperimentGroup getExperimentGroup(String userId, String layerId) {
        // 1. 获取该层的实验配置
        List<ExperimentConfig> configs = configService.getLayerConfigs(layerId);
        
        // 2. 计算用户桶号
        int bucketCount = 1000;  // 总桶数
        int bucket = getUserBucket(userId, layerId, bucketCount);
        
        // 3. 匹配实验组
        for (ExperimentConfig config : configs) {
            if (isInExperiment(bucket, config)) {
                return new ExperimentGroup(
                    config.getExperimentId(),
                    config.getGroupName(),
                    config.getVariables()
                );
            }
        }
        
        // 4. 未命中任何实验，返回对照组
        return ExperimentGroup.control();
    }
    
    /**
     * 计算用户桶号
     * 
     * 算法：MurmurHash3 + 取模
     * 
     * 特点：
     * 1. 分布均匀
     * 2. 计算快速
     * 3. 一致性好
     */
    private int getUserBucket(String userId, String layerId, int bucketCount) {
        // 拼接key
        String key = userId + "_" + layerId;
        
        // MurmurHash3计算hash值
        int hash = MurmurHash3.hash32(key);
        
        // 取模得到桶号
        return Math.abs(hash) % bucketCount;
    }
    
    /**
     * 判断用户是否在实验中
     */
    private boolean isInExperiment(int bucket, ExperimentConfig config) {
        int startBucket = config.getStartBucket();
        int endBucket = config.getEndBucket();
        
        return bucket >= startBucket && bucket < endBucket;
    }
}

/**
 * 实验分析服务
 * 
 * 知识点应用：
 * 1. 统计显著性检验
 * 2. 效应量计算
 * 3. 自动报告生成
 */
@Service
public class ExperimentAnalysisService {
    
    /**
     * 分析实验结果
     * 
     * 分析内容：
     * 1. 指标对比
     * 2. 显著性检验
     * 3. 效应量计算
     * 4. 决策建议
     */
    public ExperimentReport analyzeExperiment(String experimentId) {
        // 1. 获取实验数据
        ExperimentData data = getExperimentData(experimentId);
        
        // 2. 计算各指标
        Map<String, MetricResult> metrics = calculateMetrics(data);
        
        // 3. 显著性检验
        Map<String, SignificanceResult> significance = testSignificance(data);
        
        // 4. 生成报告
        ExperimentReport report = new ExperimentReport();
        report.setExperimentId(experimentId);
        report.setMetrics(metrics);
        report.setSignificance(significance);
        report.setRecommendation(generateRecommendation(significance));
        
        return report;
    }
    
    /**
     * 显著性检验
     * 
     * 方法：双样本t检验
     * 
     * 判断标准：
     * 1. p-value < 0.05：显著
     * 2. 置信区间不包含0：显著
     */
    private Map<String, SignificanceResult> testSignificance(ExperimentData data) {
        Map<String, SignificanceResult> results = new HashMap<>();
        
        for (String metric : data.getMetrics()) {
            // 获取对照组和实验组数据
            List<Double> controlValues = data.getControlValues(metric);
            List<Double> experimentValues = data.getExperimentValues(metric);
            
            // 计算均值
            double controlMean = mean(controlValues);
            double experimentMean = mean(experimentValues);
            
            // 计算标准差
            double controlStd = std(controlValues);
            double experimentStd = std(experimentValues);
            
            // t检验
            double tValue = calculateTValue(
                controlMean, experimentMean,
                controlStd, experimentStd,
                controlValues.size(), experimentValues.size()
            );
            
            // 计算p值
            double pValue = calculatePValue(tValue, 
                controlValues.size() + experimentValues.size() - 2);
            
            // 计算置信区间
            double[] confidenceInterval = calculateConfidenceInterval(
                controlMean, experimentMean,
                controlStd, experimentStd,
                controlValues.size(), experimentValues.size()
            );
            
            // 计算效应量（Cohen's d）
            double cohensD = calculateCohensD(
                controlMean, experimentMean,
                controlStd, experimentStd
            );
            
            SignificanceResult result = new SignificanceResult();
            result.setMetric(metric);
            result.setControlMean(controlMean);
            result.setExperimentMean(experimentMean);
            result.setLift((experimentMean - controlMean) / controlMean);
            result.setPValue(pValue);
            result.setConfidenceInterval(confidenceInterval);
            result.setCohensD(cohensD);
            result.setSignificant(pValue < 0.05);
            
            results.put(metric, result);
        }
        
        return results;
    }
    
    /**
     * 生成决策建议
     */
    private String generateRecommendation(Map<String, SignificanceResult> significance) {
        // 检查核心指标
        SignificanceResult ctrResult = significance.get("ctr");
        SignificanceResult gmvResult = significance.get("gmv");
        
        if (ctrResult == null || gmvResult == null) {
            return "数据不足，建议继续实验";
        }
        
        // 判断是否显著正向
        boolean ctrPositive = ctrResult.isSignificant() && ctrResult.getLift() > 0;
        boolean gmvPositive = gmvResult.isSignificant() && gmvResult.getLift() > 0;
        
        if (ctrPositive && gmvPositive) {
            return "建议全量发布：CTR提升" + formatPercent(ctrResult.getLift()) + 
                   "，GMV提升" + formatPercent(gmvResult.getLift());
        } else if (ctrPositive && !gmvPositive) {
            return "建议继续观察：CTR提升但GMV未显著提升";
        } else if (!ctrPositive && gmvPositive) {
            return "建议继续观察：GMV提升但CTR未显著提升";
        } else {
            return "建议停止实验：核心指标无显著提升";
        }
    }
}
```

### 17.3.4 最终效果

```
【效率指标】

┌─────────────────┬──────────┬──────────┬──────────┐
│ 指标            │ 优化前   │ 优化后   │ 提升     │
├─────────────────┼──────────┼──────────┼──────────┤
│ 实验周期        │ 14天     │ 3天      │ -79%     │
│ 并行实验数      │ 10个     │ 100个    │ +900%    │
│ 分析耗时        │ 2天      │ 1小时    │ -98%     │
│ 决策效率        │ 人工     │ 自动     │ -        │
└─────────────────┴──────────┴──────────┴──────────┘

【业务效果】

├── 算法迭代速度提升5倍
├── 实验决策准确率95%+
└── 年节省人力成本500万+
```

---

## 17.4 项目经验总结

### 17.4.1 技术选型经验

```
【技术选型原则】

1. 成熟优先
   ├── 优先选择成熟的开源组件
   ├── 避免踩坑
   └── 社区支持好

2. 适度超前
   ├── 不追求最新技术
   ├── 选择稳定版本
   └── 预留升级空间

3. 团队能力
   ├── 考虑团队技术栈
   ├── 学习成本可控
   └── 招聘容易

【推荐技术栈】

┌─────────────────┬─────────────────────────────────┐
│ 场景            │ 推荐技术                        │
├─────────────────┼─────────────────────────────────┤
│ 编程语言        │ Java（主流）、C++（高性能）     │
│ 微服务框架      │ Spring Cloud、Dubbo             │
│ 消息队列        │ Kafka（日志）、RocketMQ（业务） │
│ 缓存            │ Redis Cluster                   │
│ 数据库          │ MySQL（关系）、MongoDB（文档）  │
│ 搜索引擎        │ Elasticsearch                   │
│ 向量检索        │ Faiss、Milvus                   │
│ 实时计算        │ Flink                           │
│ 模型服务        │ TensorFlow Serving              │
│ 监控            │ Prometheus + Grafana            │
│ 容器化          │ Docker + Kubernetes             │
└─────────────────┴─────────────────────────────────┘
```

### 17.4.2 性能优化经验

```
【性能优化方法论】

1. 先监控，后优化
   ├── 建立完善的监控体系
   ├── 发现真正的瓶颈
   └── 避免盲目优化

2. 先架构，后代码
   ├── 架构优化收益大
   ├── 代码优化收益小
   └── 优先考虑架构层面

3. 先瓶颈，后全局
   ├── 找到真正的瓶颈
   ├── 集中精力优化
   └── 避免过度优化

【常见优化技巧】

┌─────────────────┬─────────────────────────────────┐
│ 优化方向        │ 具体技巧                        │
├─────────────────┼─────────────────────────────────┤
│ 减少IO          │ Pipeline、批量、缓存            │
│ 减少计算        │ 预计算、增量计算                │
│ 并行化          │ 多线程、异步、协程              │
│ 资源优化        │ 连接池、对象池、内存管理        │
│ 算法优化        │ 更好的算法、数据结构            │
└─────────────────┴─────────────────────────────────┘
```

### 17.4.3 故障处理经验

```
【故障处理原则】

1. 快速止损优先
   ├── 先恢复服务
   ├── 再定位原因
   └── 最后彻底修复

2. 充分准备预案
   ├── 回滚方案
   ├── 降级方案
   └── 应急预案

3. 完善监控告警
   ├── 及时发现
   ├── 快速定位
   └── 自动处理

【常见故障模式】

┌─────────────────┬─────────────────────────────────┐
│ 故障类型        │ 处理方法                        │
├─────────────────┼─────────────────────────────────┤
│ 服务超时        │ 超时控制、降级、熔断            │
│ 内存溢出        │ 扩容、内存优化、重启            │
│ 数据库慢        │ 慢查询优化、读写分离、缓存      │
│ 缓存击穿        │ 热点预热、互斥锁、永不过期      │
│ 流量突增        │ 限流、扩容、降级                │
└─────────────────┴─────────────────────────────────┘
```

---

[← 上一章：推荐后端工程师工作内容详解](16-backend-engineer-work.md) | [返回目录](../README.md)
