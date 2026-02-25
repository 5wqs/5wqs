工作流流程引擎 (WorkflowProcessEngine)
https://img.shields.io/badge/license-Apache%25202.0-blue.svg
https://img.shields.io/badge/Java-8+-green.svg
https://img.shields.io/badge/Caffeine-3.x-yellow.svg

一个轻量级、基于内存的工作流编排引擎，通过有向无环图（DAG）管理节点依赖，自动触发符合条件的后续节点执行。适用于低并发、短流程的业务场景（如促销活动、审批流程）。

📖 目录
特性

架构设计

核心概念

快速开始

使用示例

API参考

注意事项

许可证

✨ 特性
轻量易用：纯内存实现，无需外部中间件，开箱即用。

依赖驱动：基于节点间的依赖关系自动推进流程，减少手动编码。

并发安全：通过实例级别的锁防止同一流程的并发处理。

可扩展：支持自定义节点标识、处理器和上下文，适应不同业务需求。

循环依赖检测：添加依赖时自动检测环路，避免流程死锁。

🏗 架构设计
核心组件关系图




















节点执行时序图
