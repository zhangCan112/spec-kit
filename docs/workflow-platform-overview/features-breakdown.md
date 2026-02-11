# Feature Breakdown: SDD Workflow Platform

## Overview

本文档详细拆解了 SDD 工作流平台的 12 个独立 Feature，每个 Feature 都有清晰的边界、成功标准和依赖关系。

---

## Feature 1: 工作流可视化与编辑 (workflow-visualization-and-editing)

### 目标
建立完整的工作流可视化与编辑能力，支持用户直接在画布上编辑工作流结构。这是平台的核心交互功能。

### 核心功能
- **工作流显示**
  - 显示 SDD 工作流的所有节点
    - Constitution
    - Specification
    - Clarification (optional)
    - Planning (Phase 0, 1, 2)
    - Tasks
    - Implementation
    - Analyze (optional)
    - Checklist (optional)
  - 显示节点之间的连接关系
  - 显示节点的基本信息（名称、描述）

- **可视化交互**
  - 工作流的缩放、平移
  - 拖拽节点移动位置
  - 选择节点和连接
  - 多选和批量操作

- **工作流编辑**
  - 添加新节点（从节点库拖拽）
  - 删除选中的节点
  - 添加节点连接（拖拽创建）
  - 删除节点连接
  - 编辑节点基本信息（名称、描述）
  - 编辑节点命令和模板
  - 调整连接关系

- **编辑工具**
  - 节点库面板（显示可用节点类型）
  - 节点属性编辑器
  - 连接编辑器
  - 撤销/重做功能
  - 复制/粘贴节点
  - 快捷键支持

### 技术组件
- **React Flow**: 流程图可视化和编辑库
- **WorkflowCanvas 组件**: 工作流画布，支持缩放、平移、节点拖拽
- **WorkflowNode 组件**: 工作流节点展示和交互
- **Connection 组件**: 节点连接线展示和编辑
- **NodeLibrary 组件**: 节点库面板，显示可拖拽的节点类型
- **NodeEditor 组件**: 节点属性编辑器
- **ConnectionEditor 组件**: 连接编辑器
- **UndoRedoManager**: 撤销/重做管理器
- **WorkflowStore**: Zustand store 管理工作流状态

### 数据模型
```typescript
interface WorkflowNode {
  id: string;
  type: 'standard' | 'decision' | 'conditional' | 'start' | 'end';
  name: string;
  description: string;
  command: string;
  templatePath: string;
  position: {
    x: number;
    y: number;
  };
}

interface WorkflowConnection {
  id: string;
  source: string;  // source node id
  target: string;  // target node id
  sourceHandle?: string;
  targetHandle?: string;
}

interface WorkflowDefinition {
  id: string;
  name: string;
  version: string;
  nodes: WorkflowNode[];
  connections: WorkflowConnection[];
  metadata: {
    createdAt: Date;
    updatedAt: Date;
    author: string;
  };
}

interface WorkflowEditAction {
  type: 'add-node' | 'remove-node' | 'update-node' | 
        'add-connection' | 'remove-connection' | 'update-connection';
  timestamp: Date;
  data: any;
}
```

### 成功标准
- [x] 用户能看到完整的 SDD 工作流图
- [x] 可以缩放、平移、拖拽节点查看
- [x] 可以从节点库拖拽添加新节点
- [x] 可以删除选中的节点
- [x] 可以通过拖拽创建节点连接
- [x] 可以删除节点连接
- [x] 可以编辑节点属性（名称、描述、命令、模板）
- [x] 撤销/重做功能正常工作
- [x] 可以复制/粘贴节点
- [x] 节点有清晰的视觉区分
- [x] 加载时间 <500ms
- [x] 编辑操作响应时间 <100ms

### 依赖
无独立依赖

