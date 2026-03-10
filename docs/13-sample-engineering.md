# 十三、样本工程：顶级算法工程师的核心技能

> 样本决定模型上限，特征决定模型下限。顶级算法工程师在样本处理上有极其精细的方法论。

---

## 13.1 样本构造的核心原则

### 黄金法则

```
样本质量 > 样本数量 > 特征工程 > 模型结构
```

### 样本构造三要素

| 要素 | 说明 | 关键指标 |
|-----|------|---------|
| **正样本定义** | 什么行为算"正" | 业务目标对齐 |
| **负样本构造** | 什么行为算"负" | 分布一致性 |
| **样本权重** | 不同样本的重要性 | 梯度平衡 |

---

## 13.2 正样本构造策略

### 正样本来源

| 来源 | 置信度 | 适用场景 |
|-----|--------|---------|
| 点击 | 高 | CTR预估 |
| 转化 | 极高 | CVR预估 |
| 深度浏览 | 中 | 时长预估 |
| 加购/收藏 | 高 | 意向预估 |

### 正样本时间窗口

```
问题：转化可能延迟数小时甚至数天

解决方案：
1. 延迟归因窗口
   ├── 点击后24小时内转化 → 归因到点击
   ├── 点击后7天内转化 → 归因到点击
   └── 根据业务特点调整

2. 生存分析建模
   ├── 拟合转化延迟分布（威布尔分布）
   ├── 计算t95（95%转化发生的时间）
   └── 用t95作为归因窗口

3. 实时+延迟双轨
   ├── 实时样本：点击后立即构造（label=0）
   ├── 延迟更新：转化发生后更新label
   └── 样本回填机制
```

### 正样本置信度加权

```python
# 根据行为深度加权
def get_positive_weight(behavior):
    weight_map = {
        'click': 1.0,
        'add_cart': 2.0,
        'collect': 2.5,
        'order': 5.0,
        'pay': 10.0
    }
    return weight_map.get(behavior, 1.0)

# 根据停留时长加权
def get_dwell_weight(dwell_time_seconds):
    if dwell_time_seconds < 5:
        return 0.3  # 快速划走，低置信
    elif dwell_time_seconds < 30:
        return 0.7
    elif dwell_time_seconds < 120:
        return 1.0
    else:
        return 1.5  # 深度浏览，高置信
```

---

## 13.3 负样本构造策略（核心难点）

### 四种负样本构造方法对比

| 方法 | 定义 | 优点 | 缺点 | 适用场景 |
|-----|------|------|------|---------|
| **随机负采样** | 全局均匀抽取 | 实现简单、训练快 | 与线上分布差异大 | 快速验证 |
| **Batch内负采样** | 同批次其他样本作负例 | 显存零增加、同分布 | 热门Item被过度惩罚 | 大规模训练 |
| **曝光未点击** | 真实曝光但未点击 | 最接近线上分布 | 位置偏差、冷启动覆盖不足 | 精细化优化 |
| **难负例挖掘** | 模型高分但无正反馈 | 样本难度可控 | 计算链路重 | 效果优先 |

### 难负例挖掘流程

```
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
```

### 负样本构造实战组合

```
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
```

---

## 13.4 样本不平衡处理

### 正负样本比例

| 场景 | 典型正负比 | 处理策略 |
|-----|-----------|---------|
| CTR预估 | 1:10~1:50 | 负采样 |
| CVR预估 | 1:100~1:1000 | 分层采样+ESMM |
| 召回训练 | 1:100~1:1000 | 负采样+温度缩放 |

### 负采样策略

```python
# 热度修正负采样
def popularity_corrected_sampling(item_id, item_freq, alpha=0.75):
    """
    降低热门商品的采样概率
    避免热门商品被过度惩罚
    """
    # 原始采样概率与频率的alpha次方成正比
    # alpha < 1 时，降低热门商品的权重
    prob = item_freq[item_id] ** alpha
    return prob

# 分层负采样
def stratified_negative_sampling(user_id, item_pool, k=100):
    """
    分层采样，保证多样性
    """
    negatives = []
    
    # 30% 来自同类目（难负例）
    same_category = sample_from_category(user_category, k*0.3)
    negatives.extend(same_category)
    
    # 40% 来自热门商品（中等难度）
    popular = sample_popular_items(k*0.4)
    negatives.extend(popular)
    
    # 30% 随机采样（简单负例）
    random = random_sample(k*0.3)
    negatives.extend(random)
    
    return negatives
```

### 样本权重策略

