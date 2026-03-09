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
- [十六、推荐后端工程师工作内容详解](docs/16-backend-engineer-work.md) ⭐新增

---

## 💼 推荐后端工程师工作内容（面向非专业人士）

### 一句话解释

```
推荐后端工程师 = 让推荐系统能够"稳定、快速、可靠"地运行的人
```

### 通俗类比

```
【算法工程师】= 选品经理
├── 决定货架上放什么商品
└── 让顾客更容易找到想要的商品

【推荐后端工程师】= 超市运营经理
├── 确保货架不倒塌（系统稳定）
├── 确保顾客能快速找到商品（响应快）
├── 确保超市能容纳足够多的顾客（高并发）
└── 当出现问题时快速解决（故障处理）
```

### 日常工作清单

```
【早晨：健康检查】
├── 📊 查看系统监控（P99延迟、QPS、错误率）
├── 🔔 处理告警
└── 📋 查看今日任务

【上午：核心工作】
├── 💻 代码开发
└── 👥 代码评审

【下午：系统维护】
├── 🔧 系统优化
├── 🔍 问题排查
└── 📦 发布上线

【傍晚：总结沉淀】
├── 📝 文档编写
└── 📊 数据分析
```

### 核心工作内容

| 工作内容 | 通俗解释 |
|---------|---------|
| **系统开发** | 让算法模型能够在线上运行 |
| **性能优化** | 让系统运行得更快 |
| **故障处理** | 当系统出问题时快速解决 |
| **系统监控** | 实时监控系统健康状态 |

---

## 📈 数据分析实战案例

### 案例一：召回率提升
- 召回覆盖率：68% → 74%（+6%）
- GMV：+3.8%

### 案例二：CTR预估优化
- 在线CTR：3.2% → 3.45%（+7.8%）
- GMV：+8.8%

### 案例三：多任务跷跷板
- CTR AUC：+1.2%
- CVR AUC：+2.3%
- GMV：+11.3%

---

## 📝 版本信息

- 文档版本：v1.6
- 更新日期：2025年
- 作者：[qian9332](https://github.com/qian9332)

---

## ⭐ Star History

如果这个知识库对你有帮助，请给一个 ⭐ Star 支持一下！