### 输出产物
- `src/components/workflow/WorkflowCanvas.tsx`
- `src/components/workflow/WorkflowNode.tsx`
- `src/components/workflow/NodeLibrary.tsx`
- `src/components/workflow/NodeEditor.tsx`
- `src/components/workflow/ConnectionEditor.tsx`
- `src/lib/workflow/UndoRedoManager.ts`
- `src/stores/workflow-store.ts`
- `src/data/sdd-workflow.ts` (默认工作流定义)
- `src/data/node-templates.ts` (节点类型模板定义)

---

## Feature 2: 工作区管理系统 (workspace-management)

### 目标
实现工作区的 CRUD 操作和数据管理，为后续所有功能提供数据存储基础。

### 核心功能
- **CRUD 操作**
  - 创建工作区（名称、描述）
  - 编辑工作区元数据
  - 删除工作区
  - 克隆现有工作区
- **数据持久化**
  - 工作区列表展示
  - 工作区详情查看
  - 工作区导出/导入（JSON 格式）
  - 本地存储（SQLite 或文件系统）

### 技术组件
- **数据库/存储层**: SQLite 或文件系统 API
- **Workspace Store**: Zustand store 管理工作区状态
- **Workspace List 组件**: 工作区列表展示
- **Workspace Details 组件**: 工作区详情展示
- **Import/Export 组件**: 导入导出功能

### 数据模型
```typescript
interface Workspace {
  id: string;
  name: string;
  description: string;
  createdAt: Date;
  updatedAt: Date;
  author: string;
  baseVersion: string;
  isPublished: boolean;
  version: string;
}
```

### 成功标准
- [x] 用户可以创建和管理工作区
- [x] 工作区可以导出和导入
- [x] 基础的 CRUD 功能正常工作
- [x] 工作区保存时间 <200ms
- [x] 工作区列表加载时间 <500ms

### 依赖
无独立依赖

### 输出产物
- `src/components/workspace/WorkspaceList.tsx`
- `src/components/workspace/WorkspaceDetails.tsx`
- `src/components/workspace/WorkspaceEditor.tsx`
- `src/stores/workspace-store.ts`
- `src/lib/storage/workspace-storage.ts`

---

## Feature 3: 模板系统基础 (template-system-basic)

### 目标
实现模板的读取、展示和基础编辑能力，为模板同步和定制提供基础。

### 核心功能
- **模板读取**
  - 读取 Spec Kit 的模板文件（templates/ 目录）
  - 解析模板内容和元数据
- **模板展示**
  - 在 UI 中展示模板内容
  - Markdown 渲染和编辑器
- **模板编辑**
  - 内联 Markdown 编辑器
  - 保存自定义模板内容
  - 标记覆盖类型（partial/full）
- **模板管理**
  - 记录原始模板路径和内容
  - 记录原始模板 hash
  - 标记哪些模板已被定制

### 技术组件
- **模板读取器**: 文件系统 API 读取模板
- **Markdown 编辑器**: Monaco Editor 或 CodeMirror
- **Template Store**: Zustand store 管理模板状态
- **Template List 组件**: 模板列表展示
- **Template Editor 组件**: 模板编辑器
- **Diff Viewer 组件**: Diff 展示（基础）

### 数据模型
```typescript
interface TemplateOverride {
  originalPath: string;
  originalHash: string;
  originalContent: string;
  customContent: string;
  overrideType: 'partial' | 'full';
  notes: string;
  syncStatus: 'synced' | 'behind' | 'conflict' | 'ahead';
  lastSyncedAt: Date;
  baseVersion: string;
}
```

### 成功标准
- [x] 可以查看所有模板
- [x] 可以编辑模板内容
- [x] 修改会被保存
- [x] 模板加载时间 <300ms
- [x] 模板保存时间 <200ms

### 依赖
- Feature 2 (workspace-management): 模板覆盖数据需要存储在工作区中

### 输出产物
- `src/components/template/TemplateList.tsx`
- `src/components/template/TemplateEditor.tsx`
- `src/components/template/TemplateViewer.tsx`
- `src/lib/template/template-reader.ts`
- `src/lib/template/template-manager.ts`
- `src/stores/template-store.ts`

