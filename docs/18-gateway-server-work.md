# 十八、推荐场景下网关与服务端工作详解

> 详细讲解推荐系统中网关层和服务端层的职责划分、工作内容、技术实现。

---

## 18.1 整体架构视角

### 18.1.1 推荐系统分层架构

```
【推荐系统请求链路】

┌─────────────────────────────────────────────────────────────────┐
│                        用户端（App/Web）                        │
│  ├── 用户打开App                                              │
│  ├── 发起推荐请求                                             │
│  └── 接收推荐结果                                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ HTTP请求
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        网关层（Gateway）                        │
│  【职责：流量入口、安全、路由、限流】                          │
│  ├── 请求鉴权                                                  │
│  ├── 流量控制                                                  │
│  ├── 路由分发                                                  │
│  ├── 灰度分流                                                  │
│  ├── 监控埋点                                                  │
│  └── 日志记录                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ RPC调用
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        服务端（Server）                         │
│  【职责：业务逻辑、推荐计算、数据处理】                        │
│  ├── 召回服务                                                  │
│  ├── 排序服务                                                  │
│  ├── 重排服务                                                  │
│  ├── 特征服务                                                  │
│  └── 模型服务                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ 数据访问
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        数据层（Data）                           │
│  ├── Redis（缓存）                                            │
│  ├── MySQL（数据库）                                          │
│  ├── ES（检索）                                               │
│  └── Kafka（消息队列）                                        │
└─────────────────────────────────────────────────────────────────┘
```

### 18.1.2 职责划分对比

```
┌─────────────────┬─────────────────────────────┬─────────────────────────────┐
│ 维度            │ 网关层                      │ 服务端                      │
├─────────────────┼─────────────────────────────┼─────────────────────────────┤
│ 核心职责        │ 流量管理、安全防护          │ 业务逻辑、推荐计算          │
│ 处理内容        │ 请求转发、流量控制          │ 召回、排序、特征处理        │
│ 性能要求        │ 极高（P99<5ms）             │ 高（P99<30ms）              │
│ 复杂度          │ 相对简单                    │ 复杂                        │
│ 可复用性        │ 高（多业务共用）            │ 低（业务特定）              │
│ 变更频率        │ 低                          │ 高                          │
│ 技术栈          │ Nginx/OpenResty/Kong        │ Java/C++/Go                 │
└─────────────────┴─────────────────────────────┴─────────────────────────────┘
```

---

## 18.2 网关层工作详解

### 18.2.1 网关层核心职责

```
【网关层六大核心职责】

┌─────────────────────────────────────────────────────────────────┐
│  1. 请求鉴权                                                    │
│  ├── 验证用户身份（Token验证）                                 │
│  ├── 验证请求合法性                                            │
│  └── 过滤非法请求                                              │
├─────────────────────────────────────────────────────────────────┤
│  2. 流量控制                                                    │
│  ├── 限流（防止系统过载）                                      │
│  ├── 熔断（防止雪崩）                                          │
│  └── 降级（保护核心服务）                                      │
├─────────────────────────────────────────────────────────────────┤
│  3. 路由分发                                                    │
│  ├── 请求路由到正确的服务                                      │
│  ├── 负载均衡                                                  │
│  └── 服务发现                                                  │
├─────────────────────────────────────────────────────────────────┤
│  4. 灰度分流                                                    │
│  ├── A/B实验分流                                               │
│  ├── 灰度发布分流                                              │
│  └── 流量染色                                                  │
├─────────────────────────────────────────────────────────────────┤
│  5. 监控埋点                                                    │
│  ├── 请求日志记录                                              │
│  ├── 性能指标采集                                              │
│  └── 异常监控                                                  │
├─────────────────────────────────────────────────────────────────┤
│  6. 协议转换                                                    │
│  ├── HTTP → RPC                                                │
│  ├── 协议适配                                                  │
│  └── 数据格式转换                                              │
└─────────────────────────────────────────────────────────────────┘
```

### 18.2.2 网关层工作流程

