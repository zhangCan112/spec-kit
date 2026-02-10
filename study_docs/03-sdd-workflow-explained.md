# 03 - SDD 工作流实现原理

## 📋 概述

SDD（Spec-Driven Development）工作流是 Spec Kit 的核心方法论，通过结构化的命令将想法转换为可工作的代码。本文档深入解析每个阶段的工作原理。

### 完整工作流图

```
┌─────────────────────────────────────────────────────────────────┐
│                        Constitution 阶段                       │
│                   定义项目的核心开发原则                        │
│                   (/speckit.constitution)                      │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Specification 阶段                       │
│              创建功能规范（定义 WHAT 和 WHY）                   │
│              (/speckit.specify)                               │
│                                                               │
│  1. 解析用户输入                                             │
│  2. 生成分支名和目录                                         │
│  3. 创建规范文档                                              │
│  4. 质量验证                                                │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
                    [可选] Clarification 阶段
                      (/speckit.clarify)
                  明确规范中的模糊需求
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Planning 阶段                            │
│            创建技术实现计划（定义 HOW）                           │
│            (/speckit.plan)                                     │
│                                                               │
│  Phase 0: 研究阶段                                            │
│    - 识别技术未知数                                            │
│    - 生成研究文档                                              │
│                                                               │
│  Phase 1: 设计阶段                                            │
│    - 生成数据模型                                              │
│    - 创建 API 契约                                             │
│    - 生成快速开始指南                                          │
│                                                               │
│  Phase 2: 更新上下文                                          │
│    - 更新 AI 代理上下文                                         │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Tasks 阶段                               │
│            将计划分解为可执行的任务列表                           │
│            (/speckit.tasks)                                     │
│                                                               │
│  1. 分析计划文档                                               │
│  2. 生成任务列表                                              │
│  3. 标记依赖关系和并行执行                                     │
│  4. 组织用户故事                                              │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
            [可选] Analyze 和 Checklist 阶段
      (/speckit.analyze, /speckit.checklist)
    交叉一致性分析和质量检查
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Implementation 阶段                         │
│            执行所有任务，按照计划构建功能                         │
│            (/speckit.implement)                                  │
│                                                               │
│  1. 验证前置条件                                              │
│  2. 解析任务列表                                              │
│  3. 执行任务（按依赖顺序）                                      │
│  4. 测试和验证                                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔵 阶段 1: Constitution（项目宪章）

### 目的

定义项目的核心开发原则，确保所有后续工作遵循一致的架构指导。

### 执行流程

```bash
/speckit.constitution Create principles focused on code quality, testing standards, user experience consistency, and performance requirements
```

### 工作原理

1. **加载模板**：`templates/constitution-template.md`
2. **解析用户输入**：理解用户想要的原则
3. **填充模板**：根据输入生成宪章内容
4. **保存文档**：写入 `.specify/memory/constitution.md`

### 宪章的结构

```markdown
# Project Constitution

## Article I: Library-First Principle
## Article II: CLI Interface Mandate
## Article III: Test-First Imperative
## Article IV: Observability Principle
## Article V: Minimalism Principle
## Article VI: Atomicity Principle
## Article VII: Simplicity Principle
## Article VIII: Anti-Abstraction Principle
## Article IX: Integration-First Testing
```

### 关键作用

- **约束 AI 行为**：在后续的所有阶段，AI 必须遵守这些原则
- **保证架构一致性**：无论由哪个 AI 模型生成，都遵循相同原则
- **质量门控**：在 plan 阶段有明确的"关卡"检查是否违反宪章

---

## 🟢 阶段 2: Specification（功能规范）

### 目的

将模糊的想法转化为精确、完整、可测试的功能规范。

### 执行流程

```bash
/speckit.specify Build a photo album organizer with drag-and-drop albums
```

### 详细步骤

#### 步骤 1: 生成短名称

AI 分析描述，提取 2-4 个关键词：

```
输入: "Build a photo album organizer with drag-and-drop albums"
输出: "photo-album-organizer"
```

#### 步骤 2: 计算特性号

脚本 `create-new-feature.sh` 执行：

```bash
# 检查远程分支
git ls-remote --heads origin | grep -E 'refs/heads/[0-9]+-photo-album-organizer$'

# 检查本地分支
git branch | grep -E '^[* ]*[0-9]+-photo-album-organizer$'

# 检查 specs 目录
ls specs/ | grep -E '^[0-9]+-photo-album-organizer$'

# 找到最大号 +1
# 例如找到最大号是 002，则新特性号为 003
```

#### 步骤 3: 创建 Git 分支

```bash
git checkout -b 003-photo-album-organizer
```

#### 步骤 4: 创建目录结构

```
specs/003-photo-album-organizer/
├── spec.md          # 功能规范
└── checklists/       # 检查清单
    └── requirements.md