---

## Feature 4: 模板同步系统 (template-sync)

### 目标
实现模板的版本检测和同步能力，支持用户与 Spec Kit 官方更新保持同步。

### 核心功能
- **版本检测**
  - 检测 Spec Kit 原始模板的变化
  - 计算模板 content hash
  - 识别同步状态（synced, behind, conflict）
- **同步报告**
  - 生成同步报告
  - 识别新增、删除、修改的内容
  - 基础的 Diff 展示
- **同步状态管理**
  - 更新 syncStatus
  - 更新 baseVersion
  - 记录最后同步时间

### 技术组件
- **Hash 计算器**: 计算 content hash
- **同步检测器**: 检测模板变化
- **Diff 算法**: 基础的行级 Diff
- **Sync Manager**: 同步管理器
- **Sync Report 组件**: 同步报告展示
- **Diff Viewer 组件**: Diff 展示（增强）

### 同步算法
```typescript
enum SyncStatus {
  SYNCED = 'synced',      // 原始模板未变化
  BEHIND = 'behind',      // 原始模板有更新，无冲突
  CONFLICT = 'conflict',   // 原始模板有更新，有冲突
  AHEAD = 'ahead'          // 自定义版本领先（本地修改未发布）
}

interface TemplateSyncResult {
  templatePath: string;
  status: SyncStatus;
  changes: {
    added: string[];
    removed: string[];
    modified: string[];
  };
  conflicts?: ConflictSection[];
}

interface ConflictSection {
  lineStart: number;
  lineEnd: number;
  originalContent: string;
  newContent: string;
  customContent: string;
}
```

### 成功标准
- [x] 能检测到模板变化
- [x] 能显示哪些模板需要同步
- [x] 能展示 Diff
- [x] 同步检测时间 <5 秒（50 个模板）
- [x] Diff 生成时间 <1 秒

### 依赖
- Feature 3 (template-system-basic): 需要模板读取和管理能力

### 输出产物
- `src/lib/sync/hash-calculator.ts`
- `src/lib/sync/sync-detector.ts`
- `src/lib/sync/diff-algorithm.ts`
- `src/lib/sync/sync-manager.ts`
- `src/components/sync/SyncReport.tsx`
- `src/components/sync/DiffViewer.tsx`

---

## Feature 5: 扩展点数据模型与配置 (extension-point-model)

### 目标
建立扩展点的数据模型和基础配置界面，支持多种扩展点类型，特别是 AgentSkill。

### 核心功能
- **扩展点类型系统**
  - 定义扩展点类型：
    - agent-skill: AI agent skill 配置
    - pre-hook: 前置钩子
    - post-hook: 后置钩子
    - validation: 验证钩子
    - transformation: 转换钩子
    - custom: 自定义类型
- **扩展点数据模型**
  - 唯一标识符
  - 类型、名称、描述
  - 关联的工作流节点
  - 配置（类型特定）
  - 启用状态
  - 执行顺序
- **扩展点配置**
  - 创建、编辑、删除扩展点
  - 扩展点与节点的绑定
  - 支持多个扩展点
  - 动态启用/禁用
- **AgentSkill 配置**
  - 选择 AI 代理类型
  - 定义 skill 名称和描述
  - 配置 skill 参数
  - 设置执行时机
  - 设置组合模式

### 技术组件
- **扩展点类型注册表**: 扩展点类型定义和工厂
- **Extension Point Store**: Zustand store 管理扩展点状态
- **Extension Point Config 组件**: 扩展点配置界面
- **AgentSkill Config 组件**: AgentSkill 专用配置界面

