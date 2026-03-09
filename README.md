# 🔍 搜广推算法工程师知识体系

> 基于《搜广推算法工程师面试题库大全》系统梳理的推荐算法知识框架

[![Knowledge Base](https://img.shields.io/badge/Knowledge%20Base-22%20Modules-blue)](https://github.com/qian9332/recommendation-algorithm-knowledge)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

## 📚 知识体系概览

本文档系统梳理了搜索、广告、推荐（搜广推）系统的核心算法与架构知识，涵盖从召回到排序、从特征工程到模型部署的全链路技术栈。

---

## 📖 目录

### 📘 基础篇
- [一、基础召回链路设计](docs/01-recall.md)
- [二、排序模型特征工程](docs/02-ranking-features.md)

### 📗 进阶篇
- [三、多任务排序学习](docs/03-multi-task-learning.md)
- [四、重排与多样性策略](docs/04-rerank-diversity.md)

### 📙 工程篇
- [五、系统架构与工程实践](docs/05-system-architecture.md)

### 📕 深度篇
- [六、影响模型效果与性能的关键细节](docs/06-advanced-details.md)
- [七、粗排模型蒸馏与加速](docs/07-coarse-ranking-distillation.md)
- [八、广告竞价机制设计](docs/08-ad-bidding.md)
- [九、推荐冷启动与元学习](docs/09-cold-start-meta-learning.md)
- [十、强化学习在推荐中的应用](docs/10-reinforcement-learning.md)

### 📊 可视化篇
- [十一、可视化图表体系](docs/11-diagrams.md)
- [十二、快速参考卡片](docs/12-quick-reference.md)

### 🔬 核心技能篇
- [十三、样本工程：顶级算法工程师的核心技能](docs/13-sample-engineering.md)
- [十四、数据分析实战案例：从分析到指标提升](docs/14-data-analysis-cases.md)

### 💼 岗位指南篇
- [十五、推荐后端工程师岗位全景指南](docs/15-backend-engineer-guide.md)
- [十六、推荐后端工程师工作内容详解](docs/16-backend-engineer-work.md)

### 🚀 实操项目篇 ⭐新增
- [十七、推荐系统实操项目详解](docs/17-practical-projects.md)

---

## 🚀 实操项目详解

### 项目一：电商首页推荐系统

```
【项目背景】
├── 平台：日活用户5000万
├── 场景：首页信息流推荐
└── 目标：P99延迟<30ms，QPS>5万

【核心实现】
├── 召回服务：多路召回调度器
│   ├── Item-CF召回（倒排索引）
│   ├── 向量召回（Faiss ANN）
│   └── 并行调度优化
│
├── 精排服务：多目标排序
│   ├── 批量特征获取（Pipeline优化）
│   ├── 批量模型推理（TF Serving）
│   └── 多目标融合
│
└── 性能优化
    ├── P99延迟：45ms → 22ms（-51%）
    └── 最大QPS：3万 → 6万（+100%）
```

### 项目二：实时特征平台

```
【项目背景】
├── 问题：特征更新延迟高（小时级）
└── 目标：特征更新延迟<5秒

【核心实现】
├── 数据采集：Kafka消息队列
├── 实时计算：Flink流处理
│   ├── 窗口聚合
│   └── 状态管理
│
├── 存储层：Redis + HBase
└── 服务层：特征查询服务

【优化效果】
├── 特征更新延迟：3分钟 → 3秒（-99%）
└── 写入吞吐量：10万 → 100万QPS（+900%）
```

### 项目三：A/B实验平台

```
【项目背景】
├── 问题：算法迭代效率低（实验周期2周）
└── 目标：实验周期缩短到3天

【核心实现】
├── 分流服务：分层正交实验
├── 实验执行：多组并行
└── 数据分析：自动显著性检验

【优化效果】
├── 实验周期：14天 → 3天（-79%）
├── 并行实验数：10个 → 100个（+900%）
└── 分析耗时：2天 → 1小时（-98%）
```

---

## 📝 版本信息

- 文档版本：v1.7
- 更新日期：2025年
- 作者：[qian9332](https://github.com/qian9332)

---

## ⭐ Star History

如果这个知识库对你有帮助，请给一个 ⭐ Star 支持一下！

