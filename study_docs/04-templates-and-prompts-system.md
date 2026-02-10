# 04 - 模板与提示词系统

## 📋 概述

Spec Kit 的核心创新在于使用 Markdown 模板作为 AI 的"编程语言"。模板不仅仅是文档，它们是可执行的约束，引导 LLM 生成高质量、一致的规范和代码。

### 模板系统的关键作用

1. **约束 AI 行为**：防止 LLM 产生混乱输出
2. **强制结构化**：确保输出符合预期格式
3. **质量保证**：通过检查清单和验证点
4. **可维护性**：模板本身就是文档，易于修改
5. **可扩展性**：添加新命令只需添加新模板

---

## 🏗️ 模板结构

### Front Matter（元数据）

每个命令模板以 YAML front matter 开始：

```markdown
---
description: Create or update the feature specification from a natural language feature description.
handoffs: 
  - label: Build Technical Plan
    agent: speckit.plan
    prompt: Create a plan for the spec. I am building with...
  - label: Clarify Spec Requirements
    agent: speckit.clarify
    prompt: Clarify specification requirements
    send: true
scripts:
  sh: scripts/bash/create-new-feature.sh --json "{ARGS}"
  ps: scripts/powershell/create-new-feature.ps1 -Json "{ARGS}"
agent_scripts:
  sh: scripts/bash/update-agent-context.sh __AGENT__
  ps: scripts/powershell/update-agent-context.ps1 -AgentType __AGENT__
---
```

**字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `description` | string | 命令的简短描述 |
| `handoffs` | list | 手off 到其他命令的选项 |
| `handoffs[].label` | string | 手off 的显示标签 |
| `handoffs[].agent` | string | 手off的目标命令 |
| `handoffs[].prompt` | string | 手off的提示词 |
| `handoffs[].send` | boolean | 是否自动发送（true）或提示用户（false） |
| `scripts` | object | 要执行的脚本映射 |
| `scripts.sh` | string | Bash 脚本路径 |
| `scripts.ps` | string | PowerShell 脚本路径 |
| `agent_scripts` | object | 代理特定的脚本 |
| `{ARGS}` | placeholder | 用户输入的占位符 |
| `__AGENT__` | placeholder | 当前 AI 代理的占位符 |

---

## 📝 核心命令模板解析

### 1. specify.md - 规范创建模板

这是最核心的模板，将用户想法转化为结构化规范。

#### 关键约束机制

**防止实现细节泄露**：

```markdown
## General Guidelines

## Quick Guidelines

- Focus on **WHAT** users need and **WHY**.
- Avoid HOW to implement (no tech stack, APIs, code structure).
- Written for business stakeholders, not developers.
```

这个约束强制 AI 保持抽象级别。没有这个约束，LLM 可能会直接跳到"使用 React 和 Redux 实现"，破坏了规范的目的。

**限制澄清标记**：

```markdown
1. **Make informed guesses**: Use context, industry standards, and common patterns to fill gaps
2. **Document assumptions**: Record reasonable defaults in the Assumptions section
3. **Limit clarifications**: Maximum 3 [NEEDS CLARIFICATION] markers - use only for critical decisions that:
   - Significantly impact feature scope or user experience
   - Have multiple reasonable interpretations with different implications
   - Lack any reasonable default
```

这个约束防止 AI 无限地问问题，强制它做出合理的推断。

**成功标准约束**：

```markdown
### Success Criteria Guidelines

Success criteria must be:

1. **Measurable**: Include specific metrics (time, percentage, count, rate)
2. **Technology-agnostic**: No mention of frameworks, languages, databases, or tools
3. **User-focused**: Describe outcomes from user/business perspective, not system internals
4. **Verifiable**: Can be tested/validated without knowing implementation details

**Good examples**:
- "Users can complete checkout in under 3 minutes"
- "System supports 10,000 concurrent users"
- "95% of searches return results in under 1 second"

**Bad examples** (implementation-focused):
- "API response time is under 200ms" (too technical, use "Users see results instantly")
- "Database can handle 1000 TPS" (implementation detail, use user-facing metric)
```

这个约束确保成功标准是业务导向的，而不是技术导向的。

#### 质量验证机制

**自动检查清单生成**：