### 数据模型
```typescript
type ExtensionPointType = 
  | 'agent-skill'
  | 'pre-hook'
  | 'post-hook'
  | 'validation'
  | 'transformation'
  | 'custom';

interface ExtensionPoint {
  id: string;
  type: ExtensionPointType;
  name: string;
  description: string;
  nodeId: string;
  position: 'pre' | 'post' | 'inline';
  config: ExtensionConfig;
  enabled: boolean;
  order: number;
  createdAt: Date;
  createdBy: string;
}

interface ExtensionConfig {
  // 类型特定的配置
  [key: string]: any;
}

interface AgentSkillConfig {
  extensionPointId: string;
  agentType: string;
  skillName: string;
  skillDescription: string;
  parameters: Record<string, any>;
  trigger: 'before-node' | 'after-node' | 'inline';
  compositionMode: 'sequential' | 'parallel' | 'conditional';
}
```

### 成功标准
- [x] 可以创建不同类型的扩展点
- [x] 可以将扩展点绑定到节点
- [x] 可以配置扩展点参数
- [x] AgentSkill 配置完整
- [x] 扩展点保存时间 <200ms

### 依赖
- Feature 1 (workflow-visualization-core): 需要工作流节点定义

### 输出产物
- `src/types/extension-point.ts`
- `src/data/extension-point-types.ts`
- `src/components/extension/ExtensionPointConfig.tsx`
- `src/components/extension/AgentSkillConfig.tsx`
- `src/stores/extension-point-store.ts`
- `src/lib/extension/extension-point-manager.ts`

---

## Feature 6: 扩展点可视化展示 (extension-point-visualization)

### 目标
在工作流画布上可视化展示扩展点，让用户直观看到自定义逻辑的注入位置。

### 核心功能
- **节点徽章**
  - 显示扩展点数量徽章
  - 不同状态的颜色区分
- **图标展示**
  - 在节点或连线上显示扩展点图标
  - 不同扩展点类型的图标区分
  - AgentSkill 显示代理类型图标
- **交互功能**
  - 双击节点打开扩展点列表
  - 从列表中编辑扩展点
  - 拖拽调整扩展点执行顺序
  - 从面板启用/禁用扩展点
- **扩展点面板**
  - 显示每个扩展点的详细信息
  - 直接编辑扩展点配置
  - 显示执行顺序

### 技术组件
- **Badge 组件**: 扩展点数量徽章
- **Icon 组件**: 扩展点类型图标
- **Extension Point Panel 组件**: 扩展点列表面板
- **Extension Point Item 组件**: 扩展点项展示
- **Draggable List 组件**: 可拖拽列表

### 成功标准
- [x] 能看到哪些节点有扩展点
- [x] 能查看和编辑扩展点详情
- [x] 能调整扩展点的执行顺序
- [x] 图标清晰可辨
- [x] 徽章显示准确

### 依赖
- Feature 5 (extension-point-model): 需要扩展点数据模型

### 输出产物
- `src/components/extension/ExtensionPointBadge.tsx`
- `src/components/extension/ExtensionPointIcon.tsx`
- `src/components/extension/ExtensionPointPanel.tsx`
- `src/components/extension/ExtensionPointItem.tsx`
- `src/components/extension/DraggableExtensionList.tsx`

---

## Feature 7: AgentSkill 执行器 (agent-skill-executor)

### 目标
实现 AgentSkill 的执行能力，集成 AI 代理，支持扩展点的实际运行。

### 核心功能
- **AI 代理集成**
  - 支持 Claude CLI
  - 支持 Gemini CLI
  - 支持 Cursor CLI
  - 可扩展的代理接口
- **执行逻辑**
  - 根据 agentType 选择代理
  - 传递参数和上下文
  - 执行 skill
  - 返回结果
- **错误处理**
  - 执行超时
  - 重试机制
  - 错误日志
- **结果处理**
  - 解析执行结果
  - 返回结构化数据
  - 记录执行元数据

### 技术组件
- **代理抽象接口**: AI 代理统一接口
- **代理适配器**: Claude、Gemini、Cursor 等代理适配器
- **AgentSkill 执行器**: AgentSkill 执行引擎
- **上下文管理器**: 执行上下文管理