```
【网关处理请求流程】

用户请求
    │
    ▼
┌──────────────┐
│ 1. 接收请求   │ ← 接收HTTP请求
└──────────────┘
    │
    ▼
┌──────────────┐
│ 2. 请求解析   │ ← 解析请求头、请求体
└──────────────┘
    │
    ▼
┌──────────────┐
│ 3. 鉴权验证   │ ← 验证Token、用户身份
└──────────────┘
    │
    ├── 鉴权失败 → 返回401
    │
    ▼
┌──────────────┐
│ 4. 限流检查   │ ← 检查是否超过限流阈值
└──────────────┘
    │
    ├── 超过限流 → 返回429
    │
    ▼
┌──────────────┐
│ 5. 灰度分流   │ ← 根据用户ID分流到不同版本
└──────────────┘
    │
    ▼
┌──────────────┐
│ 6. 路由选择   │ ← 选择目标服务实例
└──────────────┘
    │
    ▼
┌──────────────┐
│ 7. 协议转换   │ ← HTTP转RPC
└──────────────┘
    │
    ▼
┌──────────────┐
│ 8. 转发请求   │ ← 调用后端服务
└──────────────┘
    │
    ▼
┌──────────────┐
│ 9. 响应处理   │ ← 处理响应、记录日志
└──────────────┘
    │
    ▼
返回给用户
```

### 18.2.3 网关层代码示例

```lua
-- OpenResty网关配置示例

-- 1. 鉴权过滤器
function auth_filter()
    -- 获取Token
    local token = ngx.req.get_headers()["Authorization"]
    
    if not token then
        ngx.status = 401
        ngx.say('{"code": 401, "msg": "未授权"}')
        ngx.exit(401)
    end
    
    -- 验证Token
    local user_id = verify_token(token)
    if not user_id then
        ngx.status = 401
        ngx.say('{"code": 401, "msg": "Token无效"}')
        ngx.exit(401)
    end
    
    -- 存储用户ID，传递给后端
    ngx.ctx.user_id = user_id
end

-- 2. 限流过滤器
function rate_limit_filter()
    local user_id = ngx.ctx.user_id
    local limit_key = "rate_limit:" .. user_id
    
    -- 使用Redis进行限流
    local redis = require "resty.redis"
    local red = redis:new()
    
    local count, err = red:incr(limit_key)
    if count == 1 then
        red:expire(limit_key, 1)  -- 1秒过期
    end
    
    -- 每秒最多100次请求
    if count > 100 then
        ngx.status = 429
        ngx.say('{"code": 429, "msg": "请求过于频繁"}')
        ngx.exit(429)
    end
end

-- 3. 灰度分流
function gray_routing()
    local user_id = ngx.ctx.user_id
    
    -- 计算用户桶号
    local bucket = crc32(user_id) % 100
    
    -- 灰度规则：10%流量到新版本
    if bucket < 10 then
        ngx.ctx.service_version = "v2"
    else
        ngx.ctx.service_version = "v1"
    end
end

-- 4. 路由选择
function route_select()
    local version = ngx.ctx.service_version
    
    -- 根据版本选择服务
    local service_map = {
        v1 = "recommend-service-v1:8080",
        v2 = "recommend-service-v2:8080"
    }
    
    ngx.ctx.upstream = service_map[version]
end

-- 5. 监控埋点
function monitor_logging()
    local start_time = ngx.req.start_time()
    local latency = ngx.now() - start_time
    
    -- 记录访问日志
    local log_data = {
        user_id = ngx.ctx.user_id,
        uri = ngx.var.uri,
        latency = latency,
        status = ngx.status,
        timestamp = ngx.now()
    }
    
    -- 异步写入日志
    ngx.log(ngx.INFO, cjson.encode(log_data))
end

-- 主处理流程
function main()
    auth_filter()        -- 鉴权
    rate_limit_filter()  -- 限流
    gray_routing()       -- 灰度
    route_select()       -- 路由
    -- 转发到后端服务
    monitor_logging()    -- 监控
end
```

### 18.2.4 网关层关键指标

```
【网关层性能指标】

┌─────────────────┬────────────┬─────────────────────────────────┐
│ 指标            │ 目标值     │ 说明                            │
├─────────────────┼────────────┼─────────────────────────────────┤
│ P99延迟         │ < 5ms      │ 网关处理延迟                    │
│ QPS             │ > 50万     │ 单机处理能力                    │
│ 可用性          │ > 99.99%   │ 网关可用性                      │
│ 错误率          │ < 0.01%    │ 请求错误率                      │
│ CPU使用率       │ < 50%      │ 资源使用                        │
│ 内存使用        │ < 2GB      │ 内存占用                        │
└─────────────────┴────────────┴─────────────────────────────────┘

【网关层业务指标】

┌─────────────────┬─────────────────────────────────────────────┐
│ 指标            │ 说明                                        │
├─────────────────┼─────────────────────────────────────────────┤
│ 鉴权成功率      │ Token验证通过率                             │
│ 限流触发率      │ 被限流的请求比例                            │
│ 灰度分流准确率  │ 灰度分流的准确性                            │
│ 路由成功率      │ 请求路由成功率                              │
└─────────────────┴─────────────────────────────────────────────┘
```

