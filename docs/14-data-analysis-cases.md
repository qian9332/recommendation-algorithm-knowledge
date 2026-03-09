# 十四、数据分析实战案例：从分析到指标提升

> 本章通过真实案例，展示如何通过数据分析发现问题、定位原因、优化模型，最终实现业务指标提升。

---

## 14.1 案例一：召回率提升分析

### 14.1.1 问题发现

```
背景：某电商平台首页推荐，日活1.2亿

监控发现：
├── 整体CTR：2.8%（稳定）
├── 整体GMV：日均2.1亿（稳定）
├── 召回覆盖率：68%（下降3%）
└── 长尾商品曝光占比：12%（下降5%）

问题定位：
通过分层数据分析发现：
├── 头部商品（Top 10%）曝光占比：78%（上升4%）
├── 腰部商品曝光占比：18%（下降2%）
└── 长尾商品曝光占比：4%（下降2%）
```

### 14.1.2 数据分析过程

```python
# Step 1: 召回通道分析
def analyze_recall_channels(log_data):
    """
    分析各召回通道的覆盖情况
    """
    channels = ['item_cf', 'vector', 'graph', 'hot', 'new']
    
    results = {}
    for channel in channels:
        channel_data = log_data[log_data['recall_channel'] == channel]
        
        results[channel] = {
            'recall_count': len(channel_data),
            'unique_items': channel_data['item_id'].nunique(),
            'long_tail_ratio': (channel_data['item_exposure'] < 100).mean(),
            'ctr': channel_data['is_click'].mean(),
            'coverage': channel_data['item_id'].nunique() / total_items
        }
    
    return results

# 分析结果
"""
通道分析结果：
┌─────────────┬──────────┬──────────┬────────────┬────────┐
│ 通道        │ 召回量   │ 覆盖率   │ 长尾占比   │ CTR    │
├─────────────┼──────────┼──────────┼────────────┼────────┤
│ item_cf     │ 45%      │ 35%      │ 8%         │ 3.2%   │
│ vector      │ 25%      │ 52%      │ 25%        │ 2.5%   │
│ graph       │ 15%      │ 28%      │ 18%        │ 2.8%   │
│ hot         │ 10%      │ 5%       │ 2%         │ 4.1%   │
│ new         │ 5%       │ 12%      │ 45%        │ 1.8%   │
└─────────────┴──────────┴──────────┴────────────┴────────┘

发现问题：
1. Item-CF通道召回量占比过高（45%），但长尾覆盖率低（8%）
2. Vector通道长尾覆盖好，但召回量占比下降（从30%降到25%）
3. New通道长尾占比高，但召回量过低（5%）
"""
```

### 14.1.3 问题定位

```python
# Step 2: 深入分析Item-CF通道
def analyze_item_cf_drift(log_data):
    """
    分析Item-CF通道的分布漂移
    """
    # 按时间分段
    log_data['date'] = pd.to_datetime(log_data['date'])
    
    # 计算各时段的分布
    daily_stats = log_data.groupby('date').agg({
        'item_id': 'nunique',
        'user_id': 'nunique',
        'is_click': 'mean'
    }).rename(columns={
        'item_id': 'unique_items',
        'user_id': 'unique_users',
        'is_click': 'ctr'
    })
    
    # 计算PSI
    baseline = daily_stats.iloc[:7]  # 前7天作为基线
    current = daily_stats.iloc[-7:]  # 最近7天
    
    psi = calculate_psi(baseline['unique_items'], current['unique_items'])
    
    return {
        'daily_stats': daily_stats,
        'psi': psi,
        'drift_detected': psi > 0.1
    }

# 分析结果
"""
Item-CF通道问题定位：
1. 热门商品共现对数增长35%
   └── 导致热门商品被过度召回
   
2. 长尾商品共现对数下降20%
   └── 原因：新商品上架速度快，共现数据积累不足
   
3. PSI = 0.18（>0.1，存在显著漂移）
   └── 分布已发生显著变化
"""
```

### 14.1.4 优化方案