```python
# Focal Loss处理类别不平衡
class FocalLoss(nn.Module):
    def __init__(self, alpha=0.25, gamma=2.0):
        super().__init__()
        self.alpha = alpha
        self.gamma = gamma
    
    def forward(self, pred, target):
        """
        pred: 预测概率
        target: 真实标签
        """
        pt = torch.where(target == 1, pred, 1 - pred)
        
        # 难样本（pt小）获得更大权重
        focal_weight = (1 - pt) ** self.gamma
        
        # 正样本权重
        alpha_weight = torch.where(target == 1, self.alpha, 1 - self.alpha)
        
        loss = -alpha_weight * focal_weight * torch.log(pt + 1e-8)
        return loss.mean()

# 类别平衡权重
def class_balanced_weight(num_samples_per_class, beta=0.999):
    """
    基于有效样本数的类别平衡权重
    """
    effective_num = 1.0 - beta ** num_samples_per_class
    weights = (1.0 - beta) / effective_num
    weights = weights / weights.sum() * len(num_samples_per_class)
    return weights
```

---

## 13.5 样本偏差纠正

### 常见偏差类型

| 偏差类型 | 定义 | 解决方案 |
|---------|------|---------|
| **选择偏差** | 训练样本非随机采样 | IPS、DR |
| **位置偏差** | 用户更倾向点击靠前位置 | 位置去偏 |
| **曝光偏差** | 只有部分商品被曝光 | 曝光模型 |
| **流行度偏差** | 热门商品过度曝光 | 流行度降权 |

### 位置偏差纠正

```python
# 逆倾向加权（IPS）
def position_debias_weight(position, click_prob_at_position):
    """
    计算位置去偏权重
    
    position: 商品展示位置
    click_prob_at_position: 该位置的点击概率（从随机实验获得）
    """
    # 权重 = 1 / P(click|position)
    weight = 1.0 / click_prob_at_position[position]
    
    # 裁剪防止极端值
    weight = min(weight, 10.0)
    
    return weight

# 位置特征分离
class PositionBiasModel(nn.Module):
    """
    将位置偏差作为独立任务建模
    """
    def __init__(self, num_positions, embed_dim):
        super().__init__()
        self.position_embed = nn.Embedding(num_positions, embed_dim)
        self.bias_tower = nn.Linear(embed_dim, 1)
        self.main_tower = MainModel()  # 主模型
    
    def forward(self, features, position):
        # 主模型输出
        main_score = self.main_tower(features)
        
        # 位置偏差
        pos_embed = self.position_embed(position)
        bias_score = self.bias_tower(pos_embed)
        
        # 训练时：分数 = 主分数 + 位置偏差
        # 推理时：分数 = 主分数（位置偏差置0）
        if self.training:
            return main_score + bias_score
        else:
            return main_score
```

### 流行度偏差纠正

```python
# 流行度降权
def popularity_debias_weight(item_id, item_freq, alpha=0.5):
    """
    降低热门商品的权重
    """
    freq = item_freq[item_id]
    max_freq = max(item_freq.values())
    
    # 权重与流行度成反比
    weight = 1.0 / (1.0 + (freq / max_freq) ** alpha)
    
    return weight

# 对比学习中的流行度修正
def popularity_corrected_temperature(item_freq, base_temp=0.07):
    """
    根据流行度动态调整温度系数
    """
    # 热门商品用更高的温度（更平滑）
    # 冷门商品用更低的温度（更尖锐）
    temp = base_temp * (1.0 + 0.1 * np.log1p(item_freq))
    return temp
```

---

## 13.6 样本穿越问题

### 什么是样本穿越

```
定义：训练时使用了线上推理时不可用的信息

常见场景：
1. 使用了"点击后"才能获得的特征
   └── 如：停留时长、是否转化
   
2. 使用了"未来"的特征
   └── 如：商品后续销量、用户后续行为
   
3. 使用了"位置"特征
   └── 推理时位置未知
```

### 样本穿越检测

```python
# 特征时间戳校验
def check_feature_leakage(sample):
    """
    检测特征穿越
    """
    sample_time = sample['event_time']
    
    for feature_name, feature_value in sample['features'].items():
        # 获取特征的时间戳
        feature_time = get_feature_timestamp(feature_name, feature_value)
        
        # 如果特征时间 > 样本时间，则存在穿越
        if feature_time > sample_time:
            log_leakage(feature_name, sample_time, feature_time)
            return True
    
    return False

# 特征可用性监控
class FeatureAvailabilityMonitor:
    """
    监控特征在训练和推理时的可用性差异
    """
    def __init__(self):
        self.train_stats = {}
        self.serve_stats = {}
    
    def record_train(self, feature_name, available):
        if feature_name not in self.train_stats:
            self.train_stats[feature_name] = {'available': 0, 'total': 0}
        self.train_stats[feature_name]['total'] += 1
        if available:
            self.train_stats[feature_name]['available'] += 1
    
    def check_consistency(self):
        """
        检查训练和推理时的特征可用性一致性
        """
        inconsistencies = []
        for feature_name in self.train_stats:
            train_rate = self.train_stats[feature_name]['available'] / self.train_stats[feature_name]['total']
            serve_rate = self.serve_stats.get(feature_name, {}).get('available', 0) / max(self.serve_stats.get(feature_name, {}).get('total', 1), 1)
            
            if abs(train_rate - serve_rate) > 0.1:
                inconsistencies.append({
                    'feature': feature_name,
                    'train_rate': train_rate,
                    'serve_rate': serve_rate
                })
        
        return inconsistencies
```

