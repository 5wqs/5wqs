# 工作流流程引擎 (WorkflowProcessEngine)

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
![Java](https://img.shields.io/badge/Java-8+-green.svg)
![Caffeine](https://img.shields.io/badge/Caffeine-3.x-yellow.svg)

一个轻量级、基于内存的工作流编排引擎，通过有向无环图（DAG）管理节点依赖，自动触发符合条件的后续节点执行。适用于低并发、短流程的业务场景（如促销活动、审批流程）。

---

## 📖 目录

- [特性](#特性)
- [架构设计](#架构设计)
- [核心概念](#核心概念)
- [快速开始](#快速开始)
- [使用示例](#使用示例)
- [API参考](#api参考)
- [注意事项](#注意事项)
- [许可证](#许可证)

---

## ✨ 特性

- **轻量易用**：纯内存实现，无需外部中间件，开箱即用。
- **依赖驱动**：基于节点间的依赖关系自动推进流程，减少手动编码。
- **并发安全**：通过实例级别的锁防止同一流程的并发处理。
- **可扩展**：支持自定义节点标识、处理器和上下文，适应不同业务需求。
- **循环依赖检测**：添加依赖时自动检测环路，避免流程死锁。

---

## 🏗 架构设计

### 核心组件关系图

```mermaid
graph TB
    subgraph 外部调用
        A[客户端] --> B[调用 process 方法]
    end

    subgraph WorkflowProcessEngine
        direction TB
        C[节点处理器注册表<br/>nodeProcessors] 
        D[依赖关系图<br/>nodeDependencies]
        E[流程实例状态池<br/>processInstances]
        F[实例锁缓存<br/>LOCKS]
    end

    B --> G{获取实例锁}
    G -->|成功| H[获取/创建实例状态]
    H --> I[执行当前节点处理器]
    I --> J[标记节点完成]
    J --> K[遍历查找可执行后续节点]
    K --> L[递归调用 process]
    L --> M{所有节点完成?}
    M -->|是| N[移除实例状态]
    M -->|否| 结束
    
    C -.-> I
    D -.-> K
    E -.-> H
    F -.-> G