```python
# Step 3: 制定优化方案
"""
优化方案：

方案1：调整召回通道配额
├── Item-CF：45% → 35%
├── Vector：25% → 30%
├── Graph：15% → 18%
├── Hot：10% → 8%
└── New：5% → 9%

方案2：Item-CF通道内部优化
├── 热门商品降权：score *= 1 / log(1 + popularity)
├── 长尾商品加权：score *= (1 + 0.1 * log(1 + 1/popularity))
└── 新商品保量：强制保留10%新商品

方案3：Vector通道增强
├── 新商品向量实时更新（从天级改为小时级）
├── 增加难负例挖掘
└── 温度系数动态调整
"""

# 实施代码
def optimize_recall_quota(item_cf_score, item_popularity, is_new):
    """
    优化Item-CF召回分数
    """
    # 热门降权
    popularity_penalty = 1.0 / np.log2(1 + item_popularity)
    
    # 长尾加权
    long_tail_boost = 1.0 + 0.1 * np.log2(1 + 1.0 / (item_popularity + 1))
    
    # 新品保量
    new_item_boost = 1.5 if is_new else 1.0
    
    optimized_score = item_cf_score * popularity_penalty * long_tail_boost * new_item_boost
    
    return optimized_score
```

### 14.1.5 效果验证

```
A/B实验设计：
├── 实验组：优化后的召回策略
├── 对照组：原召回策略
├── 流量分配：各5%
└── 实验周期：7天

实验结果：
┌─────────────────┬──────────┬──────────┬──────────┐
│ 指标            │ 对照组   │ 实验组   │ 提升     │
├─────────────────┼──────────┼──────────┼──────────┤
│ 召回覆盖率      │ 68%      │ 74%      │ +6%      │
│ 长尾商品曝光占比│ 12%      │ 18%      │ +6%      │
│ 整体CTR         │ 2.8%     │ 2.85%    │ +0.05%   │
│ 整体GMV         │ 2.1亿    │ 2.18亿   │ +3.8%    │
│ 长尾GMV         │ 0.25亿   │ 0.32亿   │ +28%     │
└─────────────────┴──────────┴──────────┴──────────┘

显著性检验：
├── CTR提升：p-value = 0.12（不显著）
├── GMV提升：p-value = 0.03（显著）
└── 长尾GMV提升：p-value = 0.001（高度显著）

结论：
优化方案在保持整体CTR稳定的前提下，显著提升了长尾商品的曝光和GMV，
整体GMV提升3.8%，达到预期目标。
"""
```

---

## 14.2 案例二：CTR预估模型优化

### 14.2.1 问题发现

```
背景：某短视频平台推荐，日活8000万

监控发现：
├── 离线AUC：0.735（稳定）
├── 在线CTR：3.2%（下降0.3%）
├── 在线GMV：日均0.8亿（下降2%）
└── 用户停留时长：人均15分钟（下降5%）

问题：
离线AUC稳定，但在线指标下降，典型的"离线在线不一致"问题
```

### 14.2.2 数据分析过程

```python
# Step 1: 离线在线一致性分析
def analyze_offline_online_gap(train_data, online_data):
    """
    分析离线训练数据与线上数据的分布差异
    """
    # 特征分布对比
    features = ['user_activity', 'item_popularity', 'time_hour', 'category']
    
    gap_report = {}
    for feature in features:
        train_dist = train_data[feature].value_counts(normalize=True)
        online_dist = online_data[feature].value_counts(normalize=True)
        
        # 计算KL散度
        kl_div = entropy(train_dist, online_dist)
        
        gap_report[feature] = {
            'kl_divergence': kl_div,
            'significant': kl_div > 0.1
        }
    
    return gap_report

# 分析结果
"""
特征分布差异分析：
┌─────────────────┬──────────────┬────────────┐
│ 特征            │ KL散度       │ 是否显著   │
├─────────────────┼──────────────┼────────────┤
│ user_activity   │ 0.05         │ 否         │
│ item_popularity │ 0.18         │ 是         │
│ time_hour       │ 0.03         │ 否         │
│ category        │ 0.12         │ 是         │
└─────────────────┴──────────────┴────────────┘

发现问题：
1. item_popularity分布差异显著（KL=0.18）
   └── 训练数据中热门商品占比更高
   
2. category分布差异显著（KL=0.12）
   └── 训练数据中某些类目占比偏高
"""
```

