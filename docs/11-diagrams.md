# 十一、可视化图表体系

> 一图胜千言。本章用图表形式呈现推荐系统的核心架构与算法，帮助快速理解复杂概念。

---

## 11.1 推荐系统整体架构

```mermaid
graph TB
    subgraph 用户请求
        A[用户请求] --> B[特征提取]
    end
    
    subgraph 召回层
        B --> C1[协同过滤召回]
        B --> C2[向量召回]
        B --> C3[图谱召回]
        B --> C4[实时行为召回]
        B --> C5[规则召回]
        C1 & C2 & C3 & C4 & C5 --> D[召回融合]
    end
    
    subgraph 粗排层
        D --> E[粗排模型]
        E --> F[Top-500候选]
    end
    
    subgraph 精排层
        F --> G[精排模型]
        G --> H[Top-50候选]
    end
    
    subgraph 重排层
        H --> I[多样性重排]
        I --> J[业务规则]
        J --> K[最终展示]
    end
    
    subgraph 反馈闭环
        K --> L[用户行为]
        L --> M[特征更新]
        M --> B
    end
    
    style A fill:#e1f5fe
    style K fill:#c8e6c9
    style L fill:#fff3e0
```

---

## 11.2 召回层详细架构

```mermaid
graph LR
    subgraph 多路召回
        A[用户请求] --> B{召回策略选择}
        
        B --> C1[Item-CF]
        B --> C2[双塔DSSM]
        B --> C3[知识图谱]
        B --> C4[实时Session]
        B --> C5[热门规则]
    end
    
    subgraph 各路召回详情
        C1 --> D1[行为相似度]
        C1 --> D2[共现矩阵]
        
        C2 --> E1[用户塔]
        C2 --> E2[商品塔]
        E1 & E2 --> E3[向量检索]
        
        C3 --> F1[实体关系]
        C3 --> F2[图嵌入]
        
        C4 --> G1[点击序列]
        C4 --> G2[实时I2I]
        
        C5 --> H1[热门商品]
        C5 --> H2[新品扶持]
    end
    
    subgraph 融合策略
        D1 & D2 & E3 & F2 & G2 & H1 & H2 --> I[候选合并]
        I --> J[去重]
        J --> K[配额分配]
        K --> L[召回结果]
    end
    
    style A fill:#e3f2fd
    style L fill:#e8f5e9
```

---

## 11.3 双塔DSSM模型结构

```mermaid
graph TB
    subgraph 用户塔
        A1[用户ID] --> B1[Embedding]
        A2[用户历史] --> B2[序列编码]
        A3[用户画像] --> B3[特征编码]
        B1 & B2 & B3 --> C1[用户向量 u]
    end
    
    subgraph 商品塔
        D1[商品ID] --> E1[Embedding]
        D2[商品属性] --> E2[特征编码]
        D3[商品内容] --> E3[内容编码]
        E1 & E2 & E3 --> C2[商品向量 v]
    end
    
    subgraph 相似度计算
        C1 --> F[内积/余弦]
        C2 --> F
        F --> G[score = u·v]
    end
    
    subgraph 负样本策略
        G --> H{负样本选择}
        H --> I1[随机负采样]
        H --> I2[Batch内负采样]
        H --> I3[难负例挖掘]
    end
    
    style C1 fill:#bbdefb
    style C2 fill:#c8e6c9
    style G fill:#fff9c4
```

---

## 11.4 多任务学习网络结构对比

```mermaid
graph TB
    subgraph MMoE
        A1[输入特征] --> B1[Expert 1]
        A1 --> B2[Expert 2]
        A1 --> B3[Expert 3]
        B1 & B2 & B3 --> C1[Gate CTR]
        B1 & B2 & B3 --> C2[Gate CVR]
        C1 --> D1[Tower CTR]
        C2 --> D2[Tower CVR]
    end
    
    subgraph PLE
        A2[输入特征] --> E1[共享Expert]
        A2 --> E2[CTR独占Expert]
        A2 --> E3[CVR独占Expert]
        E1 & E2 --> F1[Gate CTR]
        E1 & E3 --> F2[Gate CVR]
        F1 --> G1[Tower CTR]
        F2 --> G2[Tower CVR]
    end
    
    subgraph 效果对比
        H1[MMoE: CVR+1%时CTR-0.6%]
        H2[PLE: CTR+1.2%, CVR+2.1%, GMV+3.4%]
    end
    
    style D1 fill:#ffcdd2
    style D2 fill:#ffcdd2
    style G1 fill:#c8e6c9
    style G2 fill:#c8e6c9
```