---

## 18.3 服务端工作详解

### 18.3.1 服务端核心职责

```
【服务端六大核心职责】

┌─────────────────────────────────────────────────────────────────┐
│  1. 召回服务                                                    │
│  ├── 多路召回调度                                              │
│  ├── 倒排索引查询                                              │
│  ├── 向量检索                                                  │
│  └── 候选集合并去重                                            │
├─────────────────────────────────────────────────────────────────┤
│  2. 排序服务                                                    │
│  ├── 特征获取                                                  │
│  ├── 模型推理                                                  │
│  ├── 多目标融合                                                │
│  └── 排序输出                                                  │
├─────────────────────────────────────────────────────────────────┤
│  3. 重排服务                                                    │
│  ├── 多样性处理                                                │
│  ├── 业务规则过滤                                              │
│  ├── 去重处理                                                  │
│  └── 最终排序                                                  │
├─────────────────────────────────────────────────────────────────┤
│  4. 特征服务                                                    │
│  ├── 用户特征获取                                              │
│  ├── 商品特征获取                                              │
│  ├── 实时特征更新                                              │
│  └── 特征缓存管理                                              │
├─────────────────────────────────────────────────────────────────┤
│  5. 模型服务                                                    │
│  ├── 模型加载                                                  │
│  ├── 批量推理                                                  │
│  ├── 模型热更新                                                │
│  └── 模型版本管理                                              │
├─────────────────────────────────────────────────────────────────┤
│  6. 数据服务                                                    │
│  ├── 用户画像                                                  │
│  ├── 商品信息                                                  │
│  ├── 行为日志                                                  │
│  └── 实验数据                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 18.3.2 服务端工作流程

```
【推荐服务处理流程】

网关转发请求
    │
    ▼
┌──────────────┐
│ 1. 请求解析   │ ← 解析请求参数
└──────────────┘
    │
    ▼
┌──────────────┐
│ 2. 召回阶段   │ ← 从海量商品中筛选候选集
│   ├── Item-CF│    （500-2000个）
│   ├── 向量   │
│   └── 图召回 │
└──────────────┘
    │
    ▼
┌──────────────┐
│ 3. 粗排阶段   │ ← 轻量模型快速打分
└──────────────┘    （筛选到200-500个）
    │
    ▼
┌──────────────┐
│ 4. 精排阶段   │ ← 复杂模型精准打分
│   ├── 特征获取│   （排序输出50-100个）
│   ├── 模型推理│
│   └── 多目标 │
└──────────────┘
    │
    ▼
┌──────────────┐
│ 5. 重排阶段   │ ← 业务规则处理
│   ├── 多样性 │   （最终输出10-20个）
│   ├── 去重   │
│   └── 规则   │
└──────────────┘
    │
    ▼
┌──────────────┐
│ 6. 结果组装   │ ← 组装响应数据
└──────────────┘
    │
    ▼
返回给网关
```

### 18.3.3 服务端代码示例

```java
/**
 * 推荐服务端 - 主服务
 * 
 * 职责：协调召回、排序、重排各阶段
 */
@Service
public class RecommendService {
    
    @Resource
    private RecallService recallService;
    
    @Resource
    private RankingService rankingService;
    
    @Resource
    private RerankService rerankService;
    
