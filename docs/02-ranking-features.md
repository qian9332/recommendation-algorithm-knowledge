# 二、排序模型特征工程

## 2.1 连续特征离散化方法

### 自动分桶算法对比

| 算法 | 碰撞率 | χ²偏离度 | 时间漂移 | 回滚一致性 |
|-----|--------|---------|---------|-----------|
| Hash+Salt固定分桶 | 0 | 0.18 | 0.3% | 100% |
| 确定性伪随机发生器 | 0 | 0.22 | 0.8% | 99.97% |
| 分层正交拉丁方 | 0 | 0.05 | 0.1% | 100% |

### 分桶后特征边际增益验证

```python
# 五步验证流程
# 1. 离线构造唯一变量
#    - 固定数据切片：14天日志，8:2时间切分
#    - 固定模型骨架：DeepFM + ESMM
#    - 唯一差异：实验组分桶，对照组保持原值

# 2. AUC计算与显著性
#    - DeLong检验 p-value < 0.05
#    - Bootstrap 1000次，95%置信区间下限>0

# 3. 消融与一致性
#    - 桶数-AUC曲线找拐点
#    - 分层看AUC，排除辛普森悖论

# 4. 线上实验设计
#    - 5%流量A/B，AA实验24h
#    - 功效分析：power≥80%

# 5. 结论与落地
#    - 离线ΔAUC统计显著 + 线上核心指标正向 = 全量
```

### Streaming场景动态分桶边界

```
┌─────────────────────────────────────────────────────────┐
│                 动态分桶边界更新流程                      │
├─────────────────────────────────────────────────────────┤
│  Step 1: 在线近似分位数收集                             │
│  └── Flink维护KLL Sketch，窗口5min，ε=0.5%              │
├─────────────────────────────────────────────────────────┤
│  Step 2: 边界决策与灰度                                 │
│  └── KS > 0.02 触发更新，新边界写PS，版本号+1           │
├─────────────────────────────────────────────────────────┤
│  Step 3: 热切换与回滚                                   │
│  └── TF Serving版本策略，CTR下降>1%自动回滚             │
└─────────────────────────────────────────────────────────┘
```

---

## 2.2 高基数ID特征编码

### 三种编码方式权衡

| 编码方式 | 内存(5亿ID) | CTR效果 | P99延迟 | 冷启动 |
|---------|------------|---------|---------|--------|
| HashEncoding | 256MB | 基准 | 0.8ms | 差 |
| Embedding | 120GB | +6.1% | 2.5ms | 需迁移学习 |
| TargetEncoding | 400MB | +2.3% | 1.2ms | 中等 |

### 分布式Embedding表设计

```
10亿商品ID × 32维 × 4Byte ≈ 128GB

三级分层架构：
├── 分片层：MurmurHash到2^20虚拟桶，映射到64台PS节点
├── 存储层：热数据DRAM+GPU，冷数据SSD mmap
└── 服务层：本地Cache 256MB，批量查询200 ID/次

性能指标：
├── P99延迟：4.3ms
├── 扩缩容：单桶<2GB，10分钟完成重平衡
└── 容灾：Raft日志，RPO<30s
```

### OOV商品优雅降级

```python
# 三阶段降级流水线

# 阶段1：特征层降级
# - 缺失商品ID Embedding → 类目聚合向量 → 卖家聚合向量 → 全局零向量
# - 统计特征用类目-价格带贝叶斯平滑值填充

# 阶段2：模型层降级
# Score = sigmoid(W·x + b + γ·is_oov)
# γ为可学习负偏置，训练时梯度隔离

# 阶段3：后校准层
# 单调性闸门：if is_oov && score > prev_normal_score:
#                 score = prev_normal_score * 0.95
```

---

## 2.3 交叉特征自动挖掘

### FM、DeepFM、xDeepFM对比