---

## 11.5 DPP多样性采样流程

```mermaid
flowchart TD
    A[候选商品集] --> B[计算质量分 q_i]
    A --> C[计算相似度 S_ij]
    
    B --> D[构造核矩阵 L]
    C --> D
    
    D --> E{选择采样算法}
    
    E -->|快速| F[Greedy贪心]
    E -->|精确| G[Spectral谱分解]
    
    F --> H[迭代选择]
    H --> I{达到Top-K?}
    I -->|否| H
    I -->|是| J[输出结果]
    
    G --> K[特征分解]
    K --> L[Bernoulli采样]
    L --> J
    
    subgraph 时间复杂度
        M[Greedy: O n*k² ]
        N[Spectral: O n³ + O n² ]
    end
    
    style A fill:#e3f2fd
    style J fill:#e8f5e9
```

---

## 11.6 强化学习MDP建模

```mermaid
stateDiagram-v2
    [*] --> S1: 用户进入
    
    S1 --> S2: 推荐商品A
    S2 --> S3: 用户点击
    S3 --> S4: 推荐商品B
    
    S2 --> S5: 用户跳过
    S5 --> S4: 推荐商品B
    
    S4 --> S6: 用户转化
    S6 --> [*]: 会话结束
    
    S4 --> S7: 用户离开
    S7 --> [*]: 会话结束
    
    note right of S1
        状态: 用户历史行为
        动作: 推荐商品
        奖励: 点击/转化
    end note
```

```mermaid
graph LR
    subgraph MDP五元组
        A[状态 S] --> B[用户行为序列]
        A --> C[用户画像]
        A --> D[上下文特征]
        
        E[动作 A] --> F[推荐商品列表]
        
        G[奖励 R] --> H[点击 +0.3]
        G --> I[转化 +1.0]
        G --> J[停留时长 +0.1]
        G --> K[负反馈 -0.5]
        
        L[转移 P] --> M[用户反馈决定]
        
        N[折扣 γ] --> O[0.9-0.99]
    end
    
    style A fill:#e3f2fd
    style E fill:#fff3e0
    style G fill:#e8f5e9
```

---

## 11.7 特征工程流程

```mermaid
flowchart LR
    subgraph 原始特征
        A1[用户特征]
        A2[商品特征]
        A3[上下文特征]
        A4[交叉特征]
    end
    
    subgraph 特征处理
        A1 & A2 & A3 & A4 --> B[特征清洗]
        B --> C[缺失值处理]
        C --> D[异常值处理]
    end
    
    subgraph 特征编码
        D --> E1[连续特征离散化]
        D --> E2[类别特征Embedding]
        D --> E3[序列特征编码]
        D --> E4[交叉特征构造]
    end
    
    subgraph 特征选择
        E1 & E2 & E3 & E4 --> F[特征重要性评估]
        F --> G[特征筛选]
        G --> H[特征降维]
    end
    
    subgraph 特征服务
        H --> I[特征存储]
        I --> J[在线特征服务]
    end
    
    style A1 fill:#e3f2fd
    style J fill:#e8f5e9
```

---

## 11.8 A/B实验设计

```mermaid
graph TB
    subgraph 实验设计
        A[实验目标] --> B[指标定义]
        B --> C[样本量计算]
        C --> D[流量分配]
    end
    
    subgraph 流量分层
        D --> E[Layer 1: 召回层]
        D --> F[Layer 2: 排序层]
        D --> G[Layer 3: 重排层]
        
        E --> E1[实验组 5%]
        E --> E2[对照组 5%]
        
        F --> F1[实验组 5%]
        F --> F2[对照组 5%]
        
        G --> G1[实验组 5%]
        G --> G2[对照组 5%]
    end
    
    subgraph 指标监控
        E1 & E2 & F1 & F2 & G1 & G2 --> H[实时指标]
        H --> I[核心指标]
        H --> J[护栏指标]
        
        I --> I1[GMV]
        I --> I2[CTR]
        I --> I3[CVR]
        
        J --> J1[P99延迟]
        J --> J2[QPS]
        J --> J3[错误率]
    end
    
    subgraph 决策
        I & J --> K{显著性检验}
        K -->|通过| L[全量发布]
        K -->|不通过| M[继续实验/回滚]
    end
    
    style A fill:#e3f2fd
    style L fill:#e8f5e9
    style M fill:#ffcdd2
```

---

## 11.9 冷启动策略选择

