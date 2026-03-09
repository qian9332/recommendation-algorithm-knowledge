# 十九、推荐算法工程师面试题大全

> 涵盖推荐系统全链路的面试题，每道题都有详细答案和深度解析。

---

## 目录

- [一、召回模块面试题](#一召回模块面试题)
- [二、排序模块面试题](#二排序模块面试题)
- [三、特征工程面试题](#三特征工程面试题)
- [四、多任务学习面试题](#四多任务学习面试题)
- [五、样本工程面试题](#五样本工程面试题)
- [六、模型评估面试题](#六模型评估面试题)
- [七、系统工程面试题](#七系统工程面试题)
- [八、业务场景面试题](#八业务场景面试题)

---

## 一、召回模块面试题

### 1.1 请详细介绍召回模块在推荐系统中的作用和常见方法

**答案：**

```
【召回模块的作用】

召回模块是推荐系统的第一环节，核心作用是从海量候选集（百万级甚至亿级）中
快速筛选出用户可能感兴趣的候选集（千级别），为后续排序环节提供输入。

关键指标：
├── 召回率：真正感兴趣的商品有多少被召回了
├── 覆盖率：召回的商品覆盖了多少长尾商品
└── 性能：P99延迟通常要求 < 15ms

【常见召回方法】

一、协同过滤召回

1. User-CF（基于用户的协同过滤）
   原理：找相似用户，推荐相似用户喜欢的商品
   
   优点：
   ├── 推荐结果可解释性强
   └── 适合用户兴趣相似的场景
   
   缺点：
   ├── 用户兴趣变化快时效果差
   ├── 新用户冷启动问题
   └── 用户量大时计算复杂度高
   
   适用场景：用户兴趣稳定、用户量适中的场景

2. Item-CF（基于商品的协同过滤）
   原理：推荐与用户历史行为商品相似的商品
   
   优点：
   ├── 商品相似性相对稳定
   ├── 可预先计算相似度矩阵
   └── 适合商品量适中的场景
   
   缺点：
   ├── 新商品冷启动问题
   ├── 商品相似度矩阵存储开销大
   └── 热门商品容易被过度推荐
   
   适用场景：电商、视频推荐

二、向量召回

1. DSSM双塔模型
   原理：用户塔和商品塔分别输出向量，计算向量相似度
   
   优点：
   ├── 可离线计算商品向量
   ├── 支持ANN快速检索
   └── 泛化能力强
   
   缺点：
   ├── 双塔结构限制了特征交叉
   ├── 需要大量训练数据
   └── 向量空间学习难度大
   
   优化方向：
   ├── 增加难负例挖掘
   ├── 温度系数调整
   └── 多塔结构

2. YouTube DNN召回
   原理：将召回问题转化为多分类问题
   
   特点：
   ├── 用户历史行为序列作为输入
   ├── 输出层是softmax多分类
   └── 训练时负采样，推理时ANN检索

三、图召回

1. 随机游走
   原理：在用户-商品二部图上进行随机游走
   
   优点：
   ├── 可发现潜在关联
   └── 可融合多种关系
   
   缺点：
   ├── 计算复杂度高
   └── 游走策略设计困难

2. GraphSAGE
   原理：采样邻居节点，聚合邻居特征
   
   优点：
   ├── 支持归纳学习
   ├── 可处理动态图
   └── 可扩展性好

四、深度召回

1. MIND（多兴趣召回）
   原理：一个用户可能有多个兴趣，用多个向量表示
   
   优点：
   ├── 捕捉用户多兴趣
   └── 提升召回多样性

2. SDM（深度序列召回）
   原理：建模用户行为序列，捕捉长期和短期兴趣
   
   优点：
   ├── 捕捉用户兴趣演化
   └── 长短期兴趣融合

【多路召回策略】

实际系统中通常采用多路召回：

召回通道配置示例：
├── Item-CF召回：30%
├── 向量召回：25%
├── 图召回：15%
├── 热门召回：10%
├── 新品召回：10%
└── 其他召回：10%

多路召回的优势：
├── 互补性：不同方法召回不同类型的商品
├── 稳定性：某一路召回失效时，其他路可兜底
└── 多样性：提升召回结果的多样性

【召回优化方向】

1. 难负例挖掘
   问题：随机负例太简单，模型区分能力弱
   方案：选择模型预测分数高但实际无正反馈的商品作为负例
   
2. 热门降权
   问题：热门商品容易被过度召回
   方案：score *= 1 / log(1 + popularity)
   
3. 冷启动优化
   问题：新用户/新商品无历史数据
   方案：
   ├── 新用户：使用人口统计学特征、全局热门
   └── 新商品：使用内容特征、探索流量
```

---

### 1.2 请详细介绍Item-CF的实现原理和优化方法

**答案：**

```
【Item-CF核心原理】

基本思想：如果两个商品被很多用户同时喜欢，则这两个商品相似。
推荐时：找用户历史喜欢的商品，推荐与这些商品相似的商品。

【相似度计算】

1. 余弦相似度
   sim(i,j) = |N(i) ∩ N(j)| / sqrt(|N(i)| × |N(j)|)
   
   其中：
   ├── N(i)：喜欢商品i的用户集合
   ├── N(i) ∩ N(j)：同时喜欢i和j的用户集合
   └── 分母：归一化因子

2. 改进的相似度（考虑流行度）
   sim(i,j) = |N(i) ∩ N(j)| / (|N(i)|^α × |N(j)|^(1-α))
   
   α通常取0.5，可调整热门商品的权重

3. Jaccard相似度
   sim(i,j) = |N(i) ∩ N(j)| / |N(i) ∪ N(j)|

【实现步骤】

Step 1: 构建用户-商品倒排表
├── 输入：用户行为日志
└── 输出：每个用户交互过的商品列表

Step 2: 计算商品共现矩阵
├── 遍历每个用户的商品列表
├── 两两商品共现次数+1
└── 得到商品共现矩阵C

Step 3: 计算相似度矩阵
├── 根据共现矩阵计算相似度
├── 对每个商品，保留Top-K相似商品
└── 存储相似度矩阵

Step 4: 在线召回
├── 获取用户最近交互的商品
├── 查找这些商品的相似商品
├── 加权求和得到候选商品分数
└── 排序返回Top-N

【代码实现】

# Step 1: 构建用户-商品倒排表
def build_user_items(logs):
    user_items = defaultdict(set)
    for log in logs:
        user_items[log.user_id].add(log.item_id)
    return user_items

# Step 2: 计算共现矩阵
def build_cooccurrence(user_items):
    cooccur = defaultdict(lambda: defaultdict(int))
    item_count = defaultdict(int)
    
    for user, items in user_items.items():
        for i in items:
            item_count[i] += 1
            for j in items:
                if i != j:
                    cooccur[i][j] += 1
    
    return cooccur, item_count

# Step 3: 计算相似度
def calculate_similarity(cooccur, item_count, top_k=100):
    similarity = {}
    
    for i, related_items in cooccur.items():
        sim_scores = []
        for j, co_count in related_items.items():
            # 余弦相似度
            sim = co_count / math.sqrt(item_count[i] * item_count[j])
            sim_scores.append((j, sim))
        
        # 保留Top-K相似商品
        sim_scores.sort(key=lambda x: x[1], reverse=True)
        similarity[i] = sim_scores[:top_k]
    
    return similarity

# Step 4: 在线召回
def recall(user_recent_items, similarity, top_n=500):
    candidates = defaultdict(float)
    
    for item_id in user_recent_items:
        if item_id in similarity:
            for sim_item, sim_score in similarity[item_id]:
                candidates[sim_item] += sim_score
    
    # 排序返回
    sorted_items = sorted(candidates.items(), key=lambda x: x[1], reverse=True)
    return sorted_items[:top_n]

【优化方法】

1. 热门商品降权
   问题：热门商品与很多商品都有共现，相似度虚高
   
   方案：
   ├── 方案1：相似度计算时除以热门商品的影响
   │   sim(i,j) = co_count / (log(item_count[i]) × log(item_count[j]))
   │
   ├── 方案2：召回时对热门商品降权
   │   score = sim_score / log(1 + item_popularity)
   │
   └── 方案3：IIF（Inverse Item Frequency）
       sim(i,j) = co_count / (1 + log(item_count[j]))

2. 时间衰减
   问题：用户兴趣会随时间变化，早期行为参考价值低
   
   方案：
   ├── 方案1：只使用最近N天的行为数据
   ├── 方案2：时间衰减权重
   │   weight = exp(-λ × days_ago)
   │
   └── 方案3：分时段计算相似度，近期行为权重更高

3. 活跃用户降权
   问题：活跃用户的行为对相似度贡献过大
   
   方案：
   sim(i,j) = Σ 1/log(1 + |N(u)|)
   
   其中N(u)是用户u的行为数量

4. 类目约束
   问题：跨类目商品相似度意义不大
   
   方案：
   ├── 方案1：只计算同类目商品的相似度
   └── 方案2：跨类目相似度打折

5. 冷启动优化
   问题：新商品无共现数据
   
   方案：
   ├── 方案1：使用内容相似度（标题、类目、属性）
   ├── 方案2：使用商品向量相似度
   └── 方案3：新商品保量策略

【工程实现优化】

1. 存储优化
   ├── 只存储Top-K相似商品
   ├── 使用稀疏矩阵存储
   └── 压缩存储（只存商品ID和分数）

2. 计算优化
   ├── 增量更新：每天只更新有变化的商品
   ├── 分布式计算：使用Spark/MapReduce
   └── 采样：对活跃用户采样

3. 查询优化
   ├── 预计算：离线计算好相似度矩阵
   ├── 缓存：Redis缓存热门商品的相似列表
   └── 倒排索引：快速查找相似商品

【实际效果】

在电商场景的典型效果：
├── 召回率：35-45%
├── 覆盖率：60-70%
├── P99延迟：< 10ms
└── 存储成本：商品数 × Top-K × 12字节
```

---

### 1.3 请详细介绍向量召回的原理和实现

**答案：**

```
【向量召回概述】

核心思想：将用户和商品映射到同一向量空间，通过向量相似度检索召回商品。

优势：
├── 泛化能力强：可召回无直接交互的商品
├── 支持ANN检索：可快速从百万商品中检索
└── 可融合多种特征：行为、内容、上下文

【DSSM双塔模型】

一、模型结构

用户塔：
├── 输入：用户特征（ID、画像、历史行为）
├── Embedding层：将离散特征转为向量
├── MLP层：多层感知机提取特征
└── 输出：用户向量 u ∈ R^d

商品塔：
├── 输入：商品特征（ID、类目、属性）
├── Embedding层：将离散特征转为向量
├── MLP层：多层感知机提取特征
└── 输出：商品向量 v ∈ R^d

相似度计算：
├── 内积：sim(u,v) = u^T v
├── 余弦：sim(u,v) = u^T v / (||u|| × ||v||)
└── 欧氏距离：sim(u,v) = -||u - v||^2

二、训练目标

1. Softmax Loss（多分类）
   L = -log(exp(sim(u,v+)) / Σ exp(sim(u,v_i)))
   
   其中v+是正样本，v_i是所有商品

2. InfoNCE Loss（对比学习）
   L = -log(exp(sim(u,v+)/τ) / Σ exp(sim(u,v_i)/τ))
   
   τ是温度系数，控制分布的平滑程度

3. 负采样优化
   问题：Softmax需要计算所有商品，计算量大
   
   方案：
   ├── 随机负采样：随机选择负样本
   ├── Batch内负采样：同批次其他商品作为负样本
   └── 难负例挖掘：选择模型预测分数高的负样本

三、温度系数的作用

温度系数τ控制softmax分布的平滑程度：

τ较小（如0.05）：
├── 分布更尖锐
├── 模型更关注难样本
└── 容易过拟合

τ较大（如0.5）：
├── 分布更平滑
├── 模型更关注简单样本
└── 训练更稳定

经验值：
├── 通常取0.05-0.1
├── 可随训练步数衰减
└── 需要根据数据集调整

【代码实现】

import torch
import torch.nn as nn
import torch.nn.functional as F

class DSSMModel(nn.Module):
    def __init__(self, user_feature_dims, item_feature_dims, embedding_dim=64):
        super().__init__()
        
        # 用户塔
        self.user_embeddings = nn.ModuleDict({
            name: nn.Embedding(dim, embedding_dim)
            for name, dim in user_feature_dims.items()
        })
        self.user_mlp = nn.Sequential(
            nn.Linear(len(user_feature_dims) * embedding_dim, 256),
            nn.ReLU(),
            nn.Linear(256, 128),
            nn.ReLU(),
            nn.Linear(128, embedding_dim)
        )
        
        # 商品塔
        self.item_embeddings = nn.ModuleDict({
            name: nn.Embedding(dim, embedding_dim)
            for name, dim in item_feature_dims.items()
        })
        self.item_mlp = nn.Sequential(
            nn.Linear(len(item_feature_dims) * embedding_dim, 256),
            nn.ReLU(),
            nn.Linear(256, 128),
            nn.ReLU(),
            nn.Linear(128, embedding_dim)
        )
        
        self.temperature = 0.07
    
    def encode_user(self, user_features):
        # 拼接所有用户特征embedding
        embeddings = []
        for name, value in user_features.items():
            emb = self.user_embeddings[name](value)
            embeddings.append(emb)
        
        concat = torch.cat(embeddings, dim=-1)
        user_vec = self.user_mlp(concat)
        
        # L2归一化
        user_vec = F.normalize(user_vec, p=2, dim=-1)
        return user_vec
    
    def encode_item(self, item_features):
        # 拼接所有商品特征embedding
        embeddings = []
        for name, value in item_features.items():
            emb = self.item_embeddings[name](value)
            embeddings.append(emb)
        
        concat = torch.cat(embeddings, dim=-1)
        item_vec = self.item_mlp(concat)
        
        # L2归一化
        item_vec = F.normalize(item_vec, p=2, dim=-1)
        return item_vec
    
    def forward(self, user_features, item_features):
        user_vec = self.encode_user(user_features)
        item_vec = self.encode_item(item_features)
        
        # 计算相似度
        sim = torch.matmul(user_vec, item_vec.T) / self.temperature
        
        return sim, user_vec, item_vec
    
    def compute_loss(self, sim, labels):
        # InfoNCE Loss
        loss = F.cross_entropy(sim, labels)
        return loss

# 训练代码
def train(model, dataloader, optimizer, epochs):
    model.train()
    
    for epoch in range(epochs):
        for batch in dataloader:
            user_features, item_features, labels = batch
            
            # 前向传播
            sim, user_vec, item_vec = model(user_features, item_features)
            
            # 计算损失
            loss = model.compute_loss(sim, labels)
            
            # 反向传播
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()

【难负例挖掘】

问题：随机负例太简单，模型容易区分，学不到有效特征

方案：
1. 离线难负例挖掘
   ├── 用当前模型对所有商品打分
   ├── 选择分数高但无正反馈的商品
   └── 加入训练集作为难负例

2. 在线难负例挖掘
   ├── Batch内选择分数高的负样本
   └── 动态调整负样本权重

3. 混合策略
   ├── 70%随机负例 + 30%难负例
   └── 难负例提升精度，随机负例保证泛化

代码示例：
def hard_negative_mining(model, user_features, all_item_features, k=10):
    # 计算用户向量
    user_vec = model.encode_user(user_features)
    
    # 计算所有商品向量
    all_item_vecs = []
    for item_features in all_item_features:
        item_vec = model.encode_item(item_features)
        all_item_vecs.append(item_vec)
    all_item_vecs = torch.stack(all_item_vecs)
    
    # 计算相似度
    scores = torch.matmul(user_vec, all_item_vecs.T)
    
    # 选择Top-K高分但非正样本的商品
    top_k_indices = torch.topk(scores, k=k+100).indices
    
    # 过滤掉正样本
    hard_negatives = []
    for idx in top_k_indices:
        if idx not in positive_items:
            hard_negatives.append(idx)
        if len(hard_negatives) >= k:
            break
    
    return hard_negatives

【ANN检索】

训练完成后，需要从百万商品中快速检索：

1. Faiss库
   ├── 支持多种索引类型
   ├── 支持GPU加速
   └── 支持分布式索引

2. 索引类型选择
   ├── IVF（倒排索引）：适合中等规模
   ├── HNSW（层次导航小世界）：适合高精度
   └── PQ（乘积量化）：适合大规模

3. 使用示例
   import faiss
   
   # 构建索引
   d = 64  # 向量维度
   nlist = 100  # 聚类中心数
   quantizer = faiss.IndexFlatL2(d)
   index = faiss.IndexIVFFlat(quantizer, d, nlist)
   
   # 训练索引
   index.train(item_vectors)
   
   # 添加向量
   index.add(item_vectors)
   
   # 检索
   k = 100  # 返回Top-K
   distances, indices = index.search(user_vector, k)

【实际效果】

在电商场景的典型效果：
├── 召回率：40-50%
├── 覆盖率：70-80%
├── P99延迟：< 10ms（ANN检索）
└── 训练时间：数小时（百万级样本）
```

---

## 二、排序模块面试题

### 2.1 请详细介绍排序模型的发展历程和各模型的特点

**答案：**

```
【排序模型发展历程】

一、逻辑回归时代（LR，2010年前后）

模型结构：
├── 输入：人工特征工程
├── 模型：y = sigmoid(w^T x + b)
└── 输出：点击概率

优点：
├── 简单高效，易于实现
├── 可解释性强
└── 工程成熟

缺点：
├── 无法自动学习特征交叉
├── 依赖大量人工特征工程
└── 表达能力有限

代表系统：Google广告系统、早期推荐系统

二、因子分解机时代（FM，2010-2015）

模型结构：
├── 一阶特征：w^T x
├── 二阶交叉：<v_i, v_j> x_i x_j
└── 输出：y = sigmoid(一阶 + 二阶)

优点：
├── 自动学习二阶特征交叉
├── 泛化能力强
└── 可处理稀疏特征

缺点：
├── 只能学习二阶交叉
├── 高阶交叉需要人工设计
└── 计算复杂度O(kn^2)

改进版本：
├── FFM：引入特征域概念
├── DeepFM：结合深度网络
└── xDeepFM：压缩交互网络

三、深度学习时代（2015-至今）

1. Wide&Deep（Google，2016）
   结构：
   ├── Wide侧：记忆能力，学习历史模式
   └── Deep侧：泛化能力，学习特征组合
   
   优点：
   ├── 结合记忆和泛化
   ├── 工程实现简单
   └── 效果显著提升
   
   缺点：
   ├── Wide侧仍需人工特征工程
   └── 特征交叉能力有限

2. DeepFM（华为，2017）
   结构：
   ├── FM组件：自动学习二阶交叉
   └── Deep组件：学习高阶特征
   
   优点：
   ├── 无需人工特征工程
   ├── 端到端训练
   └── 效果优于Wide&Deep

3. DIN（阿里，2018）
   核心创新：注意力机制
   
   结构：
   ├── 用户行为序列输入
   ├── 注意力机制计算兴趣权重
   └── 加权求和得到用户兴趣向量
   
   优点：
   ├── 自适应学习用户兴趣
   ├── 可解释性强
   └── 适合电商场景

4. DIEN（阿里，2019）
   核心创新：兴趣演化网络
   
   结构：
   ├── 兴趣提取层：GRU提取兴趣序列
   ├── 兴趣演化层：AUGRU建模兴趣演化
   └── 输出层：预测点击概率
   
   优点：
   ├── 捕捉用户兴趣演化
   ├── 序列建模能力强

5. AutoInt（2019）
   核心创新：自注意力机制
   
   结构：
   ├── 多头自注意力层
   ├── 自动学习高阶特征交叉
   └── 残差连接
   
   优点：
   ├── 自动学习任意阶特征交叉
   ├── 可解释性强
   └── 效果优于DeepFM

6. DCN-V2（Google，2020）
   核心创新：交叉网络
   
   结构：
   ├── 交叉网络：显式特征交叉
   └── 深度网络：隐式特征交叉
   
   优点：
   ├── 显式+隐式特征交叉
   ├── 计算效率高
   └── 效果SOTA

【模型选择建议】

场景选择：
├── 快速上线：Wide&Deep
├── 效果优先：DCN-V2、AutoInt
├── 序列特征丰富：DIN、DIEN
└── 可解释性要求高：FM、AutoInt

计算资源：
├── 资源有限：LR、FM
├── 资源适中：Wide&Deep、DeepFM
└── 资源充足：DIEN、DCN-V2

【代码实现示例】

# DeepFM实现
class DeepFM(nn.Module):
    def __init__(self, feature_dims, embedding_dim=8, hidden_units=[256, 128]):
        super().__init__()
        
        # FM部分
        self.fm_embeddings = nn.ModuleDict({
            name: nn.Embedding(dim, embedding_dim)
            for name, dim in feature_dims.items()
        })
        self.fm_linear = nn.ModuleDict({
            name: nn.Embedding(dim, 1)
            for name, dim in feature_dims.items()
        })
        
        # Deep部分
        input_dim = len(feature_dims) * embedding_dim
        layers = []
        for hidden in hidden_units:
            layers.append(nn.Linear(input_dim, hidden))
            layers.append(nn.ReLU())
            layers.append(nn.Dropout(0.2))
            input_dim = hidden
        layers.append(nn.Linear(input_dim, 1))
        self.deep = nn.Sequential(*layers)
    
    def forward(self, features):
        # FM一阶
        fm_first_order = []
        for name, value in features.items():
            emb = self.fm_linear[name](value)
            fm_first_order.append(emb)
        fm_first_order = torch.sum(torch.cat(fm_first_order, dim=1), dim=1, keepdim=True)
        
        # FM二阶
        embeddings = []
        for name, value in features.items():
            emb = self.fm_embeddings[name](value)
            embeddings.append(emb)
        embeddings = torch.cat(embeddings, dim=1)
        
        # 计算二阶交叉
        sum_emb = torch.sum(embeddings, dim=1, keepdim=True)
        sum_square = torch.sum(sum_emb ** 2, dim=2)
        square_sum = torch.sum(embeddings ** 2, dim=2)
        sum_square = torch.sum(sum_square, dim=1, keepdim=True)
        square_sum = torch.sum(square_sum, dim=1, keepdim=True)
        fm_second_order = 0.5 * (sum_square - square_sum)
        
        # Deep部分
        deep_input = embeddings.view(embeddings.size(0), -1)
        deep_output = self.deep(deep_input)
        
        # 合并输出
        output = fm_first_order + fm_second_order + deep_output
        output = torch.sigmoid(output)
        
        return output
```

---

### 2.2 请详细介绍多任务学习中的MMoE和PLE模型

**答案：**

```
【多任务学习背景】

在推荐系统中，通常需要同时优化多个目标：
├── CTR（点击率）
├── CVR（转化率）
├── GMV（成交金额）
├── 时长（停留时间）
└── 互动（点赞、评论、分享）

挑战：
├── 任务冲突：不同任务的梯度方向可能相反
├── 负迁移：优化一个任务导致另一个任务效果下降
└── 跷跷板现象：一个任务提升，另一个任务下降

【Shared-Bottom模型】

结构：
├── 共享底层：提取共享特征
├── 任务塔：每个任务一个塔
└── 输出：每个任务一个输出

优点：
├── 参数少，计算快
└── 共享特征，泛化强

缺点：
├── 任务冲突严重
├── 共享层成为瓶颈
└── 负迁移问题

【MMoE模型（Google，2018）】

一、模型结构

核心思想：多专家混合，每个任务学习不同的专家组合

结构：
├── Expert层：多个专家网络
├── Gate层：每个任务一个门控网络
└── Tower层：每个任务一个塔

工作流程：
1. 输入特征经过多个Expert
2. Gate网络输出每个Expert的权重
3. 加权求和得到任务特定特征
4. Tower网络输出任务预测

二、数学表达

Expert输出：
E_i(x) = f_i(x), i = 1, 2, ..., n

Gate输出：
g_k(x) = softmax(W_gk · x)

任务k的特征：
f_k(x) = Σ g_k(x)_i · E_i(x)

任务k的输出：
y_k = h_k(f_k(x))

三、代码实现

class MMoE(nn.Module):
    def __init__(self, input_dim, num_experts=8, num_tasks=2, expert_dim=64):
        super().__init__()
        
        # Expert网络
        self.experts = nn.ModuleList([
            nn.Sequential(
                nn.Linear(input_dim, expert_dim),
                nn.ReLU(),
                nn.Linear(expert_dim, expert_dim),
                nn.ReLU()
            ) for _ in range(num_experts)
        ])
        
        # Gate网络（每个任务一个）
        self.gates = nn.ModuleList([
            nn.Sequential(
                nn.Linear(input_dim, num_experts),
                nn.Softmax(dim=-1)
            ) for _ in range(num_tasks)
        ])
        
        # Tower网络（每个任务一个）
        self.towers = nn.ModuleList([
            nn.Sequential(
                nn.Linear(expert_dim, 32),
                nn.ReLU(),
                nn.Linear(32, 1)
            ) for _ in range(num_tasks)
        ])
    
    def forward(self, x):
        # Expert输出
        expert_outputs = torch.stack([expert(x) for expert in self.experts], dim=1)
        
        # 各任务输出
        outputs = []
        for gate, tower in zip(self.gates, self.towers):
            # Gate权重
            gate_weights = gate(x)  # [batch, num_experts]
            
            # 加权求和
            task_features = torch.sum(
                expert_outputs * gate_weights.unsqueeze(-1), 
                dim=1
            )
            
            # Tower输出
            output = tower(task_features)
            outputs.append(output)
        
        return outputs

四、优点

├── 缓解任务冲突：不同任务使用不同的专家组合
├── 灵活性强：任务可以自由选择专家
└── 参数适中：比Shared-Bottom稍多

五、缺点

├── Expert仍完全共享
├── 任务冲突未完全解决
└── Gate可能收敛到单一Expert

【PLE模型（腾讯，2020）】

一、核心改进

MMoE的问题：
├── 所有Expert完全共享
├── 任务间仍存在干扰
└── Expert选择可能冲突

PLE的改进：
├── 显式分离共享Expert和任务独占Expert
├── 每层都有共享和独占部分
└── 更好地缓解任务冲突

二、模型结构

CGC（Customized Gate Control）模块：
├── 共享Expert：所有任务共享
├── 任务独占Expert：每个任务独有
└── Gate网络：选择共享和独占Expert的组合

多层PLE：
├── 多层CGC堆叠
├── 每层都有共享和独占Expert
└── 逐层提取任务特定特征

三、代码实现

class PLE(nn.Module):
    def __init__(self, input_dim, num_experts=8, num_tasks=2, expert_dim=64, num_layers=2):
        super().__init__()
        
        self.num_tasks = num_tasks
        self.num_layers = num_layers
        
        # 每层的Expert
        self.layers = nn.ModuleList()
        for _ in range(num_layers):
            layer = {}
            
            # 共享Expert
            layer['shared_experts'] = nn.ModuleList([
                self._build_expert(input_dim, expert_dim)
                for _ in range(num_experts // 2)
            ])
            
            # 任务独占Expert
            layer['task_experts'] = nn.ModuleList([
                nn.ModuleList([
                    self._build_expert(input_dim, expert_dim)
                    for _ in range(num_experts // 2)
                ]) for _ in range(num_tasks)
            ])
            
            # Gate网络
            layer['gates'] = nn.ModuleList([
                nn.Sequential(
                    nn.Linear(input_dim, num_experts),
                    nn.Softmax(dim=-1)
                ) for _ in range(num_tasks)
            ])
            
            self.layers.append(nn.ModuleDict(layer))
            input_dim = expert_dim
        
        # Tower网络
        self.towers = nn.ModuleList([
            nn.Sequential(
                nn.Linear(expert_dim, 32),
                nn.ReLU(),
                nn.Linear(32, 1)
            ) for _ in range(num_tasks)
        ])
    
    def _build_expert(self, input_dim, output_dim):
        return nn.Sequential(
            nn.Linear(input_dim, output_dim),
            nn.ReLU()
        )
    
    def forward(self, x):
        for layer in self.layers:
            # 共享Expert输出
            shared_outputs = torch.stack([
                expert(x) for expert in layer['shared_experts']
            ], dim=1)
            
            # 各任务处理
            task_features = []
            for task_id in range(self.num_tasks):
                # 任务独占Expert输出
                task_outputs = torch.stack([
                    expert(x) for expert in layer['task_experts'][task_id]
                ], dim=1)
                
                # 合并Expert
                all_outputs = torch.cat([shared_outputs, task_outputs], dim=1)
                
                # Gate权重
                gate_weights = layer['gates'][task_id](x)
                
                # 加权求和
                feature = torch.sum(all_outputs * gate_weights.unsqueeze(-1), dim=1)
                task_features.append(feature)
            
            # 更新输入
            x = torch.stack(task_features, dim=0)
        
        # Tower输出
        outputs = [tower(feature) for tower, feature in zip(self.towers, task_features)]
        
        return outputs

四、优点

├── 更好地缓解任务冲突
├── 共享和独占Expert显式分离
├── 效果优于MMoE
└── 可解释性更强

五、实验效果

在腾讯视频推荐场景：
├── CTR AUC：+0.8%
├── CVR AUC：+1.2%
└── 整体GMV：+3.5%

【模型选择建议】

场景选择：
├── 任务相关性高：Shared-Bottom
├── 任务相关性中等：MMoE
└── 任务相关性低：PLE

计算资源：
├── 资源有限：Shared-Bottom
├── 资源适中：MMoE
└── 资源充足：PLE

【梯度冲突分析】

如何判断任务冲突：
├── 计算任务梯度的余弦相似度
├── 相似度为负：存在冲突
└── 相似度接近0：无冲突

解决方案：
├── MMoE/PLE：结构层面解决
├── PCGrad：梯度投影
├── GradNorm：动态调整任务权重
└── Uncertainty Weight：不确定性加权
```

---

## 三、特征工程面试题

### 3.1 请详细介绍推荐系统中的特征类型和处理方法

**答案：**

```
【特征类型分类】

一、按数据来源分类

1. 用户特征
   静态特征：
   ├── 人口统计学：年龄、性别、城市、职业
   ├── 注册信息：注册时间、会员等级
   └── 设备信息：手机型号、系统版本
   
   动态特征：
   ├── 历史行为：点击、收藏、购买商品序列
   ├── 统计特征：历史CTR、CVR、平均订单金额
   └── 实时特征：最近1小时点击、当前session行为

2. 商品特征
   静态特征：
   ├── 基础属性：类目、品牌、价格、产地
   ├── 内容特征：标题、描述、图片、视频
   └── 上架信息：上架时间、库存、销量
   
   动态特征：
   ├── 统计特征：最近7天CTR、CVR、销量
   ├── 实时特征：最近1小时曝光、点击
   └── 价格特征：当前价格、历史最低价

3. 上下文特征
   ├── 时间：小时、星期、是否节假日
   ├── 地点：城市、商圈、室内/室外
   ├── 网络：WiFi/4G/5G
   └── 场景：首页、搜索、详情页

4. 交叉特征
   ├── 用户-商品交叉：用户对商品类目的偏好
   ├── 用户-上下文交叉：用户在不同时间段的偏好
   └── 商品-上下文交叉：商品在不同时间段的点击率

二、按特征类型分类

1. 连续特征
   示例：年龄、价格、销量、CTR
   
   处理方法：
   ├── 归一化：x' = (x - min) / (max - min)
   ├── 标准化：x' = (x - mean) / std
   ├── 分桶：将连续值离散化为桶
   └── 对数变换：x' = log(x + 1)

2. 离散特征
   示例：性别、类目、城市
   
   处理方法：
   ├── One-Hot编码：适合基数小的特征
   ├── Embedding编码：适合基数大的特征
   ├── Target编码：用目标变量编码
   └── Hash编码：固定维度，适合超大规模

3. 序列特征
   示例：用户点击序列、搜索词序列
   
   处理方法：
   ├── Pooling：平均池化、最大池化
   ├── RNN：LSTM、GRU
   ├── Attention：DIN、Transformer
   └── CNN：TextCNN

【特征处理详解】

一、连续特征处理

1. 归一化
   适用场景：
   ├── 特征量纲差异大
   ├── 需要快速收敛
   └── 神经网络输入
   
   缺点：
   ├── 对异常值敏感
   └── 可能丢失分布信息

2. 分桶
   等宽分桶：
   ├── 每个桶宽度相同
   ├── 实现简单
   └── 可能不均匀
   
   等频分桶：
   ├── 每个桶样本数相同
   ├── 分布均匀
   └── 需要计算分位数
   
   自定义分桶：
   ├── 根据业务经验划分
   ├── 更符合业务逻辑
   └── 需要领域知识

代码示例：
def bucketize(value, boundaries):
    """
    将连续值分桶
    boundaries: [10, 20, 50, 100]
    返回: 桶号 0, 1, 2, 3, 4
    """
    for i, bound in enumerate(boundaries):
        if value < bound:
            return i
    return len(boundaries)

二、离散特征处理

1. One-Hot编码
   适用场景：
   ├── 特征基数小（<100）
   ├── 线性模型
   └── 树模型
   
   缺点：
   ├── 维度爆炸
   ├── 稀疏性
   └── 无法表达相似性

2. Embedding编码
   适用场景：
   ├── 特征基数大
   ├── 深度学习模型
   └── 需要表达相似性
   
   优点：
   ├── 低维稠密
   ├── 可学习相似性
   └── 泛化能力强
   
   Embedding维度选择：
   ├── 经验公式：min(8, 1.6 * pow(n, 0.25))
   ├── 常见选择：8, 16, 32, 64
   └── 根据效果调整

代码示例：
class FeatureEmbedding(nn.Module):
    def __init__(self, vocab_size, embed_dim):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim)
    
    def forward(self, x):
        return self.embedding(x)

3. Target编码
   原理：用目标变量的统计量编码离散特征
   
   公式：
   encoding = (n * mean_target + m * global_mean) / (n + m)
   
   其中：
   ├── n：该特征值的出现次数
   ├── mean_target：该特征值的目标变量均值
   ├── m：平滑参数
   └── global_mean：全局目标变量均值
   
   优点：
   ├── 编码后有明确的业务含义
   ├── 可处理高基数特征
   └── 效果通常较好
   
   缺点：
   ├── 容易过拟合
   ├── 需要交叉验证
   └── 数据泄露风险

4. Hash编码
   原理：将特征值哈希到固定维度空间
   
   优点：
   ├── 维度固定
   ├── 无需维护映射表
   └── 适合超大规模特征
   
   缺点：
   ├── 哈希冲突
   ├── 信息损失
   └── 不可逆

代码示例：
def hash_feature(value, num_buckets):
    return hash(value) % num_buckets

三、序列特征处理

1. Pooling方法
   平均池化：
   ├── 简单高效
   ├── 适合序列长度固定
   └── 丢失顺序信息
   
   最大池化：
   ├── 提取最显著特征
   └── 适合提取关键信息
   
   加权池化：
   ├── 根据重要性加权
   └── 需要设计权重

2. RNN方法
   LSTM：
   ├── 捕捉长距离依赖
   ├── 解决梯度消失
   └── 计算量大
   
   GRU：
   ├── 参数少
   ├── 计算快
   └── 效果接近LSTM

3. Attention方法
   DIN（Deep Interest Network）：
   核心思想：根据候选商品，动态计算用户兴趣
   
   公式：
   v_u = Σ a(v_i, v_a) * v_i
   
   其中：
   ├── v_i：用户历史行为商品的向量
   ├── v_a：候选商品向量
   └── a：注意力函数

代码示例：
class DIN(nn.Module):
    def __init__(self, embed_dim):
        super().__init__()
        self.attention = nn.Sequential(
            nn.Linear(embed_dim * 4, 64),
            nn.ReLU(),
            nn.Linear(64, 1)
        )
    
    def forward(self, item_seq, candidate_item):
        # item_seq: [batch, seq_len, embed_dim]
        # candidate_item: [batch, embed_dim]
        
        # 扩展候选商品
        candidate_item = candidate_item.unsqueeze(1).expand_as(item_seq)
        
        # 注意力输入
        att_input = torch.cat([
            item_seq,
            candidate_item,
            item_seq - candidate_item,
            item_seq * candidate_item
        ], dim=-1)
        
        # 注意力权重
        att_weight = F.softmax(self.attention(att_input), dim=1)
        
        # 加权求和
        user_interest = torch.sum(att_weight * item_seq, dim=1)
        
        return user_interest

【特征选择方法】

1. 过滤法
   ├── 方差阈值：删除方差小的特征
   ├── 相关系数：删除相关性高的特征
   └── 互信息：选择与目标相关性高的特征

2. 包装法
   ├── 前向选择：逐步添加特征
   ├── 后向删除：逐步删除特征
   └── 递归特征消除

3. 嵌入法
   ├── L1正则：自动特征选择
   ├── 树模型重要性：GBDT特征重要性
   └── 深度学习：自动特征学习

【特征工程最佳实践】

1. 特征设计原则
   ├── 业务理解优先
   ├── 可解释性
   ├── 可计算性
   └── 可维护性

2. 特征迭代流程
   ├── 数据分析：理解数据分布
   ├── 特征设计：设计候选特征
   ├── 特征实现：编码实现
   ├── 特征评估：离线评估效果
   └── 特征上线：A/B测试验证

3. 特征监控
   ├── 特征覆盖率
   ├── 特征分布漂移
   ├── 特征重要性变化
   └── 特征质量监控
```

---

由于内容较多，我将继续创建更多面试题...

---

## 四、样本工程面试题

### 4.1 请详细介绍推荐系统中的样本构造方法

**答案：**

```
【样本构造的重要性】

样本质量决定模型上限：
├── 样本质量 > 样本数量 > 特征工程 > 模型结构
├── 好的样本能让简单模型也有好效果
└── 坏的样本会让复杂模型也无法发挥作用

【正样本构造】

一、正样本来源

1. 点击行为
   ├── 优点：数据量大、信号明确
   ├── 缺点：存在误点击
   └── 适用：CTR预估

2. 转化行为
   ├── 优点：价值高、信号强
   ├── 缺点：数据稀疏、延迟高
   └── 适用：CVR预估

3. 深度行为
   ├── 停留时长：>30秒算正样本
   ├── 滑动行为：滑动到详情页
   └── 互动行为：点赞、收藏、分享

二、正样本置信度加权

问题：不是所有正样本都同等重要

解决方案：
1. 行为深度加权
   weight = {
       'click': 1.0,
       'add_cart': 2.0,
       'collect': 2.5,
       'order': 5.0,
       'pay': 10.0
   }

2. 停留时长加权
   def get_dwell_weight(dwell_seconds):
       if dwell_seconds < 5:
           return 0.3  # 快速划走
       elif dwell_seconds < 30:
           return 0.7
       elif dwell_seconds < 120:
           return 1.0
       else:
           return 1.5  # 深度浏览

3. 时间衰减加权
   weight = exp(-λ * days_ago)
   
   λ通常取0.1-0.3

三、延迟归因

问题：转化可能延迟数小时甚至数天

解决方案：
1. 固定窗口归因
   ├── 点击后24小时内转化 → 归因
   ├── 点击后7天内转化 → 归因
   └── 超过窗口不归因

2. 生存分析建模
   ├── 拟合转化延迟分布（威布尔分布）
   ├── 计算t95（95%转化发生的时间）
   └── 用t95作为归因窗口

3. 实时+延迟双轨
   ├── 实时样本：点击后立即构造（label=0）
   ├── 延迟更新：转化发生后更新label
   └── 样本回填机制

【负样本构造】

一、负样本构造方法对比

1. 随机负采样
   方法：从全量商品中随机抽取
   
   优点：
   ├── 实现简单
   ├── 训练速度快
   └── 可控制负样本数量
   
   缺点：
   ├── 与线上分布差异大
   ├── 太简单，模型区分能力弱
   └── 热门商品被过度惩罚

2. Batch内负采样
   方法：同批次其他样本作为负样本
   
   优点：
   ├── 显存零增加
   ├── 分布相对一致
   └── 实现简单
   
   缺点：
   ├── 热门商品被过度惩罚
   ├── 负样本质量不可控
   └── 依赖batch size

3. 曝光未点击（INC）
   方法：真实曝光但未点击的商品
   
   优点：
   ├── 最接近线上分布
   ├── 负样本质量高
   └── 效果通常最好
   
   缺点：
   ├── 位置偏差
   ├── 冷启动覆盖不足
   └── 数据获取成本高

4. 难负例挖掘
   方法：模型预测分数高但无正反馈的商品
   
   优点：
   ├── 样本难度可控
   ├── 提升模型区分能力
   └── 效果提升明显
   
   缺点：
   ├── 计算链路重
   ├── 需要定期更新
   └── 可能过拟合

二、难负例挖掘详解

流程：
Step 1: 用轻量模型对全库打分
├── 部署异步推理服务
├── 小时级产出难负例
└── 取分数最高但无正反馈的Top-K

Step 2: 难度分层
├── Easy: 随机负例
├── Medium: Batch内负例
├── Hard: 难负例（高分未点击）
└── Super Hard: 同类目高分未点击

Step 3: 课程学习
├── 训练前20%: 用Easy+Medium
├── 训练中60%: 用Medium+Hard
├── 训练后20%: 用Hard+Super Hard
└── 随机负例作为正则（5-10%）

代码示例：
class HardNegativeMiner:
    def __init__(self, model, item_pool, top_k=100):
        self.model = model
        self.item_pool = item_pool
        self.top_k = top_k
    
    def mine(self, user_features, positive_items):
        # 对所有商品打分
        scores = []
        for item_features in self.item_pool:
            score = self.model.predict(user_features, item_features)
            scores.append((item_features['item_id'], score))
        
        # 排序
        scores.sort(key=lambda x: x[1], reverse=True)
        
        # 选择高分但非正样本的商品
        hard_negatives = []
        for item_id, score in scores:
            if item_id not in positive_items:
                hard_negatives.append(item_id)
            if len(hard_negatives) >= self.top_k:
                break
        
        return hard_negatives

三、负样本构造实战组合

推荐组合（国内大厂主流方案）：

训练步前20%：
├── 曝光未点击（INC）打基础分布
├── 随机负例作为正则
└── 比例：INC:随机 = 7:3

训练步中60%：
├── Batch内负采样提速
├── 温度系数τ=0.05~0.1
└── 比例：Batch内:INC = 8:2

训练步后20%：
├── 插入难负例精细化
├── 难负例:Batch内 = 3:7
└── 配合温度衰减

【样本不平衡处理】

一、正负样本比例

场景典型比例：
├── CTR预估：1:10~1:50
├── CVR预估：1:100~1:1000
└── 召回训练：1:100~1:1000

二、处理方法

1. 负采样
   ├── 随机负采样
   ├── 分层负采样
   └── 热度修正负采样

2. 过采样
   ├── 复制正样本
   ├── SMOTE生成正样本
   └── 数据增强

3. 欠采样
   ├── 随机删除负样本
   ├── 聚类欠采样
   └── Tomek Links

4. 损失函数加权
   ├── Focal Loss
   ├── Class-Balanced Loss
   └── 代价敏感学习

代码示例：
class FocalLoss(nn.Module):
    def __init__(self, alpha=0.25, gamma=2.0):
        super().__init__()
        self.alpha = alpha
        self.gamma = gamma
    
    def forward(self, pred, target):
        # pred: 预测概率
        # target: 真实标签
        
        pt = torch.where(target == 1, pred, 1 - pred)
        
        # 难样本（pt小）获得更大权重
        focal_weight = (1 - pt) ** self.gamma
        
        # 正样本权重
        alpha_weight = torch.where(target == 1, self.alpha, 1 - self.alpha)
        
        loss = -alpha_weight * focal_weight * torch.log(pt + 1e-8)
        return loss.mean()

【样本偏差纠正】

一、选择偏差

问题：训练样本非随机采样，存在偏差

解决方案：
1. IPS（逆倾向得分）
   weight = 1 / P(exposure|x)
   
   其中P(exposure|x)是曝光概率

2. 双重稳健估计
   结合IPS和结果回归

二、位置偏差

问题：用户更倾向点击靠前位置的商品

解决方案：
1. 位置作为特征
   ├── 训练时：位置作为输入特征
   └── 推理时：位置设为固定值

2. 位置偏差建模
   ├── 训练时：score = main_score + position_bias
   └── 推理时：score = main_score

代码示例：
class PositionBiasModel(nn.Module):
    def __init__(self, num_features, num_positions):
        super().__init__()
        self.main_tower = nn.Sequential(
            nn.Linear(num_features, 256),
            nn.ReLU(),
            nn.Linear(256, 1)
        )
        self.position_bias = nn.Embedding(num_positions, 1)
    
    def forward(self, features, position):
        main_score = self.main_tower(features)
        position_bias = self.position_bias(position)
        
        if self.training:
            return main_score + position_bias
        else:
            return main_score  # 推理时不加位置偏差

三、流行度偏差

问题：热门商品被过度曝光和点击

解决方案：
1. 流行度降权
   weight = 1 / log(1 + popularity)

2. 对比学习中的流行度修正
   temperature = base_temp * (1 + 0.1 * log(1 + popularity))

【样本穿越问题】

一、什么是样本穿越

定义：训练时使用了线上推理时不可用的信息

常见场景：
├── 使用了"点击后"才能获得的特征
├── 使用了"未来"的特征
└── 使用了"位置"特征

二、样本穿越检测

代码示例：
def check_feature_leakage(sample):
    sample_time = sample['event_time']
    
    for feature_name, feature_value in sample['features'].items():
        feature_time = get_feature_timestamp(feature_name, feature_value)
        
        if feature_time > sample_time:
            log_leakage(feature_name, sample_time, feature_time)
            return True
    
    return False

三、样本穿越修复

解决方案：
1. 特征快照
   ├── 训练时使用特征快照
   ├── 快照时间 < 样本时间
   └── 保证特征可用性

2. 特征对齐
   ├── 训练和推理使用相同的特征计算逻辑
   ├── 统一特征服务
   └── 版本控制

3. 延迟特征处理
   ├── 对延迟到达的特征做特殊处理
   ├── 填充默认值或使用历史值
   └── 记录特征缺失标志
```

---

### 4.2 请详细介绍样本权重的设计方法

**答案：**

```
【样本权重的作用】

样本权重决定每个样本对模型训练的贡献：
├── 重要样本权重高 → 模型更关注
├── 不重要样本权重低 → 模型少关注
└── 合理的权重设计能提升模型效果

【样本权重设计维度】

一、行为维度权重

1. 行为类型权重
   不同行为的价值不同：
   
   weight_map = {
       'click': 1.0,        # 点击
       'add_cart': 2.0,     # 加购
       'collect': 2.5,      # 收藏
       'order': 5.0,        # 下单
       'pay': 10.0          # 支付
   }
   
   设计原则：
   ├── 行为价值越高，权重越大
   ├── 行为成本越高，权重越大
   └── 根据业务目标调整

2. 行为深度权重
   根据用户参与深度加权：
   
   def get_depth_weight(depth):
       if depth == 'view':      # 浏览
           return 1.0
       elif depth == 'detail':  # 详情页
           return 1.5
       elif depth == 'comment': # 评论
           return 3.0
       elif depth == 'share':   # 分享
           return 5.0

二、时间维度权重

1. 时间衰减权重
   近期行为更有价值：
   
   weight = exp(-λ * days_ago)
   
   λ的选择：
   ├── λ=0.1：衰减慢，长期行为重要
   ├── λ=0.3：衰减快，近期行为重要
   └── 根据业务特点调整

2. 时间段权重
   不同时间段行为价值不同：
   
   def get_time_weight(hour):
       if 9 <= hour <= 18:    # 工作时间
           return 1.0
       elif 19 <= hour <= 23: # 晚间高峰
           return 1.5
       else:                  # 深夜
           return 0.8

3. 节假日权重
   节假日行为价值更高：
   
   weight = 2.0 if is_holiday else 1.0

三、用户维度权重

1. 用户活跃度权重
   活跃用户行为更可信：
   
   def get_activity_weight(user_activity):
       if user_activity > 100:   # 高活跃
           return 1.2
       elif user_activity > 10: # 中活跃
           return 1.0
       else:                     # 低活跃
           return 0.8

2. 用户价值权重
   高价值用户行为更重要：
   
   def get_value_weight(user_value):
       if user_value == 'high':
           return 2.0
       elif user_value == 'medium':
           return 1.0
       else:
           return 0.5

四、商品维度权重

1. 商品热度权重
   热门商品行为权重降低：
   
   weight = 1 / log(1 + item_popularity)

2. 商品新鲜度权重
   新商品行为权重提高：
   
   def get_freshness_weight(item_age_days):
       if item_age_days < 7:
           return 2.0  # 新品
       elif item_age_days < 30:
           return 1.5
       else:
           return 1.0

3. 商品价格权重
   高价商品行为权重提高：
   
   weight = log(1 + price) / log(1 + avg_price)

【样本权重实现】

一、训练时权重应用

代码示例：
class WeightedLoss(nn.Module):
    def __init__(self):
        super().__init__()
    
    def forward(self, pred, target, weight):
        """
        pred: 预测值
        target: 真实值
        weight: 样本权重
        """
        # 计算基础损失
        loss = F.binary_cross_entropy(pred, target, reduction='none')
        
        # 应用权重
        weighted_loss = loss * weight
        
        return weighted_loss.mean()

二、权重归一化

问题：权重差异过大会导致训练不稳定

解决方案：
1. Min-Max归一化
   weight_norm = (weight - min) / (max - min)

2. Z-Score归一化
   weight_norm = (weight - mean) / std

3. 对数归一化
   weight_norm = log(1 + weight) / log(1 + max_weight)

【样本权重实验效果】

在某电商场景的实验：

┌─────────────────┬──────────┬──────────┬──────────┐
│ 权重策略        │ AUC      │ CTR提升  │ GMV提升  │
├─────────────────┼──────────┼──────────┼──────────┤
│ 无权重          │ 0.725    │ 基线     │ 基线     │
│ 行为类型权重    │ 0.732    │ +2.1%    │ +3.5%    │
│ 时间衰减权重    │ 0.728    │ +1.5%    │ +2.8%    │
│ 综合权重        │ 0.738    │ +3.2%    │ +5.1%    │
└─────────────────┴──────────┴──────────┴──────────┘

结论：合理的样本权重设计能显著提升模型效果
```

---

## 五、模型评估面试题

### 5.1 请详细介绍推荐系统的评估指标

**答案：**

```
【评估指标分类】

一、离线评估指标

1. AUC（Area Under Curve）
   定义：ROC曲线下面积
   
   含义：随机正样本分数高于随机负样本的概率
   
   计算公式：
   AUC = P(score_positive > score_negative)
   
   优点：
   ├── 不受阈值影响
   ├── 不受正负比例影响
   └── 适合排序问题
   
   缺点：
   ├── 只关注相对排序
   ├── 不关注绝对概率
   └── 对全局排序敏感
   
   典型值：
   ├── 0.5：随机猜测
   ├── 0.7-0.8：中等效果
   ├── 0.8-0.9：较好效果
   └── >0.9：优秀效果

2. GAUC（Group AUC）
   定义：按用户分组计算AUC，再加权平均
   
   公式：
   GAUC = Σ(w_i * AUC_i) / Σw_i
   
   其中：
   ├── AUC_i：用户i的AUC
   └── w_i：用户i的权重（通常用曝光数）
   
   优点：
   ├── 更符合推荐场景
   ├── 消除用户间差异
   └── 与线上效果相关性更高
   
   代码示例：
   def calculate_gauc(predictions, labels, user_ids):
       # 按用户分组
       user_data = defaultdict(list)
       for pred, label, user_id in zip(predictions, labels, user_ids):
           user_data[user_id].append((pred, label))
       
       # 计算每个用户的AUC
       total_weight = 0
       weighted_auc = 0
       
       for user_id, data in user_data.items():
           preds = [d[0] for d in data]
           labels = [d[1] for d in data]
           
           # 至少有一个正样本和一个负样本
           if sum(labels) > 0 and sum(labels) < len(labels):
               auc = roc_auc_score(labels, preds)
               weight = len(labels)
               weighted_auc += auc * weight
               total_weight += weight
       
       return weighted_auc / total_weight if total_weight > 0 else 0.5

3. LogLoss
   定义：对数损失
   
   公式：
   LogLoss = -1/N * Σ(y*log(p) + (1-y)*log(1-p))
   
   优点：
   ├── 关注概率准确性
   ├── 可用于校准评估
   └── 对极端预测敏感
   
   典型值：
   ├── 0.69：随机猜测（ln(2)）
   ├── 0.3-0.5：中等效果
   └── <0.3：较好效果

4. NDCG（Normalized Discounted Cumulative Gain）
   定义：考虑位置的排序质量指标
   
   公式：
   DCG@K = Σ(rel_i / log2(i+1))
   NDCG@K = DCG@K / IDCG@K
   
   其中：
   ├── rel_i：位置i的相关性
   ├── IDCG：理想情况下的DCG
   └── K：截断位置
   
   优点：
   ├── 考虑位置因素
   ├── 考虑相关性程度
   └── 适合排序评估
   
   代码示例：
   def ndcg_at_k(relevances, k):
       # DCG
       dcg = 0
       for i, rel in enumerate(relevances[:k]):
           dcg += rel / np.log2(i + 2)  # i+2因为位置从1开始
       
       # IDCG
       ideal_relevances = sorted(relevances, reverse=True)
       idcg = 0
       for i, rel in enumerate(ideal_relevances[:k]):
           idcg += rel / np.log2(i + 2)
       
       return dcg / idcg if idcg > 0 else 0

5. MRR（Mean Reciprocal Rank）
   定义：第一个正确答案的位置倒数的平均值
   
   公式：
   MRR = 1/N * Σ(1 / rank_i)
   
   其中rank_i是第i个查询的第一个正确答案位置
   
   优点：
   ├── 关注最相关结果
   ├── 计算简单
   └── 适合搜索场景

二、在线评估指标

1. CTR（Click-Through Rate）
   定义：点击率
   
   公式：
   CTR = 点击数 / 曝光数
   
   特点：
   ├── 最核心的业务指标
   ├── 直接反映推荐质量
   └── 受位置影响大

2. CVR（Conversion Rate）
   定义：转化率
   
   公式：
   CVR = 转化数 / 点击数
   
   特点：
   ├── 反映推荐精准度
   ├── 数据稀疏
   └── 延迟高

3. GMV（Gross Merchandise Volume）
   定义：成交金额
   
   公式：
   GMV = Σ(订单金额)
   
   特点：
   ├── 最核心的商业指标
   ├── 综合反映推荐效果
   └── 受价格影响

4. 人均停留时长
   定义：用户在推荐页面的平均停留时间
   
   特点：
   ├── 反映用户粘性
   ├── 受内容质量影响
   └── 适合内容推荐

5. 负反馈率
   定义：用户负反馈的比例
   
   包括：
   ├── 不感兴趣
   ├── 举报
   └── 拉黑
   
   特点：
   ├── 反映推荐质量
   ├── 应该尽量降低
   └── 权重应该很高

【评估指标选择】

不同场景关注不同指标：

电商场景：
├── 核心：GMV、CVR
├── 次要：CTR、客单价
└── 辅助：停留时长、复购率

内容场景：
├── 核心：停留时长、完播率
├── 次要：CTR、互动率
└── 辅助：分享率、负反馈率

广告场景：
├── 核心：CPM、ROI
├── 次要：CTR、CVR
└── 辅助：点击成本、转化成本

【A/B实验评估】

一、实验设计

1. 流量分配
   ├── 随机分流：用户ID哈希
   ├── 正交实验：多实验并行
   └── 互斥实验：实验间互斥

2. 样本量计算
   公式：
   N = 16 * σ² / δ²
   
   其中：
   ├── σ²：指标方差
   ├── δ：预期提升
   └── 16来自(1.96+0.84)²

3. 实验周期
   ├── 最短周期：7天（覆盖完整周期）
   ├── 推荐周期：14天（覆盖用户行为周期）
   └── 长期实验：28天（观察长期效果）

二、显著性检验

1. t检验
   适用：正态分布、样本量大
   
   公式：
   t = (μ1 - μ2) / sqrt(s1²/n1 + s2²/n2)

2. Mann-Whitney U检验
   适用：非正态分布
   
   优点：
   ├── 不要求正态分布
   ├── 对异常值不敏感
   └── 适合业务指标

三、效果评估

代码示例：
def evaluate_experiment(control_data, treatment_data, metric):
    """
    评估A/B实验效果
    """
    # 计算均值
    control_mean = np.mean(control_data[metric])
    treatment_mean = np.mean(treatment_data[metric])
    
    # 计算提升
    lift = (treatment_mean - control_mean) / control_mean
    
    # 显著性检验
    t_stat, p_value = stats.ttest_ind(
        control_data[metric], 
        treatment_data[metric]
    )
    
    # 置信区间
    se = np.sqrt(
        np.var(control_data[metric]) / len(control_data) +
        np.var(treatment_data[metric]) / len(treatment_data)
    )
    ci_low = lift - 1.96 * se / control_mean
    ci_high = lift + 1.96 * se / control_mean
    
    return {
        'control_mean': control_mean,
        'treatment_mean': treatment_mean,
        'lift': lift,
        'p_value': p_value,
        'significant': p_value < 0.05,
        'ci': (ci_low, ci_high)
    }
```

---

## 六、系统工程面试题

### 6.1 请详细介绍推荐系统的性能优化方法

**答案：**

```
【性能优化目标】

推荐系统性能要求：
├── P99延迟 < 30ms
├── QPS > 5万
├── 可用性 > 99.99%
└── 成本可控

【性能优化方法论】

一、先监控，后优化

1. 建立监控体系
   ├── 延迟监控：P50、P90、P99
   ├── QPS监控：实时QPS、峰值QPS
   ├── 资源监控：CPU、内存、网络IO
   └── 错误监控：错误率、超时率

2. 定位瓶颈
   ├── 调用链分析：哪个环节耗时高
   ├── 热点分析：哪个函数CPU占用高
   └── 资源分析：哪个资源是瓶颈

二、先架构，后代码

1. 架构优化收益大
   ├── 缓存：减少IO
   ├── 异步：减少等待
   └── 并行：减少总时间

2. 代码优化收益小
   ├── 算法优化：提升有限
   └── 逻辑优化：效果有限

【具体优化方法】

一、减少IO

1. 缓存优化
   问题：数据库查询慢
   
   方案：
   ├── 本地缓存：Guava Cache、Caffeine
   ├── 分布式缓存：Redis
   └── 多级缓存：本地 + 分布式
   
   代码示例：
   @Service
   public class FeatureService {
       // 本地缓存
       private final Cache<String, Map<String, String>> localCache = 
           Caffeine.newBuilder()
               .maximumSize(100000)
               .expireAfterWrite(1, TimeUnit.MINUTES)
               .build();
       
       @Resource
       private RedisTemplate<String, String> redisTemplate;
       
       public Map<String, String> getFeatures(String key) {
           // 1. 查本地缓存
           Map<String, String> result = localCache.getIfPresent(key);
           if (result != null) {
               return result;
           }
           
           // 2. 查Redis
           result = redisTemplate.opsForHash().entries(key);
           if (result != null && !result.isEmpty()) {
               localCache.put(key, result);
               return result;
           }
           
           // 3. 查数据库
           result = queryFromDB(key);
           if (result != null) {
               redisTemplate.opsForHash().putAll(key, result);
               localCache.put(key, result);
           }
           
           return result;
       }
   }

2. Pipeline批量查询
   问题：多次网络IO
   
   方案：一次网络IO获取多个数据
   
   优化前：
   for (String key : keys) {
       String value = redis.get(key);  // N次网络IO
   }
   
   优化后：
   Pipeline pipeline = redis.pipelined();
   for (String key : keys) {
       pipeline.get(key);  // 1次网络IO
   }
   List<Object> results = pipeline.syncAndReturnAll();

3. 预取
   问题：实时获取延迟高
   
   方案：提前获取可能需要的数据
   
   实现：
   ├── 召回阶段预取特征
   ├── 用户进入页面时预取
   └── 后台定时预热

二、并行化

1. 多线程并行
   问题：串行执行慢
   
   方案：并行执行
   
   优化前：
   List<Item> items1 = recallChannel1.recall();  // 10ms
   List<Item> items2 = recallChannel2.recall();  // 10ms
   List<Item> items3 = recallChannel3.recall();  // 10ms
   // 总耗时：30ms
   
   优化后：
   CompletableFuture<List<Item>> f1 = CompletableFuture.supplyAsync(
       () -> recallChannel1.recall(), executor
   );
   CompletableFuture<List<Item>> f2 = CompletableFuture.supplyAsync(
       () -> recallChannel2.recall(), executor
   );
   CompletableFuture<List<Item>> f3 = CompletableFuture.supplyAsync(
       () -> recallChannel3.recall(), executor
   );
   CompletableFuture.allOf(f1, f2, f3).join();
   // 总耗时：10ms + 线程切换开销

2. 批量处理
   问题：逐个处理慢
   
   方案：批量处理
   
   优化前：
   for (Item item : items) {
       Feature feature = featureService.getFeature(item.getId());
       double score = model.predict(feature);
   }
   
   优化后：
   List<Feature> features = featureService.batchGetFeatures(itemIds);
   List<Double> scores = model.batchPredict(features);

三、减少计算

1. 算法优化
   问题：算法复杂度高
   
   方案：选择更优算法
   
   示例：
   ├── 排序：O(n²) → O(n log n)
   ├── 查找：O(n) → O(log n)
   └── 去重：O(n²) → O(n)

2. 预计算
   问题：实时计算慢
   
   方案：离线预计算
   
   示例：
   ├── 商品相似度：离线计算
   ├── 用户画像：离线计算
   └── 统计特征：离线计算

3. 增量计算
   问题：全量计算慢
   
   方案：只计算变化部分
   
   示例：
   ├── 用户CTR：增量更新
   ├── 商品热度：增量更新
   └── 实时特征：增量更新

四、资源优化

1. 连接池优化
   问题：连接创建销毁开销大
   
   方案：使用连接池
   
   配置：
   ├── 最大连接数：根据QPS和延迟计算
   ├── 最小连接数：保持一定数量的连接
   └── 连接超时：合理设置超时时间

2. 线程池优化
   问题：线程创建销毁开销大
   
   方案：使用线程池
   
   配置：
   ├── 核心线程数：CPU密集型 = CPU核数
   ├── 最大线程数：IO密集型 = CPU核数 * 2
   └── 队列大小：根据任务特点设置

3. 内存优化
   问题：内存占用高、GC频繁
   
   方案：
   ├── 对象池：复用对象
   ├── 减少对象创建：使用基本类型
   └── 内存对齐：减少内存碎片

【性能优化案例】

案例：精排服务P99延迟优化

问题：
├── P99延迟从15ms涨到35ms
├── 特征获取占比最高（60%）
└── 影响整体推荐延迟

排查：
├── 分析调用链：特征获取耗时高
├── 分析特征获取：Redis查询次数多
└── 定位根因：未使用Pipeline

优化：
├── Pipeline批量查询
├── 本地缓存
└── 并行获取

效果：
├── 特征获取：15ms → 3ms
├── 精排P99：35ms → 15ms
└── 整体P99：55ms → 25ms

代码示例：
// 优化前
public Map<String, String> getFeatures(List<String> keys) {
    Map<String, String> result = new HashMap<>();
    for (String key : keys) {
        String value = redis.get(key);  // N次网络IO
        result.put(key, value);
    }
    return result;
}

// 优化后
public Map<String, String> getFeatures(List<String> keys) {
    Map<String, String> result = new HashMap<>();
    
    try (Jedis jedis = jedisPool.getResource()) {
        Pipeline pipeline = jedis.pipelined();
        
        // 批量查询
        List<Response<String>> responses = new ArrayList<>();
        for (String key : keys) {
            responses.add(pipeline.get(key));
        }
        
        pipeline.sync();  // 1次网络IO
        
        // 解析结果
        for (int i = 0; i < keys.size(); i++) {
            result.put(keys.get(i), responses.get(i).get());
        }
    }
    
    return result;
}
```

---

由于内容较多，我将继续添加更多面试题...

---

## 七、系统工程面试题

### 7.1 请详细介绍推荐系统的架构设计

**答案：**

```
【推荐系统整体架构】

┌─────────────────────────────────────────────────────────────────┐
│                        用户请求                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API网关层                                 │
│  ├── 请求鉴权、限流、路由                                      │
│  └── 监控埋点、日志记录                                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      推荐服务层                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ 召回服务  │→│ 粗排服务  │→│ 精排服务  │→│ 重排服务  │       │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      支撑服务层                                │
│  ├── 特征服务：特征获取、缓存                                 │
│  ├── 模型服务：模型推理、热更新                               │
│  ├── 配置服务：配置管理、下发                                 │
│  └── 实验服务：A/B实验、灰度                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      数据层                                    │
│  ├── Redis：特征缓存、实时数据                                │
│  ├── MySQL：业务数据                                          │
│  ├── ES：检索索引                                             │
│  └── Kafka：消息队列                                          │
└─────────────────────────────────────────────────────────────────┘

【各层职责详解】

一、网关层

职责：
├── 流量管理：限流、熔断、降级
├── 安全防护：鉴权、防攻击
├── 路由分发：负载均衡、服务发现
└── 监控埋点：日志、指标采集

技术选型：
├── Nginx：高性能反向代理
├── OpenResty：Nginx + Lua
└── Kong：API网关

性能要求：
├── P99延迟 < 5ms
├── QPS > 50万
└── 可用性 > 99.99%

二、推荐服务层

1. 召回服务
   职责：从海量商品中筛选候选集
   
   特点：
   ├── 候选集大：百万级 → 千级
   ├── 速度快：P99 < 15ms
   └── 多路召回：Item-CF、向量、图
   
   技术选型：
   ├── 向量检索：Faiss、Milvus
   ├── 倒排索引：Elasticsearch
   └── 图检索：Neo4j

2. 粗排服务
   职责：轻量模型快速筛选
   
   特点：
   ├── 候选集：千级 → 百级
   ├── 速度快：P99 < 5ms
   └── 模型轻：双塔、小规模MLP

3. 精排服务
   职责：复杂模型精准打分
   
   特点：
   ├── 候选集：百级 → 十级
   ├── 精度高：AUC > 0.73
   └── 模型复杂：DIN、DIEN、DCN
   
   技术选型：
   ├── 模型服务：TensorFlow Serving
   ├── 特征服务：Redis + 本地缓存
   └── 计算框架：TensorFlow、PyTorch

4. 重排服务
   职责：业务规则处理
   
   特点：
   ├── 多样性：MMR、DPP
   ├── 去重：过滤已曝光商品
   └── 规则：业务规则过滤
   
   性能要求：P99 < 3ms

三、支撑服务层

1. 特征服务
   职责：高效提供特征
   
   架构：
   ├── 在线特征：Redis实时特征
   ├── 近线特征：Flink实时计算
   └── 离线特征：离线批处理
   
   优化：
   ├── 多级缓存：本地 + Redis
   ├── Pipeline批量查询
   └── 特征预取

2. 模型服务
   职责：模型推理
   
   架构：
   ├── 模型存储：模型仓库
   ├── 模型服务：TF Serving、ONNX
   └── 模型更新：热更新机制
   
   优化：
   ├── 批量推理
   ├── 模型量化
   └── GPU加速

3. 配置服务
   职责：配置管理
   
   内容：
   ├── 召回配置：召回通道、配额
   ├── 排序配置：模型版本、参数
   └── 规则配置：业务规则

4. 实验服务
   职责：A/B实验
   
   功能：
   ├── 实验配置：分流策略
   ├── 实验执行：流量分流
   └── 实验分析：效果评估

四、数据层

1. Redis
   用途：
   ├── 特征缓存
   ├── 实时数据
   └── 计数器
   
   架构：
   ├── 集群模式：分片 + 副本
   ├── 内存优化：数据压缩
   └── 持久化：RDB + AOF

2. MySQL
   用途：
   ├── 业务数据
   ├── 用户数据
   └── 商品数据
   
   优化：
   ├── 读写分离
   ├── 分库分表
   └── 索引优化

3. Elasticsearch
   用途：
   ├── 倒排索引
   ├── 全文检索
   └── 聚合分析
   
   优化：
   ├── 索引设计
   ├── 分片策略
   └── 查询优化

4. Kafka
   用途：
   ├── 行为日志
   ├── 实时流
   └── 系统解耦
   
   架构：
   ├── 分区：提高并行度
   ├── 副本：提高可靠性
   └── 消费组：负载均衡

【架构设计原则】

一、高可用设计

1. 无单点
   ├── 服务多副本
   ├── 数据多副本
   └── 依赖多实例

2. 故障隔离
   ├── 服务隔离：不同服务独立部署
   ├── 数据隔离：不同数据分片
   └── 流量隔离：不同流量独立

3. 快速恢复
   ├── 健康检查：定期检查服务状态
   ├── 自动重启：服务异常自动重启
   └── 自动切换：主备自动切换

二、高性能设计

1. 缓存
   ├── 多级缓存
   ├── 缓存预热
   └── 缓存更新策略

2. 异步
   ├── 异步处理
   ├── 消息队列
   └── 回调机制

3. 并行
   ├── 多线程
   ├── 多进程
   └── 分布式计算

三、可扩展设计

1. 水平扩展
   ├── 无状态服务
   ├── 数据分片
   └── 负载均衡

2. 垂直扩展
   ├── 升级硬件
   └── 优化配置

【架构演进】

阶段一：单体架构
├── 特点：所有功能在一个服务
├── 优点：简单、易部署
└── 缺点：难扩展、难维护

阶段二：微服务架构
├── 特点：按功能拆分服务
├── 优点：易扩展、易维护
└── 缺点：复杂度高、运维难

阶段三：云原生架构
├── 特点：容器化、服务网格
├── 优点：弹性伸缩、高可用
└── 缺点：技术门槛高
```

---

## 八、业务场景面试题

### 8.1 请详细介绍推荐系统的冷启动问题及解决方案

**答案：**

```
【冷启动问题定义】

冷启动问题：新用户或新商品缺乏历史数据，导致推荐效果差

类型：
├── 用户冷启动：新用户无行为数据
├── 商品冷启动：新商品无交互数据
└── 系统冷启动：新系统无任何数据

【用户冷启动解决方案】

一、基于人口统计学特征

方法：利用用户注册信息

可用特征：
├── 年龄：不同年龄段偏好不同
├── 性别：男女偏好差异
├── 地域：不同地区偏好不同
└── 职业：不同职业偏好不同

实现：
1. 建立人群画像
   ├── 统计不同人群的偏好分布
   └── 为新用户匹配人群画像

2. 推荐人群热门商品
   ├── 推荐该人群喜欢的商品
   └── 结合全局热门商品

代码示例：
def cold_start_user(user_profile):
    # 获取人群标签
    group = get_user_group(user_profile)
    
    # 获取人群热门商品
    group_popular = get_group_popular_items(group)
    
    # 获取全局热门商品
    global_popular = get_global_popular_items()
    
    # 混合推荐
    items = mix_items(group_popular, global_popular, ratio=0.7)
    
    return items

二、基于注册引导

方法：用户注册时收集偏好

实现：
1. 兴趣选择
   ├── 让用户选择感兴趣的类目
   ├── 让用户选择感兴趣的品牌
   └── 让用户关注感兴趣的用户

2. 问卷引导
   ├── 设计简短问卷
   ├── 了解用户偏好
   └── 根据回答推荐

三、基于探索利用

方法：给新用户展示多样化内容，观察反馈

实现：
1. 多样化探索
   ├── 展示不同类目商品
   ├── 展示不同风格商品
   └── 观察用户点击

2. 快速适应
   ├── 根据少量行为快速建模
   ├── 使用元学习快速适应
   └── 动态调整推荐策略

代码示例：
class ColdStartExplorer:
    def __init__(self, num_slots=10):
        self.num_slots = num_slots
    
    def get_explore_items(self, user_id):
        # 分配探索槽位
        items = []
        
        # 30% 全局热门
        items.extend(get_global_popular(self.num_slots * 0.3))
        
        # 40% 类目探索（不同类目）
        items.extend(get_category_explore(self.num_slots * 0.4))
        
        # 30% 随机探索
        items.extend(get_random_items(self.num_slots * 0.3))
        
        return items

【商品冷启动解决方案】

一、基于内容特征

方法：利用商品属性进行推荐

可用特征：
├── 类目：同类目商品相似
├── 品牌：同品牌商品相似
├── 属性：相同属性商品相似
└── 内容：标题、图片、视频

实现：
1. 内容相似度计算
   ├── 标题相似度：文本相似度
   ├── 图片相似度：图像相似度
   └── 属性相似度：属性匹配

2. 向量化表示
   ├── 标题向量：BERT、Word2Vec
   ├── 图片向量：CNN提取
   └── 多模态融合

代码示例：
def cold_start_item(item_features):
    # 获取商品向量
    item_vector = get_item_vector(item_features)
    
    # 查找相似商品
    similar_items = find_similar_items(item_vector, top_k=100)
    
    # 获取相似商品的用户
    potential_users = get_users_from_items(similar_items)
    
    return potential_users

二、基于探索流量

方法：给新商品分配探索流量

实现：
1. 保量机制
   ├── 新商品强制曝光
   ├── 保证最低曝光量
   └── 观察用户反馈

2. 流量分配
   ├── 根据反馈调整流量
   ├── 表现好增加流量
   └── 表现差减少流量

代码示例：
class ItemExplorer:
    def __init__(self, min_exposure=100):
        self.min_exposure = min_exposure
    
    def allocate_traffic(self, item_id, item_stats):
        # 新商品保量
        if item_stats.exposure < self.min_exposure:
            return self.min_exposure - item_stats.exposure
        
        # 根据表现分配
        ctr = item_stats.click / item_stats.exposure
        if ctr > 0.05:  # 表现好
            return 100
        elif ctr > 0.02:  # 表现中等
            return 50
        else:  # 表现差
            return 10

三、基于迁移学习

方法：利用已有商品的知识

实现：
1. 元学习
   ├── MAML：快速适应新商品
   ├── 学习初始化参数
   └── 少量样本快速适应

2. 知识迁移
   ├── 从老商品迁移知识
   ├── 利用商品属性关联
   └── 跨域迁移

【系统冷启动解决方案】

一、利用公开数据

方法：使用公开数据集预训练

实现：
├── 使用公开推荐数据集
├── 预训练推荐模型
└── 迁移到新系统

二、利用规则系统

方法：先使用规则系统，逐步过渡

实现：
├── 热门推荐：推荐热门商品
├── 最新推荐：推荐最新商品
└── 规则推荐：基于简单规则

三、快速积累数据

方法：快速积累用户行为数据

实现：
├── 引导用户行为
├── 激励用户互动
└── 快速迭代模型

【冷启动评估】

评估指标：
├── 新用户CTR：新用户的点击率
├── 新商品曝光率：新商品的曝光比例
├── 覆盖率：新商品/新用户的覆盖比例
└── 首次点击时间：新用户首次点击的时间

实验方法：
├── 分层评估：分别评估新/老用户
├── 时间窗口：观察冷启动效果随时间变化
└── A/B实验：对比不同冷启动策略
```

---

### 8.2 请详细介绍推荐系统的实时性优化

**答案：**

```
【实时性需求】

推荐系统需要实时响应用户行为：
├── 用户刚点击了一个商品，后续推荐应该相关
├── 用户刚搜索了一个词，推荐应该相关
└── 商品刚上架，应该有机会曝光

【实时性架构】

一、实时特征更新

架构：
┌─────────────┐
│ 用户行为    │
└─────────────┘
       │
       ▼
┌─────────────┐
│ Kafka       │ ← 行为日志
└─────────────┘
       │
       ▼
┌─────────────┐
│ Flink       │ ← 实时计算
└─────────────┘
       │
       ▼
┌─────────────┐
│ Redis       │ ← 实时特征
└─────────────┘
       │
       ▼
┌─────────────┐
│ 推荐服务    │ ← 使用实时特征
└─────────────┘

实现：
1. 实时特征类型
   ├── 实时CTR：最近1小时点击率
   ├── 实时CVR：最近1小时转化率
   ├── 最近行为：最近点击商品
   └── 实时热度：商品实时热度

2. Flink实时计算
   代码示例：
   public class RealtimeFeatureJob {
       public static void main(String[] args) {
           StreamExecutionEnvironment env = 
               StreamExecutionEnvironment.getExecutionEnvironment();
           
           // Kafka数据源
           FlinkKafkaConsumer<UserBehavior> source = 
               new FlinkKafkaConsumer<>("user_behavior", ...);
           
           DataStream<UserBehavior> stream = env.addSource(source);
           
           // 实时CTR计算
           stream.keyBy(UserBehavior::getItemId)
               .window(SlidingEventTimeWindows.of(
                   Time.hours(1),    // 窗口大小
                   Time.minutes(5)   // 滑动步长
               ))
               .process(new CtrCalculator())
               .addSink(new RedisSink());
           
           env.execute();
       }
   }

二、实时模型更新

架构：
├── 在线学习：实时更新模型
├── 增量训练：定期增量更新
└── 多版本模型：快速切换模型

实现：
1. 在线学习
   ├── FTRL：在线逻辑回归
   ├── Online SGD：在线梯度下降
   └── 实时更新Embedding

2. 增量训练
   ├── 定期增量训练（小时级）
   ├── 只更新变化的部分
   └── 快速迭代

三、实时召回更新

架构：
├── 实时向量更新：用户向量实时更新
├── 实时倒排索引：索引实时更新
└── 实时图更新：图结构实时更新

实现：
1. 用户向量实时更新
   def update_user_vector(user_id, new_behavior):
       # 获取当前用户向量
       current_vector = get_user_vector(user_id)
       
       # 获取新行为商品向量
       item_vector = get_item_vector(new_behavior.item_id)
       
       # 更新用户向量
       new_vector = 0.8 * current_vector + 0.2 * item_vector
       
       # 写入Redis
       set_user_vector(user_id, new_vector)

2. 实时倒排索引
   ├── 新商品实时加入索引
   ├── 用户行为实时更新索引
   └── 索引增量更新

【实时性优化案例】

案例：用户刚点击了一个手机，后续推荐应该相关

问题：
├── 用户点击手机后，推荐列表没有变化
├── 延迟高，特征更新慢
└── 影响用户体验

解决方案：
1. 实时特征更新
   ├── 用户点击后，立即更新用户实时特征
   ├── 最近点击商品列表实时更新
   └── 实时偏好类目实时更新

2. 实时召回更新
   ├── 用户向量实时更新
   ├── Item-CF相似商品召回
   └── 同类目商品召回

3. 实时排序调整
   ├── 点击商品相似商品加权
   ├── 同类目商品加权
   └── 实时特征加权

实现：
def realtime_recommend(user_id, recent_click):
    # 1. 更新用户实时特征
    update_user_realtime_features(user_id, recent_click)
    
    # 2. 实时召回
    recall_items = realtime_recall(user_id, recent_click)
    
    # 3. 实时排序
    ranked_items = realtime_rank(user_id, recall_items, recent_click)
    
    return ranked_items

def realtime_recall(user_id, recent_click):
    items = []
    
    # 点击商品相似商品
    similar_items = get_similar_items(recent_click.item_id)
    items.extend(similar_items)
    
    # 同类目热门商品
    category = get_item_category(recent_click.item_id)
    category_hot = get_category_hot(category)
    items.extend(category_hot)
    
    # 用户向量召回
    user_vector = get_user_vector(user_id)
    vector_items = vector_recall(user_vector)
    items.extend(vector_items)
    
    return items

def realtime_rank(user_id, items, recent_click):
    for item in items:
        score = item.base_score
        
        # 点击商品相似加权
        if item.similar_to(recent_click.item_id):
            score *= 1.5
        
        # 同类目加权
        if item.same_category(recent_click.item_id):
            score *= 1.3
        
        item.final_score = score
    
    return sorted(items, key=lambda x: x.final_score, reverse=True)

效果：
├── 用户点击后，推荐列表实时更新
├── 相关性提升30%
└── 用户满意度提升
```

---

## 总结

本文档涵盖了推荐算法工程师面试的核心问题，包括：

1. **召回模块**：Item-CF、向量召回、多路召回
2. **排序模块**：模型发展、多任务学习
3. **特征工程**：特征类型、处理方法
4. **样本工程**：样本构造、权重设计
5. **模型评估**：评估指标、A/B实验
6. **系统工程**：架构设计、性能优化
7. **业务场景**：冷启动、实时性

每道题都提供了详细的答案和代码示例，希望能帮助你在面试中取得好成绩！

---

[← 上一章：网关与服务端工作详解](18-gateway-server-work.md) | [返回目录](../README.md)