```python
# Step 2: 样本偏差分析
def analyze_sample_bias(log_data):
    """
    分析样本选择偏差
    """
    # 按曝光位置分组
    position_groups = log_data.groupby('position').agg({
        'is_click': 'mean',
        'is_convert': 'mean',
        'item_id': 'count'
    }).rename(columns={
        'is_click': 'ctr',
        'is_convert': 'cvr',
        'item_id': 'count'
    })
    
    # 计算位置偏差
    position_groups['ctr_ratio'] = position_groups['ctr'] / position_groups['ctr'].mean()
    
    return position_groups

# 分析结果
"""
位置偏差分析：
┌──────────┬────────┬────────┬──────────────┐
│ 位置     │ CTR    │ 样本量 │ CTR比值      │
├──────────┼────────┼────────┼──────────────┤
│ 1-3      │ 5.2%   │ 35%    │ 1.63         │
│ 4-6      │ 3.8%   │ 25%    │ 1.19         │
│ 7-10     │ 2.9%   │ 20%    │ 0.91         │
│ 11-20    │ 2.1%   │ 15%    │ 0.66         │
│ 21+      │ 1.2%   │ 5%     │ 0.38         │
└──────────┴────────┴────────┴──────────────┘

发现问题：
1. 前3位CTR是平均值的1.63倍
   └── 存在显著的位置偏差
   
2. 训练样本中前3位占比35%
   └── 模型过度学习位置信号
   
3. 线上推理时位置未知
   └── 导致模型预测偏差
"""
```

### 14.2.3 问题定位

```python
# Step 3: 特征重要性分析
def analyze_feature_importance(model, test_data):
    """
    分析特征重要性，定位问题特征
    """
    # Permutation Importance
    baseline_score = model.evaluate(test_data)
    
    importance = {}
    for feature in test_data.columns:
        if feature in ['label', 'position']:
            continue
        
        # 打乱特征
        shuffled = test_data.copy()
        shuffled[feature] = np.random.permutation(shuffled[feature])
        
        # 计算分数下降
        shuffled_score = model.evaluate(shuffled)
        importance[feature] = baseline_score - shuffled_score
    
    return importance

# 分析结果
"""
特征重要性分析：
┌─────────────────────┬────────────────┬────────────────┐
│ 特征                │ 重要性得分     │ 排名           │
├─────────────────────┼────────────────┼────────────────┤
│ position            │ 0.045          │ 1              │
│ user_history        │ 0.032          │ 2              │
│ item_embedding      │ 0.028          │ 3              │
│ category            │ 0.015          │ 4              │
│ user_activity       │ 0.012          │ 5              │
└─────────────────────┴────────────────┴────────────────┘

关键发现：
1. position特征重要性排名第一（0.045）
   └── 模型过度依赖位置信息
   
2. 线上推理时position特征不可用
   └── 导致模型预测失准
   
3. 这解释了离线AUC高但在线CTR低的原因
"""
```

### 14.2.4 优化方案

```python
# Step 4: 制定优化方案
"""
优化方案：

方案1：位置特征处理
├── 训练时：score = main_score + position_bias
├── 推理时：score = main_score（position_bias置0）
└── 位置偏差作为独立任务建模

方案2：样本重采样
├── 按位置分层采样，保证各位置样本比例一致
├── 对低位置样本过采样
└── 对高位置样本降采样

方案3：IPS加权
├── 计算各位置的倾向得分
├── 用IPS权重校正样本
└── 公式：weight = 1 / P(click|position)
"""

# 实施代码：位置偏差建模
class PositionBiasModel(nn.Module):
    def __init__(self, num_features, num_positions):
        super().__init__()
        # 主模型
        self.main_tower = nn.Sequential(
            nn.Linear(num_features, 256),
            nn.ReLU(),
            nn.Linear(256, 128),
            nn.ReLU(),
            nn.Linear(128, 1)
        )
        
        # 位置偏差塔
        self.position_tower = nn.Sequential(
            nn.Embedding(num_positions, 16),
            nn.Linear(16, 1)
        )
    
    def forward(self, features, position):
        main_score = self.main_tower(features)
        position_bias = self.position_tower(position)
        
        if self.training:
            # 训练时：分数 = 主分数 + 位置偏差
            return main_score + position_bias
        else:
            # 推理时：只返回主分数
            return main_score
    
    def get_position_bias(self, position):
        """获取位置偏差值，用于分析"""
        return self.position_tower(position)
```

### 14.2.5 效果验证