```mermaid
flowchart TD
    A[冷启动场景] --> B{冷启动类型}
    
    B -->|新用户| C[新用户冷启动]
    B -->|新商品| D[新商品冷启动]
    B -->|新场景| E[新场景冷启动]
    
    C --> C1{有注册信息?}
    C1 -->|是| C2[人口统计学特征]
    C1 -->|否| C3[实时行为捕获]
    C2 --> C4[跨域迁移]
    C3 --> C4
    
    D --> D1{有内容特征?}
    D1 -->|是| D2[内容特征编码]
    D1 -->|否| D3[知识图谱推理]
    D2 --> D4[零样本向量]
    D3 --> D4
    
    E --> E1{有相似场景?}
    E1 -->|是| E2[跨场景迁移]
    E1 -->|否| E3[元学习MAML]
    
    C4 & D4 & E2 & E3 --> F[探索策略]
    
    F --> G{探索方式}
    G --> G1[ε-greedy]
    G --> G2[UCB]
    G --> G3[Thompson Sampling]
    
    G1 & G2 & G3 --> H[冷启动评估]
    H --> I{效果达标?}
    I -->|是| J[正常服务]
    I -->|否| K[调整策略]
    K --> F
    
    style A fill:#e3f2fd
    style J fill:#e8f5e9
```

---

## 11.10 性能优化决策树

```mermaid
flowchart TD
    A[性能优化需求] --> B{瓶颈在哪?}
    
    B -->|延迟高| C[延迟优化]
    B -->|吞吐低| D[吞吐优化]
    B -->|内存大| E[内存优化]
    
    C --> C1{延迟分布}
    C1 -->|P99高| C2[长尾优化]
    C1 -->|平均高| C3[整体优化]
    
    C2 --> C2a[异步处理]
    C2 --> C2b[超时熔断]
    
    C3 --> C3a[模型压缩]
    C3 --> C3b[算子融合]
    C3 --> C3c[量化]
    
    D --> D1{计算密集?}
    D1 -->|是| D2[并行计算]
    D1 -->|否| D3[IO优化]
    
    D2 --> D2a[GPU加速]
    D2 --> D2b[分布式推理]
    
    D3 --> D3a[缓存优化]
    D3 --> D3b[批处理]
    
    E --> E1{内存分布}
    E1 -->|模型大| E2[模型压缩]
    E1 -->|特征大| E3[特征压缩]
    
    E2 --> E2a[知识蒸馏]
    E2 --> E2b[剪枝]
    E2 --> E2c[量化]
    
    E3 --> E3a[特征降维]
    E3 --> E3b[稀疏存储]
    
    style A fill:#e3f2fd
    style C2a fill:#fff3e0
    style C3a fill:#fff3e0
    style D2a fill:#fff3e0
    style E2a fill:#fff3e0
```

---

## 11.11 广告竞价流程

```mermaid
sequenceDiagram
    participant 广告主
    participant 竞价系统
    participant 排序引擎
    participant 用户
    
    广告主->>竞价系统: 设置出价、预算
    用户->>排序引擎: 发起请求
    排序引擎->>竞价系统: 请求广告
    
    竞价系统->>竞价系统: 筛选符合条件广告
    竞价系统->>竞价系统: 计算eCPM
    
    Note over 竞价系统: eCPM = pCTR × pCVR × bid
    
    竞价系统->>排序引擎: 返回广告列表
    排序引擎->>排序引擎: 综合排序
    排序引擎->>用户: 展示广告
    
    用户->>排序引擎: 点击/转化
    排序引擎->>竞价系统: 反馈结果
    竞价系统->>竞价系统: 扣费(GSP规则)
    竞价系统->>广告主: 扣费通知
```

---

## 11.12 实时特征处理流程

```mermaid
flowchart LR
    subgraph 数据源
        A1[用户行为日志]
        A2[商品变更日志]
        A3[系统事件]
    end
    
    subgraph 数据采集
        A1 & A2 & A3 --> B[Kafka消息队列]
    end
    
    subgraph 实时计算
        B --> C[Flink流处理]
        C --> D1[特征聚合]
        C --> D2[特征统计]
        C --> D3[特征更新]
    end
    
    subgraph 特征存储
        D1 & D2 & D3 --> E1[Redis热数据]
        E1 --> E2[Tair集群]
        E2 --> E3[特征服务]
    end
    
    subgraph 在线服务
        E3 --> F[召回服务]
        E3 --> G[排序服务]
        E3 --> H[重排服务]
    end
    
    subgraph 监控
        F & G & H --> I[延迟监控]
        I --> J{延迟异常?}
        J -->|是| K[降级处理]
        J -->|否| L[正常服务]
    end
    
    style A1 fill:#e3f2fd
    style E3 fill:#e8f5e9
    style K fill:#ffcdd2
```