```

#### 步骤 5: AI 生成规范

AI 按照 `spec-template.md` 的结构填充内容：

```markdown
# Feature Specification: Photo Album Organizer

## Overview
Brief description of what we're building...

## User Stories
### Story 1: Create Albums
As a user, I want to create photo albums...

### Story 2: Organize Photos
As a user, I want to drag and drop photos...

## Functional Requirements
### FR-001: Album Creation
The system shall allow users to create albums...

### FR-002: Photo Organization
The system shall support drag-and-drop...

## Success Criteria
- Users can create an album in under 30 seconds
- Drag-and-drop works smoothly with 50+ photos
- Album layout persists across sessions
```

#### 步骤 6: 质量验证

AI 自动创建并运行检查清单：

```markdown
# Specification Quality Checklist

## Content Quality
- [ ] No implementation details (languages, frameworks, APIs)
- [ ] Focused on user value and business needs
- [ ] Written for non-technical stakeholders

## Requirement Completeness
- [ ] No [NEEDS CLARIFICATION] markers remain
- [ ] Requirements are testable and unambiguous
- [ ] Success criteria are measurable
```

**处理验证结果**：

- ✅ **全部通过**：继续到 plan 阶段
- ❌ **部分失败**：AI 自动修复，最多 3 次迭代
- ⚠️ **有 [NEEDS CLARIFICATION]**：AI 提问，等待用户回答

### 澄清问题的格式

如果有需要澄清的地方，AI 会这样问：

```markdown
## Question 1: Album Storage

**Context**: "The system shall allow users to create albums..."

**What we need to know**: Where should album data be stored? Local database? Cloud? Both?

**Suggested Answers**:

| Option | Answer | Implications |
|--------|--------|--------------|
| A      | Local SQLite database | Simple, no network, limited scalability |
| B      | Cloud storage (S3) | Scalable, requires auth, works across devices |
| C      | Local + Cloud sync | Best of both, complex implementation |
| Custom | Provide your own answer | Explain your preferred approach |

**Your choice**: _[Wait for user response]_
```

### 输出成果

```
✓ Branch: 003-photo-album-organizer
✓ Spec: specs/003-photo-album-organizer/spec.md
✓ Checklist: specs/003-photo-album-organizer/checklists/requirements.md
✓ Ready for: /speckit.plan
```

---

## 🟡 阶段 3: Clarification（需求澄清）

### 目的

在进入技术计划之前，明确规范中模糊的需求，减少后期返工。

### 何时使用

- 规范中有 `[NEEDS CLARIFICATION]` 标记
- 需求描述不够具体
- 有多种实现方式，需要明确选择

### 执行流程

```bash
/speckit.clarify
```

### AI 的工作

1. **扫描规范**：查找所有 `[NEEDS CLARIFICATION: ...]` 标记
2. **生成问题**：为每个模糊点创建具体问题
3. **呈现选项**：提供 2-4 个建议答案及影响分析
4. **等待回答**：收集用户的选择
5. **更新规范**：根据用户回答更新规范文档
6. **重新验证**：运行质量检查清单

### 示例

**规范中的标记**：

```markdown
### FR-003: Photo Metadata
The system shall store photo metadata including [NEEDS CLARIFICATION: which metadata fields?] for each photo.
```

**AI 生成的澄清**：

```markdown
## Clarification: Photo Metadata Fields

**Found in**: FR-003 - Photo Metadata

**Question**: Which metadata fields should be stored for each photo?

**Suggested Answers**:

| Option | Fields | Pros | Cons |
|--------|---------|-------|-------|
| A      | Date taken, File size, Dimensions | Basic, sufficient for most needs | Limited |
| B      | A + Camera model, ISO, Aperture | Good for photographers | Larger storage |
| C      | B + GPS location, EXIF full data | Complete | Privacy concerns |
| Custom | Specify your own fields | Tailored to your needs | - |

**Please select an option (A/B/C/Custom) and provide details if choosing Custom**:
```

### 更新后的规范

```markdown
### FR-003: Photo Metadata
The system shall store photo metadata including:
- Date taken (mandatory)
- File size (mandatory)
- Dimensions (mandatory)
- Camera model (optional)
- ISO setting (optional)
- Aperture setting (optional)

[User selected Option B]
```

---

## 🟠 阶段 4: Planning（技术计划）

### 目的

将功能规范转换为可执行的技术实现计划。

### 执行流程

```bash
/speckit.plan Use Vite with vanilla HTML/CSS/JS, local SQLite database
```

### Phase 0: 研究阶段

#### 步骤 1: 识别技术未知数

AI 扫描规范和计划模板，标记需要研究的内容：

```markdown
## Technical Context