```
A/B实验设计：
├── 实验组：位置偏差建模 + IPS加权
├── 对照组：原模型
├── 流量分配：各10%
└── 实验周期：14天

实验结果：
┌─────────────────┬──────────┬──────────┬──────────┐
│ 指标            │ 对照组   │ 实验组   │ 提升     │
├─────────────────┼──────────┼──────────┼──────────┤
│ 离线AUC         │ 0.735    │ 0.732    │ -0.003   │
│ 在线CTR         │ 3.2%     │ 3.45%    │ +7.8%    │
│ 在线GMV         │ 0.8亿    │ 0.87亿   │ +8.8%    │
│ 用户停留时长    │ 15分钟   │ 16.2分钟 │ +8%      │
│ 长尾商品CTR     │ 1.8%     │ 2.1%     │ +16.7%   │
└─────────────────┴──────────┴──────────┴──────────┘

关键发现：
1. 离线AUC略有下降（-0.003）
   └── 因为模型不再依赖位置信息
   
2. 在线CTR显著提升（+7.8%）
   └── 位置偏差被有效消除
   
3. 长尾商品CTR提升更明显（+16.7%）
   └── 模型对低位置商品预测更准确

结论：
通过位置偏差建模和IPS加权，成功解决了离线在线不一致问题，
在线CTR提升7.8%，GMV提升8.8%。
"""
```

---

## 14.3 案例三：多任务学习跷跷板问题

### 14.3.1 问题发现

```
背景：某电商平台精排模型，同时优化CTR和CVR

监控发现：
├── CTR任务：AUC 0.73，线上CTR 3.5%
├── CVR任务：AUC 0.68，线上CVR 2.1%
├── 尝试优化CVR后：
│   ├── CVR AUC提升：0.68 → 0.71
│   ├── CTR AUC下降：0.73 → 0.70
│   └── 线上CTR下降：3.5% → 3.2%
└── 问题：典型的跷跷板现象
```

### 14.3.2 数据分析过程

```python
# Step 1: 梯度冲突分析
def analyze_gradient_conflict(model, train_data):
    """
    分析多任务梯度冲突
    """
    # 计算各任务梯度
    grad_ctr = compute_gradient(model, train_data, task='ctr')
    grad_cvr = compute_gradient(model, train_data, task='cvr')
    
    # 计算梯度余弦相似度
    cosine_sim = np.dot(grad_ctr, grad_cvr) / (np.linalg.norm(grad_ctr) * np.linalg.norm(grad_cvr))
    
    # 计算梯度冲突角度
    conflict_angle = np.arccos(cosine_sim) * 180 / np.pi
    
    return {
        'cosine_similarity': cosine_sim,
        'conflict_angle': conflict_angle,
        'has_conflict': cosine_sim < 0
    }

# 分析结果
"""
梯度冲突分析：
┌─────────────────────┬────────────────┐
│ 指标                │ 值             │
├─────────────────────┼────────────────┤
│ 梯度余弦相似度      │ -0.35          │
│ 冲突角度            │ 110°           │
│ 是否存在冲突        │ 是             │
└─────────────────────┴────────────────┘

发现问题：
1. 梯度余弦相似度为负（-0.35）
   └── 两个任务的梯度方向相反
   
2. 冲突角度110°
   └── 存在严重的梯度冲突
   
3. 这解释了为什么优化CVR会导致CTR下降
"""
```

```python
# Step 2: 样本重叠分析
def analyze_sample_overlap(train_data):
    """
    分析CTR和CVR任务的样本重叠情况
    """
    # CTR正样本：点击
    ctr_positive = train_data[train_data['is_click'] == 1]
    
    # CVR正样本：转化
    cvr_positive = train_data[train_data['is_convert'] == 1]
    
    # 计算重叠
    overlap = set(ctr_positive['sample_id']) & set(cvr_positive['sample_id'])
    
    return {
        'ctr_positive_count': len(ctr_positive),
        'cvr_positive_count': len(cvr_positive),
        'overlap_count': len(overlap),
        'overlap_ratio': len(overlap) / len(cvr_positive)
    }

# 分析结果
"""
样本重叠分析：
┌─────────────────────┬────────────────┐
│ 指标                │ 值             │
├─────────────────────┼────────────────┤
│ CTR正样本数         │ 1,200,000      │
│ CVR正样本数         │ 25,000         │
│ 重叠样本数          │ 25,000         │
│ 重叠比例            │ 100%           │
└─────────────────────┴────────────────┘

关键发现：
1. CVR正样本完全包含在CTR正样本中
   └── 这是ESMM论文指出的样本选择偏差问题
   
2. CVR样本量远小于CTR（25,000 vs 1,200,000）
   └── CVR任务更容易过拟合
   
3. 两个任务共享底层特征
   └── 底层特征更新方向由CTR主导
"""
```