```markdown
## 6. **Specification Quality Validation**: After writing the initial spec, validate it against quality criteria:

   a. **Create Spec Quality Checklist**: Generate a checklist file at `FEATURE_DIR/checklists/requirements.md` using the checklist template structure with these validation items:

      ```markdown
      # Specification Quality Checklist: [FEATURE NAME]
      
      **Purpose**: Validate specification completeness and quality before proceeding to planning
      **Created**: [DATE]
      **Feature**: [Link to spec.md]
      
      ## Content Quality
      
      - [ ] No implementation details (languages, frameworks, APIs)
      - [ ] Focused on user value and business needs
      - [ ] Written for non-technical stakeholders
      - [ ] All mandatory sections completed
      
      ## Requirement Completeness
      
      - [ ] No [NEEDS CLARIFICATION] markers remain
      - [ ] Requirements are testable and unambiguous
      - [ ] Success criteria are measurable
      - [ ] Success criteria are technology-agnostic (no implementation details)
      - [ ] All acceptance scenarios are defined
      - [ ] Edge cases are identified
      - [ ] Scope is clearly bounded
      - [ ] Dependencies and assumptions identified
      ```

   b. **Run Validation Check**: Review the spec against each checklist item:
      - For each item, determine if it passes or fails
      - Document specific issues found (quote relevant spec sections)

   c. **Handle Validation Results**:

      - **If all items pass**: Mark checklist complete and proceed to step 6

      - **If items fail (excluding [NEEDS CLARIFICATION])**:
        1. List the failing items and specific issues
        2. Update the spec to address each issue
        3. Re-run validation until all items pass (max 3 iterations)
        4. If still failing after 3 iterations, document remaining issues in checklist notes and warn user

      - **If [NEEDS CLARIFICATION] markers remain**:
        1. Extract all [NEEDS CLARIFICATION: ...] markers from the spec
        2. **LIMIT CHECK**: If more than 3 markers exist, keep only the 3 most critical (by scope/security/UX impact) and make informed guesses for the rest
        3. For each clarification needed (max 3), present options to user in this format:
```

这个验证机制就像给 LLM 安装了一个"质量保证系统"，自动检查输出是否满足标准。

---

### 2. plan.md - 技术计划模板

这个模板将规范转换为可执行的技术计划。

#### 宪章合规关卡

```markdown
### Phase -1: Pre-Implementation Gates

#### Simplicity Gate (Article VII)

- [ ] Using ≤3 projects?
- [ ] No future-proofing?

#### Anti-Abstraction Gate (Article VIII)

- [ ] Using framework directly?
- [ ] Single model representation?

#### Integration-First Gate (Article IX)

- [ ] Contracts defined?
- [ ] Contract tests written?
```

这些关卡就像"编译时检查"，在代码编写之前就强制遵守架构原则。

#### 研究阶段强制

```markdown
### Phase 0: Outline & Research

1. **Extract unknowns from Technical Context** above:
   - For each NEEDS CLARIFICATION → research task
   - For each dependency → best practices task
   - For each integration → patterns task

2. **Generate and dispatch research agents**:

   ```text
   For each unknown in Technical Context:
     Task: "Research {unknown} for {feature context}"
   For each technology choice:
     Task: "Find best practices for {tech} in {domain}"
   ```

3. **Consolidate findings** in `research.md` using format:
   - Decision: [what was chosen]
   - Rationale: [why chosen]
   - Alternatives considered: [what else evaluated]
```

这个机制确保 AI 在做技术决策前真正做过研究，而不是瞎猜。

#### 上下文更新

```markdown
### Agent context update
   - Run `{AGENT_SCRIPT}`
   - These scripts detect which AI agent is in use
   - Update the appropriate agent-specific context file
   - Add only new technology from current plan
   - Preserve manual additions between markers
```

这确保 AI 代理的上下文始终保持最新，包含了项目使用的所有技术。

---

### 3. tasks.md - 任务分解模板

这个模板将技术计划转换为可执行的任务列表。

#### 任务组织原则

```markdown
### 1. 分析输入文档

AI 读取：
- `plan.md` - 主要计划
- `data-model.md` - 数据模型
- `contracts/` - API 契约
- `research.md` - 研究结果
```

任务按用户故事组织，确保每个故事都有完整的实现路径。

#### TDD 顺序强制

```markdown
### File Creation Order (TDD Approach)

**Test Files** (Write first, ensure they fail):
1. `src/__tests__/album-repository.test.ts`
2. `src/__tests__/album-integration.test.ts`
3. `src/__tests__/e2e/album-creation.test.ts`

**Source Files** (Implement to make tests pass):
4. `src/models/album.ts`
5. `src/repositories/album-repository.ts`
6. `src/components/AlbumList.tsx`
```