### 数据模型
```typescript
interface AIAgent {
  type: string;
  name: string;
  cliCommand: string;
  executeSkill(skillName: string, parameters: any, context: ExecutionContext): Promise<SkillResult>;
}

interface ExecutionContext {
  node: WorkflowNode;
  workspace: Workspace;
  previousResults?: any[];
}

interface SkillResult {
  success: boolean;
  data?: any;
  error?: string;
  metadata: {
    agentType: string;
    skillName: string;
    executionTime: number;
    timestamp: Date;
  };
}
```

### 执行流程
```
1. 获取扩展点配置
   ↓
2. 根据 agentType 选择代理
   ↓
3. 准备执行上下文
   ↓
4. 执行 skill（带超时和重试）
   ↓
5. 解析结果
   ↓
6. 返回 SkillResult
```

### 成功标准
- [x] 能调用指定的 AI 代理
- [x] 能执行配置的 skill
- [x] 能展示执行结果
- [x] 错误处理完善
- [x] 超时和重试正常工作

### 依赖
- Feature 5 (extension-point-model): 需要扩展点配置

### 输出产物
- `src/engines/agent/agent-interface.ts`
- `src/engines/agent/claude-adapter.ts`
- `src/engines/agent/gemini-adapter.ts`
- `src/engines/agent/cursor-adapter.ts`
- `src/engines/agent/agent-skill-executor.ts`
- `src/lib/execution/context-manager.ts`

---

## Feature 8: 工作流执行引擎 (workflow-execution-engine)

### 目标
实现 SDD 命令的执行和状态管理，支持扩展点的执行，实现完整的工作流运行。

### 核心功能
- **命令执行**
  - 调用 SDD 命令（/speckit.specify、/speckit.plan 等）
  - 节点命令解析和执行
- **状态管理**
  - 节点执行状态（pending, running, completed, failed）
  - 工作流整体进度
  - 执行日志
- **扩展点执行**
  - 执行前置钩子（Pre-hook）
  - 执行节点命令
  - 执行后置钩子（Post-hook）
  - 处理扩展点组合（sequential, parallel, conditional）
- **错误处理**
  - 节点执行错误捕获
  - 扩展点执行错误捕获
  - 重试和回滚机制
- **前置条件验证**
  - 检查输入文档是否存在
  - 验证依赖节点是否完成
  - 检查必需的配置

### 技术组件
- **工作流执行引擎**: 核心执行引擎
- **命令执行器**: SDD 命令执行
- **状态管理器**: 执行状态管理
- **日志记录器**: 执行日志
- **前置条件验证器**: 验证前置条件

### 数据模型
```typescript
enum NodeStatus {
  PENDING = 'pending',
  RUNNING = 'running',
  COMPLETED = 'completed',
  FAILED = 'failed',
  SKIPPED = 'skipped'
}

interface ExecutionState {
  workflowId: string;
  currentNodeId: string | null;
  nodeStatus: Map<string, NodeStatus>;
  executionLogs: ExecutionLog[];
  progress: number;
  startTime: Date;
  endTime?: Date;
}

interface ExecutionLog {
  nodeId: string;
  type: 'node' | 'extension' | 'error' | 'info';
  message: string;
  timestamp: Date;
  data?: any;
}
```

### 执行流程
```
1. 验证前置条件
   ↓
2. 获取下一个待执行节点
   ↓
3. 执行 Pre-Hook 扩展点（按顺序）
   ↓ (全部成功)
4. 执行节点命令
   ↓ (成功)
5. 执行 Post-Hook 扩展点（按顺序）
   ↓ (全部成功)
6. 更新节点状态为 completed
   ↓
7. 返回到步骤 2，直到所有节点完成
```

### 成功标准
- [x] 能按顺序执行工作流节点
- [x] 能执行扩展点
- [x] 能追踪执行进度
- [x] 错误处理完善
- [x] 状态更新实时（<100ms）

### 依赖
- Feature 7 (agent-skill-executor): 需要 AgentSkill 执行能力

