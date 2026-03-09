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
- [十一、可视化图表体系](docs/11-diagrams.md) - 16张架构图/流程图/决策树
- [十二、快速参考卡片](docs/12-quick-reference.md) - 公式/参数/阈值速查表

### 🔬 核心技能篇 ⭐新增
- [十三、样本工程：顶级算法工程师的核心技能](docs/13-sample-engineering.md)

---

## 🔬 样本工程核心要点

### 样本构造黄金法则

```
样本质量 > 样本数量 > 特征工程 > 模型结构
```

### 负样本构造四种方法

| 方法 | 优点 | 缺点 | 适用场景 |
|-----|------|------|---------|
| 随机负采样 | 简单快速 | 分布差异大 | 快速验证 |
| Batch内负采样 | 显存零增加 | 热门被过度惩罚 | 大规模训练 |
| 曝光未点击 | 最接近线上 | 位置偏差 | 精细化优化 |
| 难负例挖掘 | 难度可控 | 计算重 | 效果优先 |

### 样本偏差类型

| 偏差类型 | 解决方案 |
|---------|---------|
| 选择偏差 | IPS、DR |
| 位置偏差 | 位置去偏 |
| 曝光偏差 | 曝光模型 |
| 流行度偏差 | 流行度降权 |

---

## 📈 效果指标速查

### 召回层指标

| 指标 | 目标值 |
|-----|--------|
| Recall@K | >65% |
| Precision@K | >8% |
| 覆盖率 | >30% |

### 排序层指标

| 指标 | 典型提升 |
|-----|---------|
| AUC | +0.5% |
| GAUC | +0.4% |

### 业务指标

| 指标 | 典型提升 |
|-----|---------|
| CTR | +2~5% |
| CVR | +1~3% |
| GMV | +3~5% |

---

## 🛠️ 快速开始

### 推荐学习路径

```
初学者：基础篇 → 进阶篇 → 工程篇
进阶者：深度篇 → 可视化篇 → 核心技能篇
面试冲刺：快速参考卡片 → 可视化图表 → 样本工程
```

### 重点章节

1. **面试必看**：[快速参考卡片](docs/12-quick-reference.md)
2. **系统设计**：[可视化图表体系](docs/11-diagrams.md)
3. **效果优化**：[影响模型效果的关键细节](docs/06-advanced-details.md)
4. **样本处理**：[样本工程核心技能](docs/13-sample-engineering.md)

---

## 📝 版本信息

- 文档版本：v1.3
- 更新日期：2025年
- 作者：[qian9332](https://github.com/qian9332)

---

## ⭐ Star History

如果这个知识库对你有帮助，请给一个 ⭐ Star 支持一下！