### 样本穿越修复

```
修复方案：

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

4. 位置特征处理
   ├── 训练时：分数 = 主分数 + 位置偏差
   ├── 推理时：分数 = 主分数（位置偏差置0）
   └── 或使用无偏数据集训练
```

---

## 13.7 样本质量控制

### 样本质量指标

| 指标 | 定义 | 阈值 |
|-----|------|------|
| **标签准确率** | 标签与真实行为一致性 | >99% |
| **特征完整率** | 特征非空比例 | >95% |
| **样本新鲜度** | 样本距当前时间 | <7天 |
| **样本多样性** | 覆盖的商品/用户比例 | >80% |

### 样本清洗流程

```
Step 1: 异常样本过滤
├── 机器人流量（UA异常、IP集中）
├── 刷单流量（行为模式异常）
├── 误点击（停留<800ms）
└── 异常高频行为

Step 2: 特征缺失处理
├── 缺失率>50%的特征：删除
├── 缺失率<5%的特征：填充默认值
└── 缺失率5-50%的特征：缺失指示器+填充

Step 3: 标签噪声处理
├── 一致性检验（多次行为是否一致）
├── 置信度加权（低置信样本降权）
└── 噪声标签学习（loss修正）

Step 4: 样本去重
├── 同用户同商品同session去重
├── 保留最后一次行为
└── 或合并多次行为
```

### 样本监控看板

```
实时监控指标：
├── 样本量：每小时新增样本数
├── 正负比例：实时正负样本比
├── 特征分布：关键特征的分布
├── 标签分布：正样本占比
└── 异常率：被过滤的样本比例

告警规则：
├── 样本量下降>30%：黄色告警
├── 正负比例突变>2倍：黄色告警
├── 特征分布PSI>0.1：黄色告警
├── 样本量下降>50%：红色告警
└── 正负比例突变>5倍：红色告警
```

---

## 13.8 样本采样策略

### 时间采样

```
策略1：滑动窗口
├── 使用最近N天的数据
├── 每天滑动更新
└── 窗口大小：7-30天

策略2：指数衰减
├── 近期样本权重高
├── 远期样本权重低
└── 权重 = exp(-λ * days_ago)

策略3：分层时间采样
├── 近7天：100%采样
├── 7-14天：50%采样
├── 14-30天：20%采样
└── >30天：5%采样
```

### 用户采样

```python
# 活跃度分层采样
def user_stratified_sampling(user_activity, sample_rates):
    """
    根据用户活跃度分层采样
    
    user_activity: 用户活跃度等级
    sample_rates: 各层采样率
    """
    # 高活跃用户：采样率低（样本充足）
    # 低活跃用户：采样率高（样本稀疏）
    
    if user_activity == 'high':
        return random.random() < sample_rates['high']  # 如0.3
    elif user_activity == 'medium':
        return random.random() < sample_rates['medium']  # 如0.5
    else:  # low activity
        return random.random() < sample_rates['low']  # 如0.8

# 价值分层采样
def value_stratified_sampling(user_value):
    """
    根据用户价值分层采样
    高价值用户样本更重要
    """
    if user_value == 'high':
        return True  # 高价值用户全量保留
    elif user_value == 'medium':
        return random.random() < 0.7
    else:
        return random.random() < 0.3
```

### 商品采样

```python
# 冷启动商品过采样
def cold_start_oversampling(item_id, item_age_days, oversample_factor=3):
    """
    对冷启动商品过采样
    """
    if item_age_days < 7:
        # 新商品：复制多份
        return oversample_factor
    elif item_age_days < 30:
        return 2
    else:
        return 1

# 长尾商品过采样
def long_tail_oversampling(item_id, item_exposure_count):
    """
    对长尾商品过采样
    """
    if item_exposure_count < 100:
        return 5  # 极冷门
    elif item_exposure_count < 1000:
        return 3  # 冷门
    elif item_exposure_count < 10000:
        return 1.5  # 中等
    else:
        return 1  # 热门
```