    /**
     * 推荐主流程
     * 
     * 性能要求：P99 < 30ms
     */
    public RecommendResponse recommend(RecommendRequest request) {
        long startTime = System.currentTimeMillis();
        
        // 1. 召回阶段（P99 < 15ms）
        RecallResult recallResult = recallService.recall(
            RecallRequest.builder()
                .userId(request.getUserId())
                .scene(request.getScene())
                .recallSize(1000)
                .build()
        );
        
        // 2. 粗排阶段（P99 < 5ms）
        RankingResult coarseResult = rankingService.coarseRank(
            CoarseRankingRequest.builder()
                .items(recallResult.getItems())
                .topN(200)
                .build()
        );
        
        // 3. 精排阶段（P99 < 10ms）
        RankingResult fineResult = rankingService.fineRank(
            FineRankingRequest.builder()
                .userId(request.getUserId())
                .items(coarseResult.getItems())
                .topN(50)
                .build()
        );
        
        // 4. 重排阶段（P99 < 3ms）
        RerankResult rerankResult = rerankService.rerank(
            RerankRequest.builder()
                .items(fineResult.getItems())
                .scene(request.getScene())
                .topN(20)
                .build()
        );
        
        // 5. 组装响应
        RecommendResponse response = new RecommendResponse();
        response.setItems(rerankResult.getItems());
        response.setRequestId(request.getRequestId());
        
        // 6. 记录监控
        long cost = System.currentTimeMillis() - startTime;
        MetricsUtil.recordLatency("recommend_total_latency", cost);
        
        return response;
    }
}

/**
 * 召回服务
 * 
 * 职责：从海量商品中筛选候选集
 */
@Service
public class RecallService {
    
    @Resource
    private List<RecallChannel> channels;
    
    /**
     * 多路召回
     */
    public RecallResult recall(RecallRequest request) {
        // 并行调度各路召回
        List<CompletableFuture<ChannelResult>> futures = channels.stream()
            .map(channel -> CompletableFuture.supplyAsync(
                () -> channel.recall(request),
                recallExecutor
            ))
            .collect(Collectors.toList());
        
        // 等待结果
        List<ChannelResult> results = futures.stream()
            .map(f -> f.get(100, TimeUnit.MILLISECONDS))
            .collect(Collectors.toList());
        
        // 合并去重
        List<RecallItem> merged = mergeAndDeduplicate(results);
        
        return new RecallResult(merged);
    }
}

/**
 * 排序服务
 * 
 * 职责：对候选商品打分排序
 */
@Service
public class RankingService {
    
    @Resource
    private FeatureService featureService;
    
    @Resource
    private ModelService modelService;
    
    /**
     * 精排
     */
    public RankingResult fineRank(FineRankingRequest request) {
        List<RankingItem> items = request.getItems();
        
        // 1. 批量获取特征
        FeatureBundle features = featureService.batchGetFeatures(
            request.getUserId(),
            items.stream().map(RankingItem::getItemId).collect(Collectors.toList())
        );
        
        // 2. 批量模型推理
        List<ModelOutput> outputs = modelService.batchPredict(features);
        
        // 3. 多目标融合
        for (int i = 0; i < items.size(); i++) {
            RankingItem item = items.get(i);
            ModelOutput output = outputs.get(i);
            
            // 多目标融合分数
            double score = Math.pow(output.getCtrScore(), 0.7) 
                         * Math.pow(output.getCvrScore(), 0.3);
            
            item.setScore(score);
        }
        
        // 4. 排序
        items.sort(Comparator.comparingDouble(RankingItem::getScore).reversed());
        
        // 5. 截断
        return new RankingResult(items.subList(0, request.getTopN()));
    }
}

/**
 * 特征服务
 * 
 * 职责：高效获取用户、商品特征
 */
@Service
public class FeatureService {
    
    @Resource
    private RedisClusterClient redisClient;
    
    /**
     * 批量获取特征
     * 
     * 优化：Pipeline批量查询
     */
    public FeatureBundle batchGetFeatures(String userId, List<String> itemIds) {
        FeatureBundle bundle = new FeatureBundle();
        
        // 1. 获取用户特征
        bundle.setUserFeatures(getUserFeatures(userId));
        
        // 2. 批量获取商品特征（Pipeline优化）
        bundle.setItemFeatures(batchGetItemFeatures(itemIds));
        
        return bundle;
    }
    
    /**
     * Pipeline批量获取商品特征
     */
    private Map<String, Map<String, String>> batchGetItemFeatures(List<String> itemIds) {
        Map<String, Map<String, String>> result = new HashMap<>();
        
        try (Jedis jedis = redisClient.getJedis()) {
            Pipeline pipeline = jedis.pipelined();
            
            // 批量查询
            List<Response<String>> responses = new ArrayList<>();
            for (String itemId : itemIds) {
                responses.add(pipeline.get("item_feature:" + itemId));
            }
            
            pipeline.sync();
            
            // 解析结果
            for (int i = 0; i < itemIds.size(); i++) {
                String value = responses.get(i).get();
                result.put(itemIds.get(i), parseFeatures(value));
            }
        }
        
        return result;
    }
}
```

### 18.3.4 服务端关键指标

```
【服务端性能指标】