### 14.3.3 问题定位

```python
# Step 3: 任务权重敏感性分析
def analyze_task_weight_sensitivity(model, train_data, weight_range):
    """
    分析任务权重对效果的影响
    """
    results = []
    
    for w_ctr in weight_range:
        w_cvr = 1 - w_ctr
        
        # 训练模型
        trained_model = train_with_weights(model, train_data, w_ctr, w_cvr)
        
        # 评估
        ctr_auc = evaluate(trained_model, test_data, task='ctr')
        cvr_auc = evaluate(trained_model, test_data, task='cvr')
        
        results.append({
            'w_ctr': w_ctr,
            'w_cvr': w_cvr,
            'ctr_auc': ctr_auc,
            'cvr_auc': cvr_auc
        })
    
    return pd.DataFrame(results)

# 分析结果
"""
任务权重敏感性分析：
┌────────┬────────┬─────────┬─────────┐
│ w_ctr  │ w_cvr  │ CTR AUC │ CVR AUC │
├────────┼────────┼─────────┼─────────┤
│ 0.9    │ 0.1    │ 0.735   │ 0.665   │
│ 0.8    │ 0.2    │ 0.730   │ 0.675   │
│ 0.7    │ 0.3    │ 0.720   │ 0.685   │
│ 0.6    │ 0.4    │ 0.710   │ 0.695   │
│ 0.5    │ 0.5    │ 0.700   │ 0.705   │
│ 0.4    │ 0.6    │ 0.685   │ 0.710   │
└────────┴────────┴─────────┴─────────┘

关键发现：
1. CTR权重高时，CTR AUC高但CVR AUC低
2. CVR权重高时，CVR AUC高但CTR AUC低
3. 无法通过简单调整权重同时提升两个任务
4. 需要更高级的多任务学习架构
"""
```

### 14.3.4 优化方案

```python
# Step 4: 制定优化方案
"""
优化方案：

方案1：PLE架构
├── 共享Expert + 任务独占Expert
├── 每层都有共享和独占部分
├── 隔离度>60%，有效缓解梯度冲突

方案2：PCGrad
├── 检测梯度冲突（余弦相似度<0）
├── 将冲突梯度投影到非冲突方向
└── 保证两个任务的梯度不互相干扰

方案3：不确定性加权
├── 自动学习任务权重
├── 权重与任务不确定性相关
└── 不确定性高的任务权重低
"""

# 实施代码：PLE架构
class PLEModel(nn.Module):
    def __init__(self, num_features, num_experts=8, num_tasks=2):
        super().__init__()
        
        # 共享Expert
        self.shared_experts = nn.ModuleList([
            nn.Sequential(
                nn.Linear(num_features, 128),
                nn.ReLU(),
                nn.Linear(128, 64)
            ) for _ in range(num_experts // 2)
        ])
        
        # 任务独占Expert
        self.task_experts = nn.ModuleList([
            nn.ModuleList([
                nn.Sequential(
                    nn.Linear(num_features, 128),
                    nn.ReLU(),
                    nn.Linear(128, 64)
                ) for _ in range(num_experts // 2)
            ]) for _ in range(num_tasks)
        ])
        
        # 门控网络
        self.gates = nn.ModuleList([
            nn.Sequential(
                nn.Linear(num_features, num_experts),
                nn.Softmax(dim=-1)
            ) for _ in range(num_tasks)
        ])
        
        # 任务塔
        self.towers = nn.ModuleList([
            nn.Sequential(
                nn.Linear(64, 32),
                nn.ReLU(),
                nn.Linear(32, 1)
            ) for _ in range(num_tasks)
        ])
    
    def forward(self, x):
        # 共享Expert输出
        shared_outputs = [expert(x) for expert in self.shared_experts]
        
        # 各任务输出
        task_outputs = []
        for task_id in range(2):
            # 任务独占Expert输出
            task_expert_outputs = [expert(x) for expert in self.task_experts[task_id]]
            
            # 合并所有Expert
            all_expert_outputs = shared_outputs + task_expert_outputs
            all_expert_outputs = torch.stack(all_expert_outputs, dim=1)
            
            # 门控加权
            gate_weights = self.gates[task_id](x)
            weighted_output = torch.sum(all_expert_outputs * gate_weights.unsqueeze(-1), dim=1)
            
            # 任务塔
            task_output = self.towers[task_id](weighted_output)
            task_outputs.append(task_output)
        
        return task_outputs
```

