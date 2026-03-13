# DDD Skill

帮助用户使用领域驱动设计（Domain-Driven Design）原则进行特性设计和实现。

## 目录结构

```
ddd/
├── SKILL.md                      # 入口：5 步工作流 + 决策树
├── README.md                     # 本文件
└── references/                   # 按概念拆分的详细参考文档
    ├── primitive-types.md
    ├── value-objects.md
    ├── entities.md
    ├── aggregates.md
    ├── bounded-contexts.md
    ├── anti-corruption-layer.md
    ├── domain-events.md
    ├── domain-services.md
    ├── repositories.md
    ├── application-services.md
    ├── layered-architecture.md
    ├── ubiquitous-language.md
    └── verification-checklist.md
```

## 技术分类

### 战术设计（Tactical Design）

*"用什么来建模"* — 代码层面的构建块

| 文件 | 概念 | 核心要点 |
|------|------|----------|
| `primitive-types.md` | 原子类型 / Brand Type | 禁止原始类型；用 `Brand<T, B>` + 工厂函数实现类型安全 |
| `value-objects.md` | 值对象 | 无身份、不可变、按值比较，构造即验证 |
| `entities.md` | 实体 | 有唯一身份、有生命周期、按 ID 比较，行为丰富（避免贫血模型） |
| `aggregates.md` | 聚合根 + 工厂模式 | 一致性边界，所有修改通过根，一事务一聚合，包含 Factory 模式 |

### 战略设计（Strategic Design）

*"如何划分边界"* — 系统架构层面的拆分

| 文件 | 概念 | 核心要点 |
|------|------|----------|
| `bounded-contexts.md` | 限界上下文 | 同一术语不同含义时拆分；5 种上下文映射模式（Shared Kernel、Customer-Supplier、ACL、Open Host、Conformist） |
| `ubiquitous-language.md` | 通用语言 | 代码和业务术语对齐，不确定就问用户，记录在 `.agent/ddd/glossary.md` |

### 服务与事件（Services & Events）

*"如何协调行为"* — 跨实体/跨聚合的逻辑编排

| 文件 | 概念 | 核心要点 |
|------|------|----------|
| `domain-services.md` | 领域服务 | 无状态、纯业务逻辑、跨多个实体/聚合的计算 |
| `domain-events.md` | 领域事件 | 过去时命名、Collect-then-Dispatch、实现跨聚合最终一致性 |
| `application-services.md` | 应用服务 / 用例 | Load → Execute → Persist → Publish 编排模式，不含业务逻辑 |

### 架构（Architecture）

*"代码放在哪里"* — 分层和隔离策略

| 文件 | 概念 | 核心要点 |
|------|------|----------|
| `layered-architecture.md` | 四层架构 | UI → Application → Domain ← Infrastructure，Domain 层零外部依赖 |
| `anti-corruption-layer.md` | 防腐层 / ACL | 外部数据隔离，Pure/Impure 分离，DTO → 领域模型转换 |
| `repositories.md` | 仓储 | 每个聚合根一个 Repository，接口在 Domain 层，实现在 Infrastructure 层 |

### 质量保障（Quality）

| 文件 | 概念 | 核心要点 |
|------|------|----------|
| `verification-checklist.md` | 验证清单 | 实现后逐项检查：命名、类型安全、聚合规则、分层依赖等 |

## 应用逻辑（SKILL 工作流）

当触发 DDD skill 时，按以下 5 步流程执行：

```
1. UNDERSTAND (业务发现)
   │  一次问一个问题，理解目标、参与者、触发条件
   ▼
2. RESEARCH (技术发现)
   │  搜索现有代码，识别已有领域模型，优先复用
   ▼
3. EXTRACT (战略建模)
   │  确定限界上下文，建立通用语言术语表
   │  → 参考: bounded-contexts.md, ubiquitous-language.md
   ▼
4. DESIGN (战术规划)
   │  用决策树分类每个概念：
   │  ├─ 有唯一生命周期身份？→ Entity
   │  ├─ 无身份、按值比较？→ Value Object
   │  ├─ 跨多个实体的逻辑？→ Domain Service
   │  ├─ 创建逻辑复杂？→ Factory
   │  └─ 接收外部数据？→ Anti-Corruption Layer
   │  → 参考: entities.md, value-objects.md, aggregates.md 等
   ▼
5. IMPLEMENT (代码实现)
   │  按四层架构生成代码：
   │  ├─ Domain 层: Entity, VO, Aggregate, Domain Service, Event
   │  ├─ Application 层: Use Case (编排)
   │  ├─ Infrastructure 层: Repository 实现, ACL
   │  └─ UI 层: Controller, API route
   │  → 参考: layered-architecture.md, application-services.md, repositories.md
   ▼
   实现后运行 verification-checklist.md 逐项验证
```

## 核心原则

1. **业务优先** — 先理解业务，再写代码
2. **类型安全** — 原始类型禁止出现在领域逻辑中
3. **复用优先** — 搜索现有模型，避免重复定义
4. **行为丰富** — 业务逻辑放在 Entity / Domain Service 中，不做贫血模型
5. **边界清晰** — Domain 层零外部依赖，通过接口反转