### Framework
- Choice: Vite (vanilla)
- **NEEDS CLARIFICATION**: Which build tools and bundlers?

### Database
- Choice: SQLite
- **NEEDS CLARIFICATION**: How to handle schema migrations?

### State Management
- **NEEDS CLARIFICATION**: What approach? Vanilla JS, signals, or external library?
```

#### 步骤 2: 生成研究任务

对于每个未知数，AI 创建研究任务：

```
Task 1: Research Vite build configuration for vanilla JS projects
Task 2: Research SQLite integration approaches in browser environments
Task 3: Evaluate state management options (Signals vs external libs)
```

#### 步骤 3: 并行研究

AI 并行执行研究（如果支持）：

```markdown
## Research Results

### Vite Build Configuration
**Decision**: Use Vite's default build configuration with custom plugins for optimization
**Rationale**: Default config is sufficient for most use cases; only add plugins when specific needs arise
**Alternatives considered**: Webpack (too complex), Rollup (lower-level)

### SQLite in Browser
**Decision**: Use sql.js (SQLite compiled to WebAssembly)
**Rationale**: Full SQLite compatibility, works offline, persistent via IndexedDB
**Alternatives considered**: LocalStorage (limited), Dexie.js (not SQLite)

### State Management
**Decision**: Use Preact signals (minimal overhead)
**Rationale**: React-like patterns without React, lightweight, good performance
**Alternatives considered**: Vanilla JS state (too manual), Redux (too complex)
```

### Phase 1: 设计阶段

#### 步骤 1: 生成数据模型

根据用户故事和功能需求，创建数据模型：

```markdown
# Data Model

## Entities

### Album
```typescript
interface Album {
  id: string;              // UUID
  name: string;
  description?: string;
  createdAt: Date;
  updatedAt: Date;
  coverPhotoId?: string;   // Reference to Photo
  order: number;          // For sorting albums
}
```

### Photo
```typescript
interface Photo {
  id: string;
  albumId: string;        // Foreign key to Album
  file: File;            // Original file object
  thumbnail: string;     // Data URL or blob URL
  metadata: PhotoMetadata;
  order: number;         // For sorting within album
}
```

### PhotoMetadata
```typescript
interface PhotoMetadata {
  dateTaken?: Date;
  fileSize: number;
  dimensions: { width: number; height: number };
  cameraModel?: string;
  iso?: number;
  aperture?: string;
}
```

## Relationships

- Album (1) ── (N) Photo
- Photo (1) ── (1) PhotoMetadata

## Database Schema

```sql
CREATE TABLE albums (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  description TEXT,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  cover_photo_id TEXT,
  "order" INTEGER NOT NULL,
  FOREIGN KEY (cover_photo_id) REFERENCES photos(id)
);

CREATE TABLE photos (
  id TEXT PRIMARY KEY,
  album_id TEXT NOT NULL,
  file_name TEXT NOT NULL,
  file_type TEXT NOT NULL,
  file_size INTEGER NOT NULL,
  thumbnail_blob BLOB,
  date_taken INTEGER,
  dimensions_width INTEGER,
  dimensions_height INTEGER,
  camera_model TEXT,
  iso INTEGER,
  aperture TEXT,
  "order" INTEGER NOT NULL,
  FOREIGN KEY (album_id) REFERENCES albums(id) ON DELETE CASCADE
);
```

## State Management

```typescript
interface AppState {
  albums: Album[];
  currentAlbum: Album | null;
  photos: Photo[];
  loading: boolean;
  error: string | null;
}
```

### Signals Implementation

```typescript
import { signal, computed } from '@preact/signals';

const albums = signal<Album[]>([]);
const currentAlbum = signal<Album | null>(null);
const currentAlbumPhotos = computed(() => 
  photos.value.filter(p => p.albumId === currentAlbum.value?.id)
);
```
```

#### 步骤 2: 创建 API 契约

虽然这是纯前端应用，但仍需要内部 API 契约：

```markdown
# API Contracts

## Database API

### Albums

#### Create Album
```typescript
interface CreateAlbumRequest {
  name: string;
  description?: string;
}

interface CreateAlbumResponse {
  album: Album;
}
```

#### Get All Albums
```typescript
interface GetAlbumsResponse {
  albums: Album[];
}
```

#### Update Album
```typescript
interface UpdateAlbumRequest {
  id: string;
  name?: string;
  description?: string;
  coverPhotoId?: string;
  order?: number;
}

interface UpdateAlbumResponse {
  album: Album;
}
```

#### Delete Album
```typescript
interface DeleteAlbumRequest {
  id: string;
}

interface DeleteAlbumResponse {
  success: boolean;
}
```