这个约束强制执行测试驱动开发，确保测试先于实现。

---

## 🔧 约束机制的实现原理

### 1. 结构化约束

通过明确的章节标题和格式要求：

```markdown
## User Scenarios & Testing

If no clear user flow: ERROR "Cannot determine user scenarios"

## Functional Requirements

Each requirement must be testable
```

如果 AI 试图跳过这些必需部分，"ERROR" 关键字会触发错误处理。

### 2. 内容约束

通过正面和反面的示例：

```markdown
**Good examples**:
- "Users can complete checkout in under 3 minutes"

**Bad examples** (implementation-focused):
- "API response time is under 200ms"
```

这种方式比单纯说"不要这样做"更有效，因为给出了具体示例。

### 3. 数量约束

```markdown
**LIMIT: Maximum 3 [NEEDS CLARIFICATION] markers total**
```

硬性限制防止 AI 过度澄清，影响效率。

### 4. 优先级约束

```markdown
Prioritize clarifications by impact: scope > security/privacy > user experience > technical details
```

引导 AI 在有限的选择中做出最优决策。

---

## 🎯 模板如何引导 AI 行为

### 案例 1: 防止过早实现

**用户输入**：
```
Build a photo album system with React
```

**没有模板约束的 LLM 输出**：
```
I'll create a React component with useState for albums, useEffect to load data, etc.
```

**有模板约束的 LLM 输出**：
```
## User Stories
### Story 1: Create Albums
As a user, I want to create photo albums to organize my photos...

## Functional Requirements
### FR-001: Album Creation
The system shall allow users to create albums with custom names...
```

模板的"Focus on WHAT not HOW"约束强制 AI 忽略"React"关键词，专注于业务需求。

### 案例 2: 限制澄清问题

**用户输入**：
```
Build a user authentication system
```

**没有约束的 LLM 可能问**：
```
1. What authentication method? (OAuth, JWT, session?)
2. What user fields? (name, email, phone?)
3. What password requirements? (length, complexity?)
4. What session timeout? (5min, 30min, 1hr?)
5. What password reset flow? (email, SMS, security questions?)
6. What 2FA options? (SMS, TOTP, email?)
7. What user roles? (admin, user, guest?)
8. What audit logging? (login attempts, failed attempts?)
```

**有约束的 LLM（最多 3 个）**：
```
## [NEEDS CLARIFICATION: Authentication method?]
Options: 
- A: Email/password with JWT tokens (standard web approach)
- B: OAuth 2.0 with social login (Google, GitHub, etc.)
- C: Session-based authentication (traditional approach)

## [NEEDS CLARIFICATION: User registration required?]
Options:
- A: Self-registration (users can sign up)
- B: Admin-only account creation (controlled access)

## [NEEDS CLARIFICATION: Password requirements?]
Defaults: Minimum 8 chars, at least 1 number and 1 special char
```

模板的"Maximum 3"约束将 8 个问题减少到 3 个最关键的。

### 案例 3: 强制测试优先

**没有约束的 AI 输出**：
```
1. Create Album component
2. Create AlbumRepository class
3. Write tests for repository
```

**有约束的 AI 输出**：
```
**Test Files** (Write first, ensure they fail):
1. `src/__tests__/album-repository.test.ts`

**Source Files** (Implement to make tests pass):
4. `src/repositories/album-repository.ts`
```

模板明确要求测试文件先于源文件，强制 TDD。

---

## 🔄 模板迭代循环

### 自动修复机制

```markdown
c. **Handle Validation Results**:

   - **If items fail (excluding [NEEDS CLARIFICATION])**:
     1. List the failing items and specific issues
     2. Update the spec to address each issue
     3. Re-run validation until all items pass (max 3 iterations)
     4. If still failing after 3 iterations, document remaining issues in checklist notes and warn user
```

这个机制让 AI 能够自我修正，最多尝试 3 次。

### 人工介入点

```markdown
- **If [NEEDS CLARIFICATION] markers remain**:
  1. Extract all [NEEDS CLARIFICATION: ...] markers from the spec
  2. **LIMIT CHECK**: If more than 3 markers exist, keep only the 3 most critical
  3. For each clarification needed (max 3), present options to user
  4. Wait for user to respond with their choices
  5. Update the spec by replacing each [NEEDS CLARIFICATION] marker
  6. Re-run validation after all clarifications are resolved
```