┌─────────────────┬────────────┬─────────────────────────────────┐
│ 服务            │ P99延迟    │ 说明                            │
├─────────────────┼────────────┼─────────────────────────────────┤
│ 召回服务        │ < 15ms     │ 多路召回                        │
│ 粗排服务        │ < 5ms      │ 轻量模型打分                    │
│ 精排服务        │ < 10ms     │ 复杂模型打分                    │
│ 重排服务        │ < 3ms      │ 业务规则处理                    │
│ 特征服务        │ < 5ms      │ 特征获取                        │
│ 模型服务        │ < 8ms      │ 模型推理                        │
├─────────────────┼────────────┼─────────────────────────────────┤
│ 整体            │ < 30ms     │ 完整推荐流程                    │
└─────────────────┴────────────┴─────────────────────────────────┘

【服务端业务指标】

┌─────────────────┬────────────┬─────────────────────────────────┐
│ 指标            │ 目标值     │ 说明                            │
├─────────────────┼────────────┼─────────────────────────────────┤
│ 召回覆盖率      │ > 70%      │ 召回的商品覆盖率                │
│ 排序AUC        │ > 0.73     │ 排序模型效果                    │
│ 特征命中率      │ > 95%      │ 特征缓存命中率                  │
│ 模型推理成功率  │ > 99.9%    │ 模型推理成功率                  │
└─────────────────┴────────────┴─────────────────────────────────┘
```

---

## 18.4 网关与服务端协作

### 18.4.1 请求流转过程

```
【完整请求流转】

┌─────────────────────────────────────────────────────────────────┐
│ 1. 用户发起请求                                                 │
│    POST /api/recommend                                          │
│    Headers: { Authorization: Bearer xxx }                       │
│    Body: { user_id: "123", scene: "home" }                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 网关处理                                                     │
│    ├── 鉴权：验证Token，提取user_id                            │
│    ├── 限流：检查用户请求频率                                   │
│    ├── 灰度：根据user_id分流到v1/v2                            │
│    ├── 路由：选择recommend-service实例                         │
│    ├── 转换：HTTP → RPC                                        │
│    └── 转发：调用recommend-service.recommend()                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 服务端处理                                                   │
│    ├── 召回：Item-CF + 向量召回 → 1000个候选                   │
│    ├── 粗排：轻量模型打分 → 200个候选                          │
│    ├── 精排：复杂模型打分 → 50个候选                           │
│    ├── 重排：多样性+规则 → 20个结果                            │
│    └── 返回：组装响应                                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 网关返回                                                     │
│    ├── 记录日志：请求耗时、状态码                               │
│    ├── 监控上报：性能指标                                       │
│    └── 返回响应：HTTP 200 + 推荐结果                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 用户收到响应                                                 │
│    {                                                            │
│      "code": 0,                                                 │
│      "items": [                                                 │
│        {"item_id": "1", "title": "商品1", ...},                │
│        {"item_id": "2", "title": "商品2", ...},                │
│        ...                                                      │
│      ]                                                          │
│    }                                                            │
└─────────────────────────────────────────────────────────────────┘
```

### 18.4.2 数据传递方式

```
【网关 → 服务端 数据传递】

方式一：HTTP Header传递
├── 网关在HTTP Header中添加用户信息
├── 服务端从Header中提取
└── 示例：
    网关设置：
    ngx.req.set_header("X-User-Id", user_id)
    
    服务端获取：
    String userId = request.getHeader("X-User-Id");

方式二：RPC Context传递
├── 网关在RPC Context中设置
├── 服务端从Context中获取
└── 示例：
    网关设置：
    RpcContext.getContext().setAttachment("user_id", user_id);
    
    服务端获取：
    String userId = RpcContext.getContext().getAttachment("user_id");

方式三：请求体传递
├── 网关解析请求体，添加字段
├── 服务端直接使用
└── 示例：
    网关处理：
    JSONObject body = JSON.parseObject(requestBody);
    body.put("user_id", user_id);
    
    服务端接收：
    String userId = request.getUserId();