### 输出产物
- `src/engines/workflow/workflow-execution-engine.ts`
- `src/engines/workflow/command-executor.ts`
- `src/engines/workflow/state-manager.ts`
- `src/engines/workflow/precondition-validator.ts`
- `src/lib/execution/logger.ts`
- `src/components/execution/ExecutionTracker.tsx`

---

## Feature 9: 工作流验证与高级定制 (workflow-validation-and-advanced-customization)

### 目标
提供工作流验证和高级定制能力，支持高级用户创建自定义节点类型和工作流模板。

### 核心功能
- **工作流验证**
  - 检查孤立的节点
  - 检查循环依赖
  - 检查无效的连接
  - 检查必需的开始和结束节点
  - 验证节点配置的完整性
  - 提供验证错误和建议

- **自定义节点类型**
  - 创建新的节点类型定义
  - 定义节点的默认配置
  - 定义节点的输入/输出端口
  - 节点类型导出和导入

- **工作流模板**
  - 保存工作流为模板
  - 从模板创建新工作流
  - 模板分类和管理
  - 模板预览

- **高级编辑功能**
  - 批量编辑节点属性
  - 查找和替换节点
  - 工作流快照和版本对比
  - 工作流导入/导出（支持多种格式）

### 技术组件
- **工作流验证器**: 工作流合法性验证
- **节点类型管理器**: 自定义节点类型管理
- **工作流模板管理器**: 工作流模板管理
- **批量编辑器**: 批量编辑功能
- **导入导出管理器**: 多格式导入导出

### 数据模型
```typescript
interface WorkflowValidationResult {
  isValid: boolean;
  errors: ValidationError[];
  warnings: ValidationWarning[];
  suggestions: ValidationSuggestion[];
}

interface ValidationError {
  id: string;
  type: 'isolated-node' | 'circular-dependency' | 'invalid-connection' | 
         'missing-start-node' | 'missing-end-node' | 'incomplete-config';
  nodeId?: string;
  connectionId?: string;
  message: string;
  severity: 'error' | 'warning';
}

interface CustomNodeType {
  id: string;
  name: string;
  category: string;
  defaultConfig: NodeConfig;
  inputPorts: Port[];
  outputPorts: Port[];
  icon?: string;
}

interface WorkflowTemplate {
  id: string;
  name: string;
  description: string;
  category: string;
  workflow: WorkflowDefinition;
  preview?: string;
  tags: string[];
  createdAt: Date;
}
```

### 成功标准
- [x] 能检测并报告工作流错误
- [x] 能提供验证建议
- [x] 可以创建自定义节点类型
- [x] 可以保存和使用工作流模板
- [x] 批量编辑功能正常工作
- [x] 导入导出支持多种格式

### 依赖
- Feature 1 (workflow-visualization-and-editing): 需要工作流编辑功能
- Feature 8 (workflow-execution-engine): 需要执行引擎验证工作流

### 输出产物
- `src/lib/workflow/workflow-validator.ts`
- `src/components/workflow/ValidationPanel.tsx`
- `src/lib/workflow/node-type-manager.ts`
- `src/components/workflow/NodeTypeEditor.tsx`
- `src/lib/workflow/template-manager.ts`
- `src/components/workflow/TemplateLibrary.tsx`
- `src/lib/workflow/batch-editor.ts`
- `src/lib/workflow/import-export-manager.ts`

---

## Feature 10: 用户角色和权限管理 (user-roles-and-permissions)

### 目标
实现用户角色和访问控制，区分高级用户和普通用户的权限。

### 核心功能
- **用户角色**
  - 高级用户：创建和编辑工作区、定制模板和工作流、发布工作区
  - 普通用户：查看和使用工作区（只读）、执行工作流
- **权限控制**
  - 基于角色的访问控制
  - 工作区级别的权限
- **发布管理**
  - 高级用户可以发布工作区
  - 普通用户可以选择发布的工作区