---

## 11.13 模型训练流程

```mermaid
flowchart TD
    subgraph 数据准备
        A[原始日志] --> B[数据清洗]
        B --> C[特征提取]
        C --> D[样本构造]
        D --> E[数据集划分]
    end
    
    subgraph 模型训练
        E --> F[模型初始化]
        F --> G[前向传播]
        G --> H[损失计算]
        H --> I[反向传播]
        I --> J[参数更新]
        J --> K{收敛?}
        K -->|否| G
        K -->|是| L[模型评估]
    end
    
    subgraph 模型评估
        L --> M[离线指标]
        M --> N1[AUC]
        M --> N2[GAUC]
        M --> N3[LogLoss]
        
        L --> O[在线评估]
        O --> P1[CTR]
        O --> P2[GMV]
        O --> P3[延迟]
    end
    
    subgraph 模型部署
        N1 & N2 & N3 & P1 & P2 & P3 --> Q{指标达标?}
        Q -->|是| R[模型导出]
        Q -->|否| S[调参重训]
        S --> F
        
        R --> T[模型压缩]
        T --> U[灰度发布]
        U --> V[全量上线]
    end
    
    style A fill:#e3f2fd
    style V fill:#e8f5e9
```

---

## 11.14 算法选型决策图

```mermaid
flowchart TD
    A[推荐场景] --> B{候选集规模}
    
    B -->|小 <1万| C[精排优化]
    B -->|中 1-10万| D[粗排+精排]
    B -->|大 >10万| E[召回+粗排+精排]
    
    C --> C1{实时性要求}
    C1 -->|高| C2[轻量模型]
    C1 -->|低| C3[复杂模型]
    
    D --> D1{延迟预算}
    D1 -->|<10ms| D2[双塔粗排]
    D1 -->|<20ms| D3[轻量DNN粗排]
    
    E --> E1{召回路数}
    E1 -->|单路| E2[向量召回]
    E1 -->|多路| E3[多路召回融合]
    
    C2 --> F[模型选择]
    C3 --> F
    D2 --> F
    D3 --> F
    E2 --> F
    E3 --> F
    
    F --> G{任务数量}
    G -->|单任务| H[单塔模型]
    G -->|多任务| I[多任务模型]
    
    H --> H1{特征交叉}
    H1 -->|简单| H2[Wide&Deep]
    H1 -->|复杂| H3[DCN/DeepFM]
    
    I --> I1{任务冲突}
    I1 -->|弱| I2[MMoE]
    I1 -->|强| I3[PLE]
    
    style A fill:#e3f2fd
    style H2 fill:#fff3e0
    style H3 fill:#fff3e0
    style I2 fill:#c8e6c9
    style I3 fill:#c8e6c9
```

---

## 11.15 指标体系全景图

```mermaid
mindmap
  root((推荐系统指标))
    业务指标
      收入指标
        GMV
        RPM
        ARPU
      转化指标
        CTR
        CVR
        CTCVR
      用户指标
        DAU
        留存率
        人均时长
    模型指标
      排序能力
        AUC
        GAUC
        NDCG
      校准能力
        ECE
        PCOC
        Calibration
      多样性
        ILS
        ILD
        覆盖率
    系统指标
      性能指标
        P99延迟
        QPS
        吞吐量
      稳定性指标
        可用性
        错误率
        超时率
      资源指标
        CPU利用率
        内存占用
        GPU利用率
```

---

## 图表使用说明

### 如何阅读这些图表

1. **架构图**：理解系统整体结构，把握各模块关系
2. **流程图**：跟随箭头方向，理解数据/处理流程
3. **对比图**：关注颜色标注，理解不同方案的差异
4. **决策树**：根据条件分支，选择合适的技术方案
5. **时序图**：理解各组件间的交互顺序

### 图表与文档对应关系

| 图表 | 对应章节 |
|-----|---------|
| 推荐系统整体架构 | 第一章 召回链路 |
| 双塔DSSM模型 | 第一章 文本语义召回 |
| 多任务学习网络 | 第三章 多任务排序 |
| DPP采样流程 | 第四章 重排多样性 |
| 强化学习MDP | 第十章 强化学习 |
| 特征工程流程 | 第二章 特征工程 |
| A/B实验设计 | 第五章 系统架构 |
| 冷启动策略 | 第九章 冷启动 |
| 性能优化决策 | 第五章 性能优化 |

---

[← 上一章：强化学习在推荐中的应用](10-reinforcement-learning.md) | [返回目录](../README.md)
