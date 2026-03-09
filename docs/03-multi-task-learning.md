# 三、多任务排序学习

## 3.1 共享底层网络设计

### MMoE、PLE、CGC对比

| 结构 | 参数隔离度 | 门控稳定性 | 冲突缓解 | 线上效果 |
|-----|-----------|-----------|---------|---------|
| MMoE | 0% | 易"赢者通吃" | 弱 | CVR涨1%时CTR掉0.6% |
| CGC | ~30% | 中等 | 中 | 中等 |
| PLE | >60% | 门控熵>0.8 | 强 | CTR+1.2%、CVR+2.1%、GMV+3.4% |

### 门控网络温度系数设置

```python
# 门控公式
g_i = exp(z_i / τ) / Σ_j exp(z_j / τ)

# τ设置策略
# τ→0：趋近one-hot（赢者通吃）
# τ→∞：趋近均匀分布

# 三步设置法
# Step 1: 离线熵校准
#    - 固定τ=1.0训练baseline
#    - 若H < ln(k)/2，提升τ到2.0

# Step 2: 业务导向退火
#    - τ_t = max(τ_final, τ_init·α^t)
#    - α=0.95, τ_init=2.0
#    - CTR任务最终τ=0.5，CVR任务最终τ=0.3

# Step 3: 线上可学习化
#    - τ作为标量参数训练
#    - 约束范围[0.3, 3.0]
#    - 损失加熵正则项
```

### GradNorm实现步骤

```python
# 梯度归一化（GradNorm）实现

# Step 1: 构建多任务网络
# 共享底层 + T个任务塔

# Step 2: 计算各任务梯度范数
# G_i(t) = ∂(w_i·L_i)/∂W
# ||G_i(t)||_2

# Step 3: 计算相对训练速率
# r_i(t) = L_i(t) / L_i(0)

# Step 4: 构造GradNorm损失
# L_grad = Σ| ||G_i(t)||_2 − α(t)·r_i(t)^β |

# Step 5: 更新任务权重
# ∂L_grad/∂w_i，投影到简单x约束

# Step 6: 融合损失
# L_total = Σw_i·L_i + λ·L_grad
```

### 任务不确定性加权Loss

```python
import torch
import torch.nn as nn

class UncertaintyWeightedLoss(nn.Module):
    """任务不确定性加权Loss"""
    
    def __init__(self, num_tasks: int, eps: float = 1e-6):
        super().__init__()
        self.log_sigma = nn.Parameter(torch.zeros(num_tasks))
        self.eps = eps
    
    def forward(self, losses: list) -> torch.Tensor:
        precision = torch.exp(-self.log_sigma)
        
        # 主loss项
        loss_sum = sum(p * l for p, l in zip(precision, losses))
        
        # 正则项
        reg_sum = torch.sum(self.log_sigma)
        
        # 总loss
        total = 0.5 * loss_sum + reg_sum
        return total
    
    def export_weights(self):
        """导出固化权重供线上推理"""
        with torch.no_grad():
            sigma = torch.exp(self.log_sigma).clamp(min=self.eps)
            weights = (1.0 / (2.0 * sigma)).cpu().tolist()
        return weights
```

---

## 3.2 跷跷板现象缓解

### 根本原因分析

```
跷跷板现象三大根因：

1. 目标函数梯度冲突
   └── CTR与CVR在共享Embedding上的梯度夹角>90°
   └── 参数更新方向互斥

2. 样本空间不一致
   └── 高CTR样本：低价引流商品
   └── 高CVR样本：高信任品牌
   └── 样本重叠率<20%

3. 资源硬约束
   └── 首屏仅10个曝光位
   └── 任何目标增益都挤占其他目标流量
```

### 可观测指标

| 指标 | 正常范围 | 异常信号 |
|-----|---------|---------|
| CTR-CVR Pearson相关 | 正值 | 由正转负 |
| GMV/曝光与CTR斜率 | 同向 | 反向变化 |
| Pareto超体积(PV) | 稳定/上升 | 连续3天下跌>2% |