- **版本控制**
  - 工作区版本管理（基于 Git）
  - 版本历史查看
  - 回滚到历史版本

### 技术组件
- **用户认证**: 用户登录和认证
- **权限管理器**: 角色和权限管理
- **发布管理器**: 工作区发布管理
- **版本管理器**: 基于 Git 的版本管理

### 数据模型
```typescript
interface User {
  id: string;
  name: string;
  email: string;
  role: 'advanced' | 'standard';
  createdAt: Date;
}

interface WorkspacePermission {
  workspaceId: string;
  userId: string;
  permission: 'owner' | 'editor' | 'viewer';
}

interface WorkspaceVersion {
  workspaceId: string;
  version: string;
  commitHash: string;
  author: string;
  createdAt: Date;
  changes: string;
}
```

### 成功标准
- [x] 能区分用户角色
- [x] 高级用户有编辑权限，普通用户只读
- [x] 工作区可以被发布和版本控制

### 依赖
- Feature 9 (advanced-workflow-customization): 需要工作流定制功能

### 输出产物
- `src/components/auth/UserProfile.tsx`
- `src/lib/auth/permission-manager.ts`
- `src/lib/workspace/publishing-manager.ts`
- `src/lib/workspace/version-manager.ts`
- `src/stores/user-store.ts`

---

## Feature 11: 团队协作与共享 (team-collaboration)

### 目标
支持团队协作和共享工作区，让团队成员可以使用统一的工作流。

### 核心功能
- **工作区共享**
  - 工作区共享机制
  - 团队工作区
  - 共享链接生成
- **协作编辑**
  - 实时同步（可选，未来增强）
  - 评论和讨论
  - 变更通知
- **团队管理**
  - 团队成员管理
  - 团队权限设置

### 技术组件
- **共享管理器**: 工作区共享管理
- **团队管理器**: 团队管理
- **评论系统**: 评论和讨论
- **通知系统**: 变更通知

### 数据模型
```typescript
interface Team {
  id: string;
  name: string;
  description: string;
  ownerId: string;
  members: TeamMember[];
  createdAt: Date;
}

interface TeamMember {
  userId: string;
  role: 'owner' | 'admin' | 'member';
  joinedAt: Date;
}

interface Comment {
  id: string;
  workspaceId: string;
  userId: string;
  content: string;
  createdAt: Date;
  replies: Comment[];
}

interface Notification {
  id: string;
  userId: string;
  type: 'update' | 'comment' | 'shared';
  message: string;
  read: boolean;
  createdAt: Date;
}
```

### 成功标准
- [x] 团队成员可以共享工作区
- [x] 可以协作编辑
- [x] 可以讨论和评论
- [x] 能接收变更通知

### 依赖
- Feature 10 (user-roles-and-permissions): 需要用户角色和权限管理

### 输出产物
- `src/components/team/TeamManagement.tsx`
- `src/components/workspace/WorkspaceSharing.tsx`
- `src/components/collaboration/CommentSystem.tsx`
- `src/components/collaboration/NotificationPanel.tsx`
- `src/lib/collaboration/team-manager.ts`
- `src/lib/collaboration/sharing-manager.ts`

---

## Feature 12: 模板智能合并 (template-smart-merge)

### 目标
实现模板的智能合并和冲突解决，简化用户同步模板时的操作。

### 核心功能
- **高级 Diff 算法**
  - 行级 Diff
  - 语义 Diff（可选，未来增强）
- **自动合并策略**
  - 无冲突区域自动合并
  - 冲突区域标记
- **可视化合并工具**
  - 三方 Diff 视图：
    - Base version
    - Custom version
    - New version
  - 冲突区域高亮
  - 并排或统一视图
- **合并辅助**
  - 合并建议
  - 预览合并结果
  - 回滚合并

### 技术组件
- **高级 Diff 引擎**: 增强的 Diff 算法
- **合并引擎**: 智能合并逻辑
- **合并可视化工具**: 可视化合并界面
- **合并预览器**: 合并结果预览

