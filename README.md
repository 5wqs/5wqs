# 工作流流程引擎技术文档

## 1. 概述

工作流流程引擎（`WorkflowProcessEngine`）是一个轻量级的**内存级流程驱动组件**，用于管理和驱动具有前置依赖关系的节点任务。它基于有向无环图（DAG）模型，支持节点注册、依赖注入、状态追踪以及自动触发后续节点。该引擎适用于非持久化、低并发、短生命周期的流程场景（如一次性批处理、测试环境模拟等）。

> ⚠️ **重要提示**：引擎状态完全驻留在内存中，**不支持持久化**，且未引入分布式锁，因此**不适合高并发、长周期或关键业务场景**。

## 2. 架构设计

### 2.1 整体架构图

```mermaid
graph TD
    A[外部调用] --> B{WorkflowProcessEngine}
    B --> C[注册节点处理器<br/>nodeProcessors]
    B --> D[维护依赖关系<br/>nodeDependencies]
    B --> E[流程实例状态缓存<br/>processInstances]
    B --> F[分布式锁缓存<br/>LOCKS（Caffeine）]
    
    C --> G[NodeProcessor]
    D --> H[Set<IProcessNode> 前置节点]
    E --> I[ProcessInstanceState]
    I --> J[completedNodes]
    
    subgraph 处理流程
        K[process(instanceId, currentNode, context)] --> L[获取/创建实例状态]
        L --> M[执行NodeProcessor]
        M --> N{是否完成?}
        N -->|是| O[标记节点完成]
        O --> P[遍历所有节点]
        P --> Q{依赖满足?}
        Q -->|是| K
    end
```

### 2.2 核心组件

| 组件 | 说明 |
|------|------|
| `WorkflowProcessEngine` | 流程引擎主类，提供注册、依赖添加、节点处理等核心方法 |
| `IProcessNode` | 节点标识接口，通常由枚举实现，用于区分不同节点 |
| `NodeProcessor` | 节点处理器接口，定义节点执行逻辑 |
| `ProcessContext` | 流程上下文接口，用于传递流程共享数据 |
| `ProcessInstanceState` | 流程实例状态类，记录某个实例下各节点的完成情况 |
| `Caffeine Cache` | 用于存储流程级别锁对象，防止并发处理同一实例 |

## 3. 核心功能

### 3.1 节点注册与依赖管理

- **注册节点处理器**：将节点与其处理逻辑绑定。
- **添加依赖关系**：指定某个节点依赖于一个或多个前置节点。引擎在添加依赖时会自动进行**循环依赖检测**，若发现环则拒绝添加并抛出异常。

### 3.2 流程驱动

- 通过 `process(processInstanceId, currentNode, context)` 方法触发节点处理。
- 每个流程实例拥有独立的状态（`ProcessInstanceState`），记录已完成的节点。
- 节点执行成功后，引擎自动检查所有依赖该节点的后续节点，若其所有前置节点均已完成，则递归触发后续节点处理。
- 当所有节点完成时，自动清理实例状态，释放内存。

### 3.3 并发控制

- 使用 `Caffeine` 缓存为每个流程实例生成一个锁对象（`LOCKS`），并通过 `synchronized` 块保证同一时刻只有一个线程在处理同一流程实例。

## 4. 工作流程详解

1. **初始化引擎**：创建 `WorkflowProcessEngine` 实例。
2. **注册节点及依赖**：
   - 调用 `registerProcessor(node, processor)` 注册节点处理器。
   - 调用 `addDependency(node, prerequisite)` 建立节点间的依赖关系。
3. **外部触发**：调用 `process(instanceId, startNode, context)` 开始处理某个节点（通常为起始节点）。
4. **节点执行**：
   - 获取或创建实例状态。
   - 根据 `currentNode` 查找对应的 `NodeProcessor` 并执行。
   - 若处理器返回 `true`，标记该节点为已完成。
5. **后续节点触发**：
   - 遍历所有已注册节点，找出依赖当前节点的后续节点。
   - 检查这些后续节点的所有前置节点是否均已完成，若满足则递归调用 `process` 处理该节点。
6. **流程结束**：当所有节点完成时，从 `processInstances` 中移除该实例状态。

## 5. 使用指南

### 5.1 定义节点枚举

```java
public enum DemoNode implements IProcessNode {
    NODE_A,
    NODE_B,
    NODE_C
}
```

### 5.2 实现节点处理器

```java
public class DemoProcessor implements NodeProcessor {
    private final DemoNode node;
    
    public DemoProcessor(DemoNode node) {
        this.node = node;
    }
    
    @Override
    public boolean process(String processInstanceId, ProcessContext context) {
        System.out.println("处理节点：" + node + "，实例：" + processInstanceId);
        // 执行实际业务逻辑
        return true; // 返回true表示节点成功完成
    }
}
```

### 5.3 实现上下文

```java
public class DemoContext implements ProcessContext {
    // 自定义上下文数据
    private Map<String, Object> data = new HashMap<>();
    // getter/setter...
}
```

### 5.4 装配引擎并执行

```java
public class Demo {
    public static void main(String[] args) {
        WorkflowProcessEngine engine = new WorkflowProcessEngine();
        
        // 注册处理器
        engine.registerProcessor(DemoNode.NODE_A, new DemoProcessor(DemoNode.NODE_A));
        engine.registerProcessor(DemoNode.NODE_B, new DemoProcessor(DemoNode.NODE_B));
        engine.registerProcessor(DemoNode.NODE_C, new DemoProcessor(DemoNode.NODE_C));
        
        // 建立依赖：B依赖于A，C依赖于B
        engine.addDependency(DemoNode.NODE_B, DemoNode.NODE_A);
        engine.addDependency(DemoNode.NODE_C, DemoNode.NODE_B);
        
        // 创建上下文
        DemoContext context = new DemoContext();
        
        // 触发流程（从A开始）
        engine.process("instance-001", DemoNode.NODE_A, context);
        
        // 输出：NODE_A -> NODE_B -> NODE_C 依次执行
    }
}
```

## 6. 注意事项

1. **内存状态**：所有流程实例状态存储在 `ConcurrentHashMap` 中，应用重启后状态丢失。
2. **不支持并发实例**：虽然通过锁控制了同一实例的并发，但若大量不同实例同时触发，引擎本身无容量限制，可能造成内存溢出。
3. **节点处理器需幂等**：由于未提供重试或事务机制，处理器内部应保证幂等性，避免重复执行产生副作用。
4. **避免长时间阻塞**：节点处理器应快速返回，若需耗时操作建议异步处理，否则会阻塞锁并影响后续节点。
5. **循环依赖检测**：添加依赖时即时校验，但仅限于当前已存在的依赖关系，不保证后续修改的安全性。
6. **适用场景**：仅适用于低频、非关键、可容忍状态丢失的流程，如单元测试、简单批处理、临时任务编排。

## 7. 扩展性讨论

- **持久化支持**：可将 `ProcessInstanceState` 持久化到数据库，并在引擎启动时恢复，但需配合分布式锁。
- **异步执行**：当前为同步递归调用，可改造为线程池异步触发，提高吞吐量。
- **事件监听**：可引入节点完成事件，便于外部监听和日志记录。
- **超时与重试**：可增加节点执行超时控制和失败重试机制。

## 8. 总结

`WorkflowProcessEngine` 提供了一个简洁的基于内存的流程驱动解决方案，核心思想是利用 DAG 模型自动触发后续节点。其代码清晰、易于理解，适合小型项目或作为学习工作流引擎的入门示例。使用时务必注意其内存特性及适用范围，避免在生产高并发场景下直接使用。
```