### 14.3.5 效果验证

```
A/B实验设计：
├── 实验组：PLE架构 + PCGrad
├── 对照组：MMoE架构
├── 流量分配：各10%
└── 实验周期：14天

实验结果：
┌─────────────────┬──────────┬──────────┬──────────┐
│ 指标            │ MMoE     │ PLE      │ 提升     │
├─────────────────┼──────────┼──────────┼──────────┤
│ CTR AUC         │ 0.730    │ 0.742    │ +1.2%    │
│ CVR AUC         │ 0.685    │ 0.708    │ +2.3%    │
│ 梯度冲突角度    │ 110°     │ 25°      │ -85°     │
│ 在线CTR         │ 3.2%     │ 3.35%    │ +4.7%    │
│ 在线CVR         │ 2.0%     │ 2.18%    │ +9.0%    │
│ GMV             │ 0.8亿    │ 0.89亿   │ +11.3%   │
└─────────────────┴──────────┴──────────┴──────────┘

关键发现：
1. 梯度冲突角度从110°降到25°
   └── PLE有效缓解了梯度冲突
   
2. CTR和CVR同时提升
   └── 解决了跷跷板问题
   
3. GMV提升11.3%
   └── 业务收益显著

结论：
通过PLE架构和PCGrad，成功解决了多任务学习的跷跷板问题，
CTR提升4.7%，CVR提升9.0%，GMV提升11.3%。
"""
```

---

## 14.4 案例四：冷启动优化

### 14.4.1 问题发现

```
背景：某电商平台，日均新商品上架50万件

监控发现：
├── 新商品首日曝光率：12%（目标>30%）
├── 新商品首日CTR：1.2%（老商品3.5%）
├── 新商品首周存活率：35%（目标>60%）
└── 问题：新商品冷启动效果差
```

### 14.4.2 数据分析过程

```python
# Step 1: 新商品曝光分析
def analyze_new_item_exposure(log_data):
    """
    分析新商品曝光情况
    """
    # 定义新商品：上架时间<24小时
    new_items = log_data[log_data['item_age_hours'] < 24]
    old_items = log_data[log_data['item_age_hours'] >= 24]
    
    # 计算各召回通道对新商品的覆盖
    channels = ['item_cf', 'vector', 'graph', 'hot', 'new']
    
    results = {}
    for channel in channels:
        channel_data = log_data[log_data['recall_channel'] == channel]
        new_in_channel = channel_data[channel_data['item_age_hours'] < 24]
        
        results[channel] = {
            'total_recall': len(channel_data),
            'new_item_recall': len(new_in_channel),
            'new_item_ratio': len(new_in_channel) / len(channel_data),
            'new_item_ctr': new_in_channel['is_click'].mean() if len(new_in_channel) > 0 else 0
        }
    
    return results

# 分析结果
"""
新商品召回分析：
┌─────────────┬────────────┬────────────────┬────────────┬────────────┐
│ 通道        │ 总召回量   │ 新商品召回量   │ 新商品占比 │ 新商品CTR  │
├─────────────┼────────────┼────────────────┼────────────┼────────────┤
│ item_cf     │ 45%        │ 0.5%           │ 1.1%       │ 0.8%       │
│ vector      │ 25%        │ 8%             │ 32%        │ 1.5%       │
│ graph       │ 15%        │ 5%             │ 33%        │ 1.3%       │
│ hot         │ 10%        │ 0%             │ 0%         │ -          │
│ new         │ 5%         │ 86%            │ 1720%      │ 1.2%       │
└─────────────┴────────────┴────────────────┴────────────┴────────────┘

发现问题：
1. Item-CF对新商品几乎无召回（1.1%）
   └── 新商品无共现数据
   
2. Vector通道新商品占比高（32%）
   └── 但整体召回量低（25%）
   
3. New通道专门针对新商品
   └── 但召回量过低（5%）
   
4. 新商品CTR普遍低于老商品
   └── 需要优化排序模型对新商品的处理
"""
```