| 模型 | 显式最高阶数 | 隐式深度 | 交互方式 | 计算复杂度 |
|-----|-------------|---------|---------|-----------|
| FM | 二阶 | 无 | pair-wise内积 | O(kn) |
| DeepFM | 二阶 | 隐式高阶 | bit-wise | 中 |
| xDeepFM | 四阶+ | 隐式高阶 | vector-wise | 高 |

### AutoInt多头注意力设置

```python
# 经验公式
# 单头维度 d_head ≥ 16 才能稳定收敛
# 总维度 d_model = h × d_head ≤ 256

# 典型配置（60-field场景）
h = 8  # 头数
d_head = 32  # 单头维度
d_model = 256  # 总维度

# 性能数据（A100 40G）
# h=8: 显存利用率45%，P99延迟28ms
# h=16: 显存利用率78%，P99延迟42ms（超SLA）
```

### 二阶特征池化层实现

```python
import tensorflow as tf

class SecondOrderPool(tf.keras.layers.Layer):
    """可训练权重的一次性二阶特征池化层"""
    
    def __init__(self, projection_dim=64, l2_reg=1e-6, **kwargs):
        super().__init__(**kwargs)
        self.projection_dim = projection_dim
        self.l2_reg = l2_reg
    
    def build(self, input_shape):
        last_dim = int(input_shape[-1])
        self.V = self.add_weight(
            name='V',
            shape=(last_dim, self.projection_dim),
            initializer='glorot_uniform',
            regularizer=tf.keras.regularizers.l2(self.l2_reg),
            trainable=True
        )
        super().build(input_shape)
    
    def call(self, inputs, mask=None):
        # 投影到k维
        p = tf.einsum('bte,ek->btk', inputs, self.V)
        
        # mask处理
        if mask is not None:
            p = p * tf.expand_dims(tf.cast(mask, tf.float32), -1)
        
        # FM压缩公式
        sum_p = tf.reduce_sum(p, axis=1)
        sum_p2 = tf.reduce_sum(p * p, axis=1)
        second_order = 0.5 * (tf.square(sum_p) - sum_p2)
        
        return second_order
```

---

## 2.4 多模态特征融合

### 图文融合方式对比

| 融合方式 | CTR提升 | GMV提升 | P99延迟 | 推荐度 |
|---------|--------|---------|---------|--------|
| Concat | 基准 | 基准 | 23ms | 低 |
| MHA | +0.31% | +0.28% | 38ms | 中 |
| Cross-attention | +0.29% | +0.26% | 24ms | 高 |

### 模态缺失处理策略

```
三段式处理：
1. 模态随机丢弃训练（p=0.2丢弃图像，p=0.1丢弃文本）
2. 缺失指示器 + 可学习token替代
3. 模态置信门控（MCG）动态加权

效果：
├── 图像缺失带来的CTR抖动：5.2% → 1.1%
├── ΔGMV回正：+0.4%
└── 实验置信度提升
```

---

## 2.5 特征重要性评估

### Permutation Importance vs SHAP

| 维度 | Permutation Importance | SHAP |
|-----|----------------------|------|
| 时间复杂度 | O(n·k) | O(n·k·2^m) |
| 十亿样本可行性 | 高（1.5小时） | 低（3天+） |
| 精度 | 近似 | 精确 |
| 国内落地 | 主流 | 仅用于审计 |

### PSI稳定性指标计算

```python
# 分布式Spark计算PSI
# PSI = Σ((实际占比 − 期望占比) × ln(实际占比 / 期望占比))

# 优化策略
# 1. 一次扫描多特征并行计算
# 2. 广播期望分布，避免二次join
# 3. 自定义聚合函数，WholeStageCodeGen优化

# 性能数据
# 1000亿样本、2万特征
# 200 executor × 4 core × 8G
# 运行时间：4.3分钟
```

---

[← 上一章：基础召回链路设计](01-recall.md) | [返回目录](../README.md) | [下一章：多任务排序学习 →](03-multi-task-learning.md)