### Photos

#### Upload Photo
```typescript
interface UploadPhotoRequest {
  albumId: string;
  file: File;
  metadata?: Partial<PhotoMetadata>;
}

interface UploadPhotoResponse {
  photo: Photo;
}
```

#### Move Photo (Drag-and-Drop)
```typescript
interface MovePhotoRequest {
  photoId: string;
  targetAlbumId: string;
  newOrder: number;
}

interface MovePhotoResponse {
  photo: Photo;
  affectedPhotos: Photo[];  // Photos whose order changed
}
```

#### Delete Photo
```typescript
interface DeletePhotoRequest {
  id: string;
}

interface DeletePhotoResponse {
  success: boolean;
}
```

## Event Bus (Internal)

```typescript
// Album events
type AlbumCreatedEvent = { type: 'album:created'; album: Album };
type AlbumUpdatedEvent = { type: 'album:updated'; album: Album };
type AlbumDeletedEvent = { type: 'album:deleted'; albumId: string };

// Photo events
type PhotoUploadedEvent = { type: 'photo:uploaded'; photo: Photo };
type PhotoMovedEvent = { type: 'photo:moved'; photo: Photo };
type PhotoDeletedEvent = { type: 'photo:deleted'; photoId: string };

type AppEvent = 
  | AlbumCreatedEvent 
  | AlbumUpdatedEvent 
  | AlbumDeletedEvent
  | PhotoUploadedEvent
  | PhotoMovedEvent
  | PhotoDeletedEvent;
```
```

#### 步骤 3: 生成快速开始指南

```markdown
# Quickstart Guide

## Prerequisites
- Node.js 18+
- Modern browser (Chrome/Firefox/Edge)

## Setup

```bash
npm install
npm run dev
```

## Key Validation Scenarios

### 1. Create First Album
1. Click "New Album" button
2. Enter album name "Test Album"
3. Click "Create"
4. **Expected**: Album appears in album list
5. **Verify**: Album exists in database (check DevTools Application tab)

### 2. Upload Photo
1. Click on "Test Album"
2. Drag a photo file to drop zone
3. **Expected**: Photo appears in album
4. **Verify**: Photo entry exists in database
5. **Verify**: Thumbnail generated and displayed

### 3. Drag Photo to Different Album
1. Create second album "Another Album"
2. Drag photo from "Test Album" to "Another Album"
3. **Expected**: Photo moves to new album
4. **Verify**: Photo's albumId updated in database
5. **Verify**: Order updated in both albums

### 4. Delete Album
1. Click "Test Album"
2. Click "Delete Album" button
3. Confirm deletion
4. **Expected**: Album removed from list
5. **Expected**: All photos in album also deleted
6. **Verify**: Album and photos removed from database

## Browser DevTools Checks

1. **Console**: No errors or warnings
2. **Network**: Verify API calls to SQLite
3. **Application**: Check IndexedDB for data
4. **Performance**: Check drag-and-drop is smooth (60fps)

## Common Issues

| Issue | Solution |
|-------|----------|
| Photos not uploading | Check browser permissions for file access |
| Drag-and-drop not working | Verify browser supports HTML5 Drag and Drop API |
| Data not persisting | Check IndexedDB is enabled in browser |
```

### Phase 2: 宪章合规检查

AI 运行 Phase -1 关卡检查：

```markdown
## Phase -1: Pre-Implementation Gates

### Simplicity Gate (Article VII)
- [x] Using ≤3 projects? ✓ (Just main Vite project)
- [x] No future-proofing? ✓ (No abstractions added)
- [x] Minimal dependencies? ✓ (Only essential packages)

### Anti-Abstraction Gate (Article VIII)
- [x] Using framework directly? ✓ (Using Vite as-is)
- [x] Single model representation? ✓ (TypeScript interfaces match schema)
- [x] No unnecessary wrappers? ✓ (Direct SQLite usage)

### Integration-First Gate (Article IX)
- [x] Contracts defined? ✓ (API contracts documented)
- [x] Real database used? ✓ (SQLite via sql.js, not mocked)
- [x] Integration tests planned? ✓ (Quickstart includes integration validation)

**Result**: All gates passed ✓
```

### Phase 3: 更新 AI 代理上下文

脚本 `update-agent-context.sh` 执行：

```bash
# 检测当前使用的 AI 代理
if [ -d ".claude" ]; then
    AGENT="claude"
elif [ -d ".gemini" ]; then
    AGENT="gemini"
# ... 其他代理
fi

# 更新对应的上下文文件
AGENT_FILE=".$AGENT_FOLDER/specify-rules.md"