### 缓解策略对比

| 策略 | 副作用 | 适用场景 |
|-----|--------|---------|
| 梯度裁剪 | 方向保持≠方向最优 | 快速止血 |
| 梯度投影 | 方向偏差大，计算开销高 | 广告出价信任域 |
| 梯度修正 | 超参爆炸，可能"反学习" | RL重排 |

---

## 3.3 多任务线上Serving

### TensorFlow Serving多输出签名

```python
import tensorflow as tf

class MultiTaskModel(tf.keras.Model):
    def __init__(self):
        super().__init__()
        self.embedding = tf.keras.layers.Embedding(vocab_size, dim)
        self.ctr_tower = tf.keras.layers.Dense(1, activation='sigmoid', name='ctr_head')
        self.cvr_tower = tf.keras.layers.Dense(1, activation='sigmoid', name='cvr_head')
    
    def call(self, inputs, training=None):
        feat = self.embedding(inputs['item_id'])
        ctr_score = self.ctr_tower(feat)
        cvr_score = self.cvr_tower(feat)
        return {'ctr_prob': ctr_score, 'cvr_prob': cvr_score}

# 导出签名
model = MultiTaskModel()
tf.keras.backend.set_learning_phase(0)

@tf.function
def serving_fn(inputs):
    return model(inputs, training=False)

signatures = {"serving_default": serving_fn}
tf.saved_model.save(model, export_dir="./ctr_cvr_model/1", signatures=signatures)
```

### 概率校准与单调性保证

```
三段式校准方案：

1. 离线阶段：分桶保序映射
   └── 等频分100桶，统计桶内真实CTR
   └── PAV算法做保序回归
   └── 缓存为分段线性LUT（<200KB）

2. 在线阶段：毫秒级校准
   └── CPU二分查找定位区间
   └── 线性插值得到校准后概率
   └── EWMA平滑增量更新

3. 统一刻度：概率对齐层
   └── 全局Temperature Scaling
   └── p_final = σ(logit / T)
   └── 只改数值刻度，不改序
```

---

## 3.4 样本标签构造与偏差纠正

### CVR延迟反馈时间窗设计

```
三阶段量化方法：

阶段1：生存分布拟合
└── 取近90天点击-转化日志
└── 按类目、出价类型、用户新老分层
└── 拟合威布尔分布参数λ、k
└── 计算t95使S(t95)=5%

阶段2：业务可结算边界裁剪
└── 与财务对齐账期政策
└── 若账期 < t95，取账期；否则保留t95

阶段3：回溯模拟选拐点
└── 用T0-24h、T0、T0+24h三档窗口回溯
└── 观测ΔCVR、GAUC、训练延迟
└── 选择ΔCVR<1%且GAUC降幅<0.3%的最小窗口
```

### ESMM估计偏差评估

```
三层评估体系：

1. 分层校准检查（ECE分区）
   └── 按pCVR排序后十等分
   └── ECE = Σ(N_k/N_total)·|pCVR_k − trueCVR_k|
   └── ESMM的ECE > 3%即判定系统性偏差

2. PCOC与MRSE双指标
   └── PCOC = ΣpCVR / Σlabel_cvr
   └── MRSE = mean((pCVR − label_cvr)² / (label_cvr + ε))
   └── 电商大促：PCOC∈[0.95,1.05]为绿色区间

3. IPS/DR无偏基准
   └── 用随机流量桶收集无偏估计
   └── Bias_Rate = |ESMM_CVR − trueCVR_unbiased| / trueCVR_unbiased
   └── Bias_Rate > 8%则拒绝上线
```

---

[← 上一章：排序模型特征工程](02-ranking-features.md) | [返回目录](../README.md) | [下一章：重排与多样性策略 →](04-rerank-diversity.md)
