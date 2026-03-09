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

### 🚀 实操项目篇
- [十七、推荐系统实操项目详解](docs/17-practical-projects.md)

### 🏗️ 架构详解篇 ⭐新增
- [十八、推荐场景下网关与服务端工作详解](docs/18-gateway-server-work.md)

---

## 🏗️ 网关与服务端工作详解

### 职责划分

```
【网关层】
├── 核心职责：流量管理、安全防护
├── 主要工作：鉴权、限流、路由、灰度
└── 性能要求：P99 < 5ms

【服务端】
├── 核心职责：业务逻辑、推荐计算
├── 主要工作：召回、排序、重排、特征
└── 性能要求：P99 < 30ms
```

### 网关层六大职责

| 职责 | 说明 |
|-----|------|
| **请求鉴权** | 验证用户身份、Token验证 |
| **流量控制** | 限流、熔断、降级 |
| **路由分发** | 请求路由、负载均衡 |
| **灰度分流** | A/B实验、灰度发布 |
| **监控埋点** | 日志记录、指标采集 |
| **协议转换** | HTTP → RPC |

### 服务端六大职责

| 职责 | 说明 |
|-----|------|
| **召回服务** | 多路召回、向量检索 |
| **排序服务** | 特征获取、模型推理 |
| **重排服务** | 多样性、业务规则 |
| **特征服务** | 特征获取、缓存管理 |
| **模型服务** | 模型推理、热更新 |
| **数据服务** | 用户画像、商品信息 |

---

## 📝 版本信息

- 文档版本：v1.8
- 更新日期：2025年
- 作者：[qian9332](https://github.com/qian9332)

---

## ⭐ Star History

如果这个知识库对你有帮助，请给一个 ⭐ Star 支持一下！