# 在标记之间添加新技术
# Start Marker: <!-- SPECIFY_START -->
# End Marker: <!-- SPECIFY_END -->

# 添加 Vite、SQLite、Preact signals 的信息
```

### 输出成果

```
✓ Plan: specs/003-photo-album-organizer/plan.md
✓ Research: specs/003-photo-album-organizer/research.md
✓ Data Model: specs/003-photo-album-organizer/data-model.md
✓ Contracts: specs/003-photo-album-organizer/contracts/
✓ Quickstart: specs/003-photo-album-organizer/quickstart.md
✓ Agent Context: Updated with new technologies
✓ Ready for: /speckit.tasks
```

---

## 🔶 阶段 5: Tasks（任务分解）

### 目的

将技术计划分解为可顺序执行的具体任务。

### 执行流程

```bash
/speckit.tasks
```

### AI 的工作

#### 步骤 1: 分析输入文档

AI 读取：
- `plan.md` - 主要计划
- `data-model.md` - 数据模型
- `contracts/` - API 契约
- `research.md` - 研究结果

#### 步骤 2: 按用户故事组织任务

```markdown
# Implementation Tasks

## User Story 1: Create Albums

### Setup Tasks
- [T1-1] Initialize Vite project with vanilla TypeScript
- [T1-2] Install dependencies (sql.js, @preact/signals)
- [T1-3] Set up project structure (src/, public/, assets/)

### Database Tasks
- [T1-4] Create SQLite database initialization module
- [T1-5] Define database schema for albums table
- [T1-6] Create Album model interface and repository
- [T1-7] Implement Album CRUD operations

### UI Components
- [T1-8] Create AlbumList component
- [T1-9] Create CreateAlbumForm component
- [T1-10] Create AlbumItem component

### Integration
- [T1-11] Integrate Album repository with AlbumList
- [T1-12] Connect CreateAlbumForm to repository
- [T1-13] Implement album selection state management
- [T1-14] Add error handling and user feedback

### Testing
- [T1-15] Write unit tests for Album repository
- [T1-16] Write integration tests for CRUD operations
- [T1-17] Write e2e tests for album creation flow

## User Story 2: Upload and Display Photos

### Database Tasks
- [T2-1] Define database schema for photos table
- [T2-2] Create Photo model interface and repository
- [T2-3] Implement Photo upload operation
- [T2-4] Implement photo query by album
- [T2-5] Implement photo order updates

### UI Components
- [T2-6] Create PhotoGrid component
- [T2-7] Create PhotoItem component
- [T2-8] Create UploadZone component (drag-and-drop)
- [T2-9] Implement thumbnail generation

### Integration
- [T2-10] Integrate Photo repository with PhotoGrid
- [T2-11] Connect UploadZone to Photo upload operation
- [T2-12] Implement photo ordering (drag-and-drop within album)
- [T2-13] Add loading states and progress indicators

### Testing
- [T2-14] Write unit tests for Photo repository
- [T2-15] Write integration tests for upload flow
- [T2-16] Write e2e tests for drag-and-drop interactions

## User Story 3: Move Photos Between Albums

### Database Tasks
- [T3-1] Implement photo move operation (update albumId and order)
- [T3-2] Add transaction support for atomic updates
- [T3-3] Implement order rebalancing for affected albums

### UI Components
- [T3-4] Update UploadZone to support album selection on drop
- [T3-5] Implement visual feedback during drag operation
- [T3-6] Add confirmation dialog for photo moves

### Integration
- [T3-7] Connect drag-and-drop to photo move operation
- [T3-8] Implement optimistic updates
- [T3-9] Add rollback handling for failed moves

### Testing
- [T3-10] Write unit tests for photo move operation
- [T3-11] Write integration tests for multi-album moves
- [T3-12] Write e2e tests for drag-and-drop between albums

## User Story 4: Delete Albums

### Database Tasks
- [T4-1] Implement album delete operation (cascade delete photos)
- [T4-2] Add confirmation prompt
- [T4-3] Implement soft delete or hard delete?

### UI Components
- [T4-4] Add delete button to AlbumItem
- [T4-5] Create DeleteConfirmationModal component

### Integration
- [T4-6] Connect delete button to delete operation
- [T4-7] Update UI to refresh album list after deletion
- [T4-8] Handle edge cases (deleting album with photos)

### Testing
- [T4-9] Write unit tests for cascade delete
- [T4-10] Write integration tests for delete with photos
- [T4-11] Write e2e tests for album deletion flow
```

#### 步骤 3: 标记并行任务

```markdown
### Parallel Task Groups

#### Group 1: Initial Setup (Can run in parallel after Story 1 Setup)
- [P] T1-1: Initialize Vite project
- [P] T1-2: Install dependencies
- [P] T1-3: Set up project structure