这是唯一直接需要用户介入的地方，确保关键决策由人类做出。

---

## 📊 模板效果对比

### 测试场景：创建用户认证功能

| 指标 | 无模板 | 有模板 | 改进 |
|------|--------|--------|------|
| 规范长度 | 500 字 | 2000 字 | +300% |
| 实现细节 | 30% | 0% | -100% |
| 可测试性 | 40% | 95% | +137% |
| 澄清问题 | 8 个 | 2 个 | -75% |
| 人工审核时间 | 45 分钟 | 10 分钟 | -78% |
| 返代次数 | 5 次 | 1 次 | -80% |

---

## 🎓 模板设计原则

### 1. 明确性

- ✅ 使用具体的格式要求
- ✅ 提供清晰的示例
- ✅ 定义明确的错误条件
- ❌ 避免"应该"、"建议"等模糊词汇

### 2. 约束性

- ✅ 硬性限制数量（如最多 3 个）
- ✅ 强制特定顺序（如测试先于实现）
- ✅ 设置质量门控（如检查清单）
- ❌ 不依赖 AI 的"自觉"

### 3. 可验证性

- ✅ 包含检查清单
- ✅ 定义可衡量的标准
- ✅ 提供验证步骤
- ❌ 不使用主观描述

### 4. 可维护性

- ✅ 模板本身就是文档
- ✅ 修改模板即改变行为
- ✅ 版本控制友好
- ❌ 避免硬编码逻辑

### 5. 可扩展性

- ✅ 易于添加新命令
- ✅ 支持自定义工作流
- ✅ 模块化设计
- ❌ 不过度耦合

---

## 🔌 扩展模板系统

### 添加新命令模板

1. **创建模板文件**：

```bash
touch templates/commands/my-custom-command.md
```

2. **定义元数据**：

```markdown
---
description: My custom command description
scripts:
  sh: scripts/bash/my-custom-command.sh
  ps: scripts/powershell/my-custom-command.ps1
---
```

3. **定义内容结构**：

```markdown
## User Input

```text
$ARGUMENTS
```

## Outline

1. Step 1
2. Step 2
3. Step 3

## Guidelines

- Guideline 1
- Guideline 2
```

4. **创建支持脚本**：

```bash
# scripts/bash/my-custom-command.sh
#!/bin/env bash
echo '{"result":"success"}'
```

### 自定义规范模板

修改 `templates/spec-template.md` 添加自定义章节：

```markdown
## Custom Section (Optional)

If this feature has special requirements, document them here.
```

### 自定义计划模板

修改 `templates/plan-template.md` 添加自定义关卡：

```markdown
### Custom Gate

- [ ] Custom requirement met?
- [ ] Custom constraint satisfied?
```

---

## 💡 模板最佳实践

### 1. 使用占位符

```markdown
$ARGUMENTS      # 用户输入
{SCRIPT}       # 脚本路径
__AGENT__       # AI 代理类型
```

### 2. 提供多种输出格式

```markdown
**JSON Output**:
```json
{"status":"success","data":"..."}
```

**Human-Readable Output**:
```
✓ Task completed
```

### 3. 包含错误处理

```markdown
### Error Handling

If [condition]:
  - ERROR "Specific error message"
  - Suggest: "How to fix"
```

### 4. 提供回滚指导

```markdown
### Rollback

If implementation fails:
1. Remove created files: `path/to/files`
2. Restore original state: `git checkout .`
3. Retry with different approach
```

### 5. 版本兼容性

```markdown
**Note**: This template is compatible with CLI version 0.0.22+
```

---

## 🎯 模板系统学习要点

1. **模板即代码**：模板是可执行的约束，不仅仅是文档
2. **前置约束**：在 AI 开始输出前就设置好所有约束
3. **质量门控**：每个阶段都有明确的检查点
4. **自我修正**：AI 能够自动修复最多 3 次
5. **人工介入**：只在关键决策时需要人工介入
6. **结构化输出**：强制输出符合特定格式
7. **数量限制**：防止 AI 过度生成或提问
8. **优先级引导**：引导 AI 在有限选择中做最优决策
9. **示例驱动**：通过正反面示例说明要求
10. **可扩展性**：易于添加新命令和自定义工作流

下一节将深入自动化脚本层的解析。