```python
# Step 2: 新商品特征分析
def analyze_new_item_features(new_items, old_items):
    """
    分析新商品与老商品的特征差异
    """
    features = ['has_image', 'has_title', 'category_depth', 'price_range']
    
    results = {}
    for feature in features:
        new_dist = new_items[feature].value_counts(normalize=True)
        old_dist = old_items[feature].value_counts(normalize=True)
        
        # 计算分布差异
        kl_div = entropy(new_dist, old_dist)
        
        results[feature] = {
            'new_distribution': new_dist.to_dict(),
            'old_distribution': old_dist.to_dict(),
            'kl_divergence': kl_div
        }
    
    return results

# 分析结果
"""
新商品特征分析：
┌─────────────────┬──────────────┬──────────────┬────────────┐
│ 特征            │ 新商品分布   │ 老商品分布   │ KL散度     │
├─────────────────┼──────────────┼──────────────┼────────────┤
│ has_image       │ 85%有图      │ 95%有图      │ 0.08       │
│ has_title       │ 92%有标题    │ 98%有标题    │ 0.05       │
│ category_depth  │ 平均3.2层    │ 平均2.8层    │ 0.12       │
│ price_range     │ 中低价为主   │ 全价位分布   │ 0.18       │
└─────────────────┴──────────────┴──────────────┴────────────┘

关键发现：
1. 新商品图片完整率低于老商品（85% vs 95%）
   └── 影响视觉特征质量
   
2. 新商品类目深度更深（3.2 vs 2.8）
   └── 可能是更细分的商品
   
3. 新商品价格分布与老商品差异大（KL=0.18）
   └── 需要考虑价格因素
"""
```

### 14.4.3 优化方案

```python
# Step 3: 制定优化方案
"""
优化方案：

方案1：召回层优化
├── 新商品向量实时更新（小时级）
├── 增加New通道配额：5% → 15%
├── 新商品保量机制：强制曝光配额
└── 基于内容的相似召回

方案2：排序层优化
├── 新商品特征补全
├── 新商品置信度加权
├── 探索-利用平衡（UCB/Thompson Sampling）
└── 元学习快速适应

方案3：特征工程优化
├── 零样本向量构建
├── 图文多模态融合
├── 知识图谱增强
└── 新商品Embedding初始化
"""

# 实施代码：新商品保量机制
class NewItemExploration:
    def __init__(self, min_exposure=100, exploration_ratio=0.15):
        self.min_exposure = min_exposure
        self.exploration_ratio = exploration_ratio
        self.item_exposure_count = {}
    
    def get_exploration_score(self, item_id, item_age_hours, base_score):
        """
        计算新商品的探索分数
        """
        # 新商品定义
        is_new = item_age_hours < 24
        
        if not is_new:
            return base_score
        
        # 获取当前曝光量
        current_exposure = self.item_exposure_count.get(item_id, 0)
        
        # 未达到最低曝光量，强制提升
        if current_exposure < self.min_exposure:
            # UCB公式
            exploration_bonus = np.sqrt(2 * np.log(self.total_requests + 1) / (current_exposure + 1))
            return base_score + exploration_bonus
        
        # 已达到最低曝光量，使用Thompson Sampling
        else:
            # 假设点击率服从Beta分布
            alpha = self.item_clicks.get(item_id, 1) + 1
            beta = current_exposure - self.item_clicks.get(item_id, 0) + 1
            
            # 从后验分布采样
            sampled_ctr = np.random.beta(alpha, beta)
            return base_score * (1 + sampled_ctr)
    
    def update(self, item_id, is_click):
        """
        更新商品曝光和点击统计
        """
        self.item_exposure_count[item_id] = self.item_exposure_count.get(item_id, 0) + 1
        if is_click:
            self.item_clicks[item_id] = self.item_clicks.get(item_id, 0) + 1
```

### 14.4.4 效果验证