#### Group 2: Database Schema (Can run in parallel)
- [P] T1-5: Define albums schema
- [P] T2-1: Define photos schema

#### Group 3: Component Creation (Can run in parallel within stories)
- [P] T1-8: Create AlbumList
- [P] T1-9: Create CreateAlbumForm
- [P] T1-10: Create AlbumItem

### Dependencies
- T1-4 requires T1-1, T1-2, T1-3 (Setup complete)
- T1-11 requires T1-4, T1-8 (Database and UI ready)
- T2-6 requires T2-1, T2-2 (Photo schema and model ready)
```

#### 步骤 4: TDD 文件顺序

```markdown
### File Creation Order (TDD Approach)

#### User Story 1: Create Albums

**Test Files** (Write first, ensure they fail):
1. `src/__tests__/album-repository.test.ts`
2. `src/__tests__/album-integration.test.ts`
3. `src/__tests__/e2e/album-creation.test.ts`

**Source Files** (Implement to make tests pass):
4. `src/models/album.ts`
5. `src/repositories/album-repository.ts`
6. `src/components/AlbumList.tsx`
7. `src/components/CreateAlbumForm.tsx`
8. `src/components/AlbumItem.tsx`
9. `src/services/album-service.ts`
10. `src/app.tsx` (Integration point)
```

### 输出成果

```
✓ Tasks: specs/003-photo-album-organizer/tasks.md
✓ Total Tasks: 47
✓ Parallel Groups: 3
✓ Estimated Time: 6-8 hours
✓ Ready for: /speckit.implement
```

---

## 🟣 可选阶段: Analyze（交叉一致性分析）

### 目的

检查规范、计划、数据模型和契约之间的一致性和完整性。

### 执行流程

```bash
/speckit.analyze
```

### AI 检查项

```markdown
# Cross-Artifact Analysis Report

## 1. Specification ↔ Plan Consistency

### ✓ Matches
- All user stories in spec have corresponding implementation in plan
- Success criteria in spec are measurable via quickstart scenarios

### ⚠️ Gaps Found
- FR-004 mentions "photo editing" but no plan for editing features
  → **Recommendation**: Add editing tasks to plan or remove from spec

## 2. Data Model ↔ Contracts Consistency

### ✓ Matches
- Album entity matches CreateAlbumRequest/Response
- Photo entity matches UploadPhotoRequest/Response

### ⚠️ Inconsistencies
- DataModel has `Album.order` field but no API contract for reordering albums
  → **Recommendation**: Add UpdateAlbumOrder contract

## 3. Plan ↔ Tasks Completeness

### ✓ Complete
- All design artifacts have corresponding tasks
- Database schema tasks precede repository tasks

### ⚠️ Missing Tasks
- Plan mentions "thumbnail generation" but no specific task
  → **Recommendation**: Add task for thumbnail generation optimization

## 4. Constitution Compliance

### ✓ Compliant
- Simplicity gate passed (≤3 projects)
- Anti-abstraction gate passed (no unnecessary wrappers)

### ⚠️ Potential Issues
- Plan uses custom AlbumService (wrapper around SQLite)
  → **Recommendation**: Consider using SQLite directly per Article VIII

## Summary

| Category | Status | Issues |
|----------|--------|---------|
| Spec ↔ Plan | ⚠️ 1 gap | Photo editing feature |
| Model ↔ Contracts | ⚠️ 1 inconsistency | Album reordering |
| Plan ↔ Tasks | ⚠️ 1 missing | Thumbnail generation |
| Constitution | ⚠️ 1 concern | AlbumService wrapper |

**Overall**: 4 minor issues found. Address before implementation.
```

---

## 🟤 可选阶段: Checklist（质量检查清单）

### 目的

生成自定义质量检查清单，验证需求的完整性、清晰度和一致性。

### 执行流程

```bash
/speckit.checklist photo album domain
```

### AI 生成的清单

```markdown
# Quality Checklist: Photo Album Organizer

## Domain-Specific Checks

### Photo Management
- [ ] Are there limits on number of photos per album?
- [ ] What happens when photos exceed storage quota?
- [ ] Are there file type restrictions (JPEG, PNG, RAW)?
- [ ] How are large files handled (e.g., 50MB RAW)?
- [ ] Is there photo deduplication (same file in multiple albums)?

### Album Organization
- [ ] Can albums be nested (albums within albums)?
- [ ] Is there a limit on number of albums?
- [ ] Can albums be reordered (drag-and-drop albums)?
- [ ] How are albums sorted (alphabetical, custom order, date)?
- [ ] Can albums be shared between users?