---

## 13.9 样本分析工具

### 样本统计分析

```python
# 样本分布分析
def analyze_sample_distribution(samples):
    """
    分析样本分布
    """
    analysis = {}
    
    # 1. 时间分布
    analysis['time_dist'] = {
        'hour': samples.groupby('hour').size(),
        'day_of_week': samples.groupby('day_of_week').size(),
    }
    
    # 2. 用户分布
    analysis['user_dist'] = {
        'activity_level': samples.groupby('user_activity').size(),
        'value_level': samples.groupby('user_value').size(),
    }
    
    # 3. 商品分布
    analysis['item_dist'] = {
        'category': samples.groupby('item_category').size(),
        'popularity': samples.groupby('item_popularity_bucket').size(),
    }
    
    # 4. 标签分布
    analysis['label_dist'] = {
        'positive_rate': samples['label'].mean(),
        'by_user_group': samples.groupby('user_group')['label'].mean(),
        'by_item_group': samples.groupby('item_group')['label'].mean(),
    }
    
    return analysis

# 特征重要性分析
def analyze_feature_importance(samples, model):
    """
    分析特征对预测的贡献
    """
    # Permutation Importance
    baseline_score = model.evaluate(samples)
    
    importance = {}
    for feature in samples.columns:
        if feature == 'label':
            continue
        
        # 打乱特征
        shuffled = samples.copy()
        shuffled[feature] = np.random.permutation(shuffled[feature])
        
        # 计算分数下降
        shuffled_score = model.evaluate(shuffled)
        importance[feature] = baseline_score - shuffled_score
    
    return importance
```

### 样本偏差检测

```python
# 分布漂移检测
def detect_distribution_shift(train_samples, online_samples, features):
    """
    检测训练样本与线上样本的分布漂移
    """
    shift_report = {}
    
    for feature in features:
        # 计算PSI
        psi = calculate_psi(
            train_samples[feature],
            online_samples[feature]
        )
        
        shift_report[feature] = {
            'psi': psi,
            'status': 'OK' if psi < 0.1 else ('WARNING' if psi < 0.25 else 'ALERT')
        }
    
    return shift_report

def calculate_psi(expected, actual, buckets=10):
    """
    计算PSI（Population Stability Index）
    """
    def scale_range(series, buckets):
        return pd.qcut(series, buckets, labels=False, duplicates='drop')
    
    expected_percents = scale_range(expected, buckets).value_counts(normalize=True).sort_index()
    actual_percents = scale_range(actual, buckets).value_counts(normalize=True).sort_index()
    
    psi = 0
    for i in expected_percents.index:
        e = expected_percents.get(i, 0.0001)
        a = actual_percents.get(i, 0.0001)
        psi += (a - e) * np.log(a / e)
    
    return psi
```

---

## 13.10 样本工程最佳实践

### 样本构造Checklist

```
□ 正样本定义是否与业务目标对齐？
□ 负样本构造是否与线上分布一致？
□ 样本权重是否合理？
□ 是否存在样本穿越？
□ 正负样本比例是否合理？
□ 是否处理了样本不平衡？
□ 是否纠正了已知偏差？
□ 样本质量是否达标？
□ 样本分布是否稳定？
□ 是否有样本监控告警？
```

### 样本迭代流程

```
1. 样本设计
   ├── 明确业务目标
   ├── 定义正负样本
   └── 设计采样策略

2. 样本构造
   ├── 数据采集
   ├── 特征提取
   └── 标签生成

3. 样本清洗
   ├── 异常过滤
   ├── 缺失处理
   └── 去重

4. 样本分析
   ├── 分布分析
   ├── 偏差检测
   └── 质量评估

5. 样本验证
   ├── 离线实验
   ├── A/B测试
   └── 效果评估

6. 样本监控
   ├── 实时监控
   ├── 告警机制
   └── 定期复盘
```

### 常见问题与解决方案

| 问题 | 症状 | 解决方案 |
|-----|------|---------|
| 样本不平衡 | 模型偏向多数类 | 负采样、过采样、Focal Loss |
| 样本偏差 | 离线涨在线跌 | 偏差纠正、分布对齐 |
| 样本穿越 | 特征不可用 | 特征快照、时间对齐 |
| 样本噪声 | 标签不准确 | 噪声学习、置信度加权 |
| 样本老化 | 效果下降 | 滑动窗口、增量更新 |
| 样本稀疏 | 冷启动差 | 过采样、迁移学习 |

---

[← 上一章：快速参考卡片](12-quick-reference.md) | [返回目录](../README.md)
