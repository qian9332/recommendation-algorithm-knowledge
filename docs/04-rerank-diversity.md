# 四、重排与多样性策略

## 4.1 行列式点过程（DPP）原理

### 核矩阵构造

```
L_ij = q_i · q_j · S_ij

其中：
- q_i：商品i的质量分
- S_ij：商品i与j的相似度
- 多样性强度通过温度系数τ调节：S_ij^τ
```

### DPP采样算法对比

| 算法 | 效果 | 速度 | 适用规模 |
|-----|------|------|---------|
| Greedy | 中 | 快 | Top-50 |
| MCMC | 高 | 慢 | 小规模 |
| Spectral | 高 | 中 | 中等规模 |

### 100ms内完成Top-50 DPP采样

```
优化策略：
1. 核矩阵低秩近似
   └── 用Cholesky分解，复杂度从O(n³)降到O(n²k)

2. 滑动窗口
   └── 只对当前窗口内商品做DPP
   └── 窗口大小=10，步长=5

3. GPU加速
   └── CUDA kernel实现矩阵运算
   └── 单次采样<5ms
```

---

## 4.2 滑动窗口多样性控制

### 类目距离矩阵

```python
# 类目距离定义
def category_distance(cat1, cat2):
    """基于类目树的距离计算"""
    # LCA深度法
    lca_depth = find_lca_depth(cat1, cat2)
    return 1.0 / (lca_depth + 1)

# 相似度矩阵
S_ij = 1 - category_distance(cat_i, cat_j)
```

### 多样性与相关性平衡

```
在线权重调节：
score_final = α·relevance_score + (1-α)·diversity_penalty

α动态调整策略：
├── 用户历史多样性偏好高 → α↓
├── 当前session已展示同类目多 → α↓
└── 用户明确意图（搜索） → α↑
```

---

## 4.3 强化学习重排模型

### MDP建模

| 要素 | 定义 |
|-----|------|
| 状态S | 用户历史行为序列、当前候选集、上下文特征 |
| 动作A | 对候选集的排列顺序 |
| 奖励R | List-wise奖励函数（点击、转化、停留时长加权） |
| 折扣因子γ | 0.9（兼顾短期与长期） |

### 算法对比

| 算法 | 方差表现 | 适用场景 |
|-----|---------|---------|
| REINFORCE | 高 | 简单场景 |
| Actor-Critic | 中 | 通用场景 |
| DDPG | 低 | 连续动作空间 |

### Off-policy修正

```python
# IPS+DR降低策略评估方差
def doubly_robust_estimate(reward, propensity, target_policy, value_estimate):
    """
    Doubly Robust估计
    """
    ips_weight = target_policy / propensity
    dr_estimate = (
        value_estimate + 
        ips_weight * (reward - value_estimate)
    )
    return dr_estimate
```

---

## 4.4 用户疲劳建模

### 疲劳衰减函数对比

| 函数类型 | 公式 | 特点 |
|---------|------|------|
| 指数衰减 | f(t) = e^(-λt) | 平滑，但衰减过快 |
| 阶跃衰减 | f(t) = 1 if t<T else 0 | 简单，但不连续 |
| 自适应衰减 | f(t) = 1/(1+λt) | 平衡，推荐使用 |

### 疲劳因子引入

```
重排阶段疲劳惩罚：
score_final = score × fatigue_factor

fatigue_factor计算：
├── 统计用户近期同类目曝光次数n
├── fatigue_factor = 1 / (1 + λ·n)
└── λ根据类目敏感度调整
```

---

## 4.5 业务规则约束

### 保量约束

```
广告主保量需求：
├── 合同约定曝光量：N
├── 剩余需完成量：N_remaining
├── 剩余时间窗口：T
└── 每小时目标：N_remaining / T

实现方式：
├── 重排阶段提升保量商品权重
├── 动态调整权重：w = base_weight × pacing_factor
└── pacing_factor = (目标进度 / 实际进度)
```

### 禁投规则

```python
# 禁投规则引擎
class BlockRuleEngine:
    def __init__(self):
        self.rules = []
    
    def add_rule(self, condition, action):
        """添加禁投规则"""
        self.rules.append((condition, action))
    
    def filter(self, items, user_info):
        """过滤禁投商品"""
        result = []
        for item in items:
            blocked = False
            for condition, action in self.rules:
                if condition(item, user_info):
                    blocked = True
                    break
            if not blocked:
                result.append(item)
        return result
```

---

[← 上一章：多任务排序学习](03-multi-task-learning.md) | [返回目录](../README.md) | [下一章：系统架构与工程实践 →](05-system-architecture.md)