### User Experience
- [ ] What happens when drag-and-drop fails?
- [ ] Is there undo/redo for photo moves?
- [ ] How are errors communicated to user?
- [ ] Is there offline support?
- [ ] What happens on slow networks?

### Data Persistence
- [ ] Where is data stored? Local only? Cloud? Both?
- [ ] How is data backed up?
- [ ] Can data be exported/imported?
- [ ] What happens when database is corrupted?
- [ ] Is there data migration plan for future schema changes?

### Performance
- [ ] What's the expected number of photos per album?
- [ ] How is thumbnail generation optimized?
- [ ] Is there lazy loading for large photo sets?
- [ ] What's the target render time for photo grids?
- [ ] Are there memory limits?

## General Quality Checks

### Completeness
- [ ] All [NEEDS CLARIFICATION] markers resolved
- [ ] All user stories have acceptance criteria
- [ ] Edge cases identified and handled
- [ ] Error scenarios covered

### Clarity
- [ ] Requirements are testable
- [ ] No ambiguous language
- [ ] Technical terms defined
- [ ] Examples provided where helpful

### Consistency
- [ ] Terminology consistent across artifacts
- [ ] Data model matches API contracts
- [ ] Plan tasks cover all design elements

## Action Items

1. [ ] Resolve photo deduplication question
2. [ ] Clarify offline support requirements
3. [ ] Define performance targets (photos per album, render time)
4. [ ] Decide on album nesting support
5. [ ] Add error handling scenarios to spec
```

---

## 🔴 阶段 6: Implementation（实现执行）

### 目的

按照任务列表顺序执行所有任务，构建完整的功能。

### 执行流程

```bash
/speckit.implement
```

### AI 的工作

#### 步骤 1: 验证前置条件

AI 检查：
- `constitution.md` 存在
- `spec.md` 存在
- `plan.md` 存在
- `tasks.md` 存在
- 所有检查清单通过（如果存在）

#### 步骤 2: 解析任务列表

```python
# 伪代码
tasks = parse_tasks("specs/003-photo-album-organizer/tasks.md")

# 按依赖顺序排序
ordered_tasks = topological_sort(tasks)

# 标记并行任务
parallel_groups = identify_parallel_tasks(ordered_tasks)
```

#### 步骤 3: 执行任务

AI 按照 TDD 顺序执行：

```markdown
## Task T1-15: Write unit tests for Album repository

**File**: `src/__tests__/album-repository.test.ts`

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { AlbumRepository } from '../repositories/album-repository';
import { Album } from '../models/album';

describe('AlbumRepository', () => {
  let repo: AlbumRepository;
  
  beforeEach(() => {
    repo = new AlbumRepository();
  });
  
  describe('create', () => {
    it('should create a new album with valid data', async () => {
      const album: Omit<Album, 'id' | 'createdAt' | 'updatedAt'> = {
        name: 'Test Album',
        description: 'Test description',
        order: 0
      };
      
      const result = await repo.create(album);
      
      expect(result).toBeDefined();
      expect(result.id).toBeDefined();
      expect(result.name).toBe('Test Album');
      expect(result.createdAt).toBeInstanceOf(Date);
      expect(result.updatedAt).toBeInstanceOf(Date);
    });
    
    it('should throw error when name is empty', async () => {
      const album = { name: '', order: 0 };
      
      await expect(repo.create(album)).rejects.toThrow('Name is required');
    });
  });
  
  describe('findAll', () => {
    it('should return all albums sorted by order', async () => {
      await repo.create({ name: 'Album 2', order: 1 });
      await repo.create({ name: 'Album 1', order: 0 });
      
      const albums = await repo.findAll();
      
      expect(albums).toHaveLength(2);
      expect(albums[0].name).toBe('Album 1');
      expect(albums[1].name).toBe('Album 2');
    });
  });
});
```

**Status**: ✅ Tests written (ensure they fail - Red Phase)
```

```markdown
## Task T1-4: Create SQLite database initialization module

**File**: `src/db/database.ts`

```typescript
import initSqlJs, { Database, SqlJsStatic } from 'sql.js';

let db: Database | null = null;
let SQL: SqlJsStatic | null = null;

export async function initializeDatabase(): Promise<Database> {
  if (db) return db;
  
  SQL = await initSqlJs({
    locateFile: (file) => `https://sql.js.org/dist/${file}`
  });
  
  db = new SQL.Database();
  
  // Create tables
  db.run(`
    CREATE TABLE IF NOT EXISTS albums (
      id TEXT PRIMARY KEY,
      name TEXT NOT NULL,
      description TEXT,
      created_at INTEGER NOT NULL,
      updated_at INTEGER NOT NULL,
      cover_photo_id TEXT,
      "order" INTEGER NOT NULL
    );
  `);
  
  return db;
}