### 数据模型
```typescript
interface MergeResult {
  success: boolean;
  mergedContent: string;
  conflicts: MergeConflict[];
  autoMergedCount: number;
  manualReviewCount: number;
}

interface MergeConflict {
  id: string;
  lineStart: number;
  lineEnd: number;
  baseContent: string;
  customContent: string;
  newContent: string;
  suggestions: MergeSuggestion[];
}

interface MergeSuggestion {
  type: 'keep-custom' | 'adopt-new' | 'manual';
  recommendation: string;
  confidence: number;
}
```

### 合并流程
```
1. 读取三个版本（Base, Custom, New）
   ↓
2. 执行三方 Diff
   ↓
3. 识别冲突区域
   ↓
4. 对无冲突区域执行自动合并
   ↓
5. 对冲突区域生成合并建议
   ↓
6. 用户审查合并结果
   ↓
7. 应用合并（用户确认后）
```

### 成功标准
- [x] 能自动合并无冲突的更改
- [x] 能清晰展示冲突
- [x] 能帮助用户解决冲突
- [x] 合并建议准确
- [x] 预览功能正常

### 依赖
- Feature 4 (template-sync): 需要基础的模板同步能力

### 输出产物
- `src/lib/merge/advanced-diff-engine.ts`
- `src/lib/merge/merge-engine.ts`
- `src/lib/merge/merge-suggestion-generator.ts`
- `src/components/merge/ThreeWayDiffViewer.tsx`
- `src/components/merge/MergeConflictPanel.tsx`
- `src/components/merge/MergePreview.tsx`

---

## 依赖关系图

```
                    F1: 工作流可视化与编辑 (核心可视化编辑功能)
                            │
                            ├────────────────────────────┐
                            │                            │
                    F5: 扩展点数据模型与配置       F8: 工作流执行引擎
                            │                            │
                    F6: 扩展点可视化展示              │
                            │                            │
                    F7: AgentSkill 执行器 ────────────┘
                            │
                            ▼
                    F9: 工作流验证与高级定制 (验证、自定义节点类型、模板)
                            │
                            ▼
                    F10: 用户角色和权限管理
                            │
                            ▼
                    F11: 团队协作与共享

F2: 工作区管理系统
        │
        ├────────────────────────────┐
        │                            │
F3: 模板系统基础              (基础数据存储)
        │
        ▼
F4: 模板同步系统
        │
        ▼
F12: 模板智能合并
```

## Feature 优先级矩阵

| Feature | 业务价值 | 技术复杂度 | 依赖数量 | 优先级 |
|---------|----------|-----------|---------|--------|
| F1: 工作流可视化与编辑 | 高 | 高 | 0 | P0 (核心) |
| F2: 工作区管理系统 | 高 | 低 | 0 | P0 (核心) |
| F3: 模板系统基础 | 高 | 中 | 1 | P0 |
| F5: 扩展点数据模型与配置 | 高 | 高 | 1 | P0 |
| F7: AgentSkill 执行器 | 高 | 高 | 1 | P0 |
| F8: 工作流执行引擎 | 高 | 高 | 1 | P0 |
| F4: 模板同步系统 | 中 | 中 | 1 | P1 |
| F6: 扩展点可视化展示 | 中 | 中 | 1 | P1 |
| F9: 工作流验证与高级定制 | 中 | 高 | 2 | P1 |
| F10: 用户角色和权限管理 | 中 | 中 | 1 | P2 |
| F11: 团队协作与共享 | 低 | 高 | 1 | P2 |
| F12: 模板智能合并 | 低 | 高 | 1 | P3 |

**说明**：
- F1 现在是核心功能，包含完整的可视化编辑能力，复杂度从中提升到高
- F9 重新定位为高级定制功能（验证、自定义节点类型、模板），依赖 F1 和 F8
- F10 的依赖保持不变，依赖 F9