```
A/B实验设计：
├── 实验组：新商品保量 + UCB探索 + 零样本向量
├── 对照组：原策略
├── 流量分配：各10%
└── 实验周期：14天

实验结果：
┌─────────────────────┬──────────┬──────────┬──────────┐
│ 指标                │ 对照组   │ 实验组   │ 提升     │
├─────────────────────┼──────────┼──────────┼──────────┤
│ 新商品首日曝光率    │ 12%      │ 35%      │ +23%     │
│ 新商品首日CTR       │ 1.2%     │ 1.8%     │ +50%     │
│ 新商品首周存活率    │ 35%      │ 58%      │ +23%     │
│ 新商品首周GMV       │ 0.05亿   │ 0.09亿   │ +80%     │
│ 整体GMV             │ 0.8亿    │ 0.85亿   │ +6.3%    │
│ 老商品CTR           │ 3.5%     │ 3.45%    │ -1.4%    │
└─────────────────────┴──────────┴──────────┴──────────┘

关键发现：
1. 新商品首日曝光率大幅提升（+23%）
   └── 保量机制有效
   
2. 新商品CTR提升50%
   └── 零样本向量质量好
   
3. 老商品CTR略有下降（-1.4%）
   └── 可接受的代价
   
4. 整体GMV提升6.3%
   └── 新商品贡献显著

结论：
通过新商品保量机制和零样本向量，成功解决了冷启动问题，
新商品首日曝光率提升23%，首周存活率提升23%，整体GMV提升6.3%。
"""
```

---

## 14.5 数据分析方法论总结

### 14.5.1 数据分析流程

```
标准数据分析流程：

Step 1: 问题定义
├── 明确业务目标
├── 定义核心指标
└── 设定预期效果

Step 2: 数据收集
├── 确定数据源
├── 数据清洗
└── 特征提取

Step 3: 探索性分析
├── 分布分析
├── 相关性分析
└── 异常检测

Step 4: 问题定位
├── 假设生成
├── 验证假设
└── 根因分析

Step 5: 方案设计
├── 优化方案
├── 预期效果
└── 风险评估

Step 6: 实验验证
├── A/B实验
├── 效果评估
└── 显著性检验

Step 7: 全量上线
├── 灰度发布
├── 监控告警
└── 持续优化
```

### 14.5.2 常用分析工具

```python
# 分布分析
def analyze_distribution(data, feature):
    """
    分析特征分布
    """
    return {
        'mean': data[feature].mean(),
        'std': data[feature].std(),
        'min': data[feature].min(),
        'max': data[feature].max(),
        'quantiles': data[feature].quantile([0.25, 0.5, 0.75]).to_dict(),
        'histogram': np.histogram(data[feature], bins=20)
    }

# 相关性分析
def analyze_correlation(data, features):
    """
    分析特征相关性
    """
    corr_matrix = data[features].corr()
    return corr_matrix

# 漂移检测
def detect_drift(train_data, online_data, feature):
    """
    检测分布漂移
    """
    psi = calculate_psi(train_data[feature], online_data[feature])
    return {
        'psi': psi,
        'drift_detected': psi > 0.1,
        'drift_level': 'high' if psi > 0.25 else ('medium' if psi > 0.1 else 'low')
    }

# 显著性检验
def significance_test(control, treatment, metric):
    """
    显著性检验
    """
    from scipy import stats
    
    # t检验
    t_stat, p_value = stats.ttest_ind(control[metric], treatment[metric])
    
    # 效应量
    cohens_d = (treatment[metric].mean() - control[metric].mean()) / np.sqrt(
        (treatment[metric].std()**2 + control[metric].std()**2) / 2
    )
    
    return {
        't_statistic': t_stat,
        'p_value': p_value,
        'significant': p_value < 0.05,
        'cohens_d': cohens_d,
        'effect_size': 'large' if abs(cohens_d) > 0.8 else ('medium' if abs(cohens_d) > 0.5 else 'small')
    }
```

### 14.5.3 效果评估框架

```
效果评估维度：

1. 离线指标
├── AUC/GAUC
├── LogLoss
├── Calibration
└── NDCG

2. 在线指标
├── CTR/CVR
├── GMV/RPM
├── 用户停留时长
└── 用户留存率

3. 业务指标
├── 收入
├── 利润
├── 用户满意度
└── 市场份额

4. 系统指标
├── 延迟
├── QPS
├── 资源消耗
└── 稳定性

评估原则：
├── 离线指标是参考，在线指标是标准
├── 业务指标是最终目标
├── 系统指标是约束条件
└── 多指标综合评估，避免单一指标优化
```

---

[← 上一章：样本工程核心技能](13-sample-engineering.md) | [返回目录](../README.md)