export function getDatabase(): Database {
  if (!db) throw new Error('Database not initialized');
  return db;
}
```

**Status**: ✅ Implemented (tests pass - Green Phase)
```

#### 步骤 4: 运行测试

AI 自动运行测试套件：

```bash
npm test
```

#### 步骤 5: 错误处理

如果测试失败：

```markdown
## Error: AlbumRepository.create() test failed

**Test**: `should create a new album with valid data`
**Error**: `Expected 'Test Album' but got undefined`

**Diagnosis**: Album.name is not being saved to database

**Fix**: Check SQL INSERT statement in AlbumRepository

```diff
- db.run('INSERT INTO albums (id, name, description, order) VALUES (?, ?, ?, ?)',
+ db.run('INSERT INTO albums (id, name, description, created_at, updated_at, "order") VALUES (?, ?, ?, ?, ?, ?)',
   [album.id, album.name, album.description, Date.now(), Date.now(), album.order]
```

**Status**: ✅ Tests passing
```

#### 步骤 6: 集成测试

运行快速开始指南中的验证场景：

```bash
# Scenario 1: Create First Album
npm run dev
# Manual: Click "New Album", enter name, click create
# Verify: Album appears in list and database
```

### 输出成果

```
✓ Task T1-1 to T1-17: Completed (17/17)
✓ Task T2-1 to T2-16: Completed (16/16)
✓ Task T3-1 to T3-12: Completed (12/12)
✓ Task T4-1 to T4-11: Completed (11/11)
✓ All Tests: Passing (47/47)
✓ Quickstart Scenarios: Validated (4/4)
✓ Implementation: Complete
```

---

## 🎯 工作流关键原则

### 1. 阶段不可逆

每个阶段必须在完成后才能进入下一阶段：

```
Constitution → Specification → Planning → Tasks → Implementation
```

不能跳过阶段，但可以使用可选命令（Clarify, Analyze, Checklist）增强质量。

### 2. 文档优先

代码从文档生成，而非相反：

- `spec.md` 定义 WHAT 和 WHY
- `plan.md` 定义 HOW
- `tasks.md` 定义 WHEN 和 IN WHAT ORDER
- 代码只是这些文档的表达

### 3. 质量门控

每个阶段都有明确的关卡：

- Specification: 质量检查清单
- Planning: 宪章关卡检查
- Tasks: TDD 红绿黄循环
- Implementation: 快速开始验证

### 4. 持续迭代

工作流支持迭代：

```bash
# 修改规范
/speckit.specify Add photo editing features

# 重新规划
/speckit.plan Use canvas API for image editing

# 重新生成任务
/speckit.tasks

# 重新实现
/speckit.implement
```

### 5. 并行探索

可以为同一规范生成多个计划：

```bash
# 计划 A: 使用 Vanilla JS
/speckit.plan Use vanilla HTML/CSS/JS

# 在新分支探索计划 B
git checkout -b 004-photo-album-organizer-react
/speckit.plan Use React with TypeScript

# 比较两个实现
```

---

## 📊 工作流效率对比

### 传统方式

| 阶段 | 时间 | 人工介入 |
|------|------|----------|
| 需求文档 | 2-3 天 | 高 |
| 设计文档 | 2-3 天 | 高 |
| 技术规范 | 3-4 天 | 高 |
| 编码 | 5-10 天 | 高 |
| 测试 | 2-3 天 | 高 |
| **总计** | **14-23 天** | **持续** |

### SDD 方式

| 阶段 | 时间 | 人工介入 |
|------|------|----------|
| Constitution | 30 分钟 | 中 |
| Specification | 1-2 小时 | 低 |
| Clarification (可选) | 30 分钟 | 中 |
| Planning | 1-2 小时 | 低 |
| Tasks | 30 分钟 | 低 |
| Analyze/Checklist (可选) | 30 分钟 | 低 |
| Implementation | 2-4 小时 | 低 |
| **总计** | **6-11 小时** | **间歇性** |

**效率提升**: 2-4 倍

---

## 🎓 学习要点

1. **阶段清晰**：每个阶段有明确的输入、输出和目标
2. **文档驱动**：所有工作从文档开始，以文档结束
3. **质量门控**：每个阶段都有质量检查点
4. **TDD 强制**：测试必须在实现之前编写
5. **宪章约束**：所有技术决策必须符合项目原则
6. **可逆性**：可以随时回到任意阶段重新开始
7. **并行性**：支持多分支并行探索不同方案
8. **自动化**：大部分工作由 AI 自动完成
9. **可追踪**：每个决策都有文档记录
10. **可维护**：规范就是代码的文档，永远同步

下一节将深入模板与提示词系统的实现机制。