```

### 18.4.3 异常处理协作

```
【异常处理流程】

┌─────────────────────────────────────────────────────────────────┐
│ 服务端异常                                                      │
├─────────────────────────────────────────────────────────────────┤
│  1. 服务端捕获异常                                              │
│     ├── 业务异常：返回错误码和错误信息                          │
│     ├── 系统异常：返回500，记录日志                             │
│     └── 超时异常：返回504                                       │
│                                                                 │
│  2. 网关处理异常响应                                            │
│     ├── 记录异常日志                                            │
│     ├── 触发告警                                                │
│     └── 返回用户友好错误信息                                    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 网关异常                                                        │
├─────────────────────────────────────────────────────────────────┤
│  1. 鉴权失败                                                    │
│     └── 返回401 Unauthorized                                   │
│                                                                 │
│  2. 限流触发                                                    │
│     └── 返回429 Too Many Requests                              │
│                                                                 │
│  3. 服务不可用                                                  │
│     └── 返回503 Service Unavailable                            │
│                                                                 │
│  4. 网关超时                                                    │
│     └── 返回504 Gateway Timeout                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 18.5 实际案例

### 18.5.1 案例：高并发场景下的网关优化

```
【问题背景】
├── 大促期间QPS从5万涨到20万
├── 网关CPU使用率飙升到90%
└── P99延迟从5ms涨到20ms

【优化过程】

Step 1: 问题定位
├── 分析CPU热点
│   └── JSON序列化占比最高（40%）
│
├── 分析内存
│   └── 大量临时对象创建
│
└── 定位根因
    └── 日志记录使用JSON序列化，开销大

Step 2: 优化方案
├── 日志异步化
│   └── 使用异步日志队列
│
├── 减少序列化
│   └── 使用二进制格式替代JSON
│
└── 连接池优化
    └── 增加连接池大小

Step 3: 优化效果
├── CPU使用率：90% → 50%
├── P99延迟：20ms → 5ms
└── 最大QPS：20万 → 50万
```

### 18.5.2 案例：服务端性能优化

```
【问题背景】
├── 精排服务P99延迟从10ms涨到25ms
├── 特征获取耗时占比最高
└── 影响整体推荐延迟

【优化过程】

Step 1: 问题定位
├── 分析特征获取
│   ├── Redis查询次数多
│   └── 每次查询都是单独请求
│
└── 定位根因
    └── 未使用Pipeline批量查询

Step 2: 优化方案
├── Pipeline批量查询
│   └── 一次网络IO获取所有特征
│
├── 本地缓存
│   └── 热点特征缓存到本地
│
└── 并行获取
    └── 用户特征和商品特征并行获取

Step 3: 优化效果
├── 特征获取延迟：15ms → 3ms
├── 精排P99延迟：25ms → 12ms
└── 整体P99延迟：45ms → 22ms
```

---

## 18.6 总结

### 18.6.1 职责划分总结

```
【网关层 vs 服务端】

┌─────────────────┬─────────────────────────────┬─────────────────────────────┐
│ 维度            │ 网关层                      │ 服务端                      │
├─────────────────┼─────────────────────────────┼─────────────────────────────┤
│ 核心职责        │ 流量管理                    │ 业务逻辑                    │
│ 主要工作        │ 鉴权、限流、路由            │ 召回、排序、重排            │
│ 性能要求        │ P99 < 5ms                   │ P99 < 30ms                  │
│ 技术栈          │ Nginx/OpenResty             │ Java/C++/Go                 │
│ 变更频率        │ 低                          │ 高                          │
│ 可复用性        │ 高（多业务共用）            │ 低（业务特定）              │
└─────────────────┴─────────────────────────────┴─────────────────────────────┘
```

### 18.6.2 关键要点

```
【网关层关键要点】

1. 职责单一
   └── 只做流量管理，不处理业务逻辑

2. 性能优先
   └── P99延迟必须控制在5ms以内

3. 高可用
   └── 网关不可用会影响所有业务

4. 可观测
   └── 完善的监控和日志

【服务端关键要点】

1. 业务完整
   └── 实现完整的推荐逻辑

2. 性能优化
   └── 各阶段延迟控制

3. 容错处理
   └── 降级、熔断、重试

4. 可扩展
   └── 支持新业务快速接入
```

---

[← 上一章：推荐系统实操项目详解](17-practical-projects.md) | [返回目录](../README.md)
