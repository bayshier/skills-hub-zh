# Skills Hub - 项目规范

## 概述

Skills Hub 是一个跨平台桌面应用（Tauri 2 + React 19），用于管理 AI Agent Skills 并将它们同步到 47+ 个 AI 编程工具。核心理念："一次安装，处处同步。"

## 技术栈

- **前端**：React 19 + TypeScript 5.9（严格模式）+ Vite 7 + Tailwind CSS 4
- **后端**：Rust（Edition 2021，MSRV 1.77.2）+ Tauri 2
- **数据库**：SQLite（rusqlite，bundled）
- **Git**：libgit2（git2 crate，vendored-openssl）
- **HTTP**：reqwest（rustls-tls，blocking）
- **i18n**：i18next（英文/中文双语）
- **通知**：sonner（toast）
- **图标**：lucide-react

## 常用命令

```bash
npm run dev              # Vite 开发服务器（端口 5173）
npm run tauri:dev        # Tauri 开发窗口（前端 + 后端）
npm run build            # tsc + vite build
npm run check            # 完整检查：lint + build + rust:fmt:check + rust:clippy + rust:test
npm run lint             # ESLint（flat config v9）
npm run rust:test        # cargo test
npm run rust:clippy      # Rust lint
npm run rust:fmt         # Rust 格式化
npm run rust:fmt:check   # Rust 格式化检查
```

提交前务必运行 `npm run check`，确保所有检查通过。

## 目录结构

```
src/                          # React 前端
├── App.tsx                   # 根组件（集中式状态，所有弹窗状态）
├── App.css                   # 全局样式（所有组件样式都写在这里）
├── index.css                 # CSS 变量（主题化）+ Tailwind 入口
├── components/
│   ├── Layout.tsx            # 主布局（侧边栏 + 内容区）
│   └── skills/               # Skills 功能模块
│       ├── Header.tsx        # 顶栏（品牌标识 + 语言切换 + 新建按钮）
│       ├── FilterBar.tsx     # 筛选/排序栏
│       ├── SkillsList.tsx    # Skills 列表容器
│       ├── SkillCard.tsx     # 单个 Skill 卡片
│       ├── LoadingOverlay.tsx
│       ├── types.ts          # 共享 DTO 类型定义（前端 ↔ 后端）
│       └── modals/           # 弹窗组件（共 8 个）
└── i18n/
    ├── index.ts              # i18next 初始化
    └── resources.ts          # 翻译资源（EN/ZH）

src-tauri/src/                # Rust 后端
├── main.rs                   # 入口（调用 app_lib::run）
├── lib.rs                    # 应用初始化（插件注册、数据库、清理任务）
├── commands/
│   ├── mod.rs                # Tauri 命令层（23 个命令 + DTO）
│   └── tests/
└── core/                     # 核心业务逻辑
    ├── skill_store.rs        # SQLite ORM（4 张表：skills、skill_targets、settings、discovered_skills）
    ├── installer.rs          # Skill 安装（本地/git，带多 Skill 检测）
    ├── sync_engine.rs        # 同步引擎（symlink/junction/copy 三重回退）
    ├── git_fetcher.rs        # Git clone/pull（带缓存和 TTL）
    ├── tool_adapters/mod.rs  # 工具适配器注册表（47 个 AI 工具）
    ├── onboarding.rs         # 已有 Skill 扫描/发现
    ├── github_search.rs      # GitHub API 搜索
    ├── central_repo.rs       # 中心仓库路径管理
    ├── content_hash.rs       # SHA256 目录内容哈希
    ├── cache_cleanup.rs      # Git 缓存清理
    ├── temp_cleanup.rs       # 临时目录清理
    └── tests/                # 每个模块一个测试文件（共 10 个）
```

## 架构

### 前端 ↔ 后端通信
- 使用 Tauri IPC（`invoke`）调用后端命令
- 前端调用模式：`const result = await invoke('command_name', { param })`
- 后端命令定义在 `commands/mod.rs`，并通过 `generate_handler!` 在 `lib.rs` 中注册
- 新增命令必须在这两处同时注册

### 前端状态管理
- **不使用状态管理库** —— 所有状态通过 `useState` 集中在 `App.tsx` 中
- 通过 props 逐层传递给子组件（弹窗会接收大量 props）
- 数据刷新模式：操作完成后调用 `invoke('get_managed_skills')` 重新获取列表

### 后端分层
- `commands/` 层：Tauri 命令定义、DTO 转换、错误格式化（不含业务逻辑）
- `core/` 层：纯业务逻辑，可独立测试
- 异步命令使用 `tauri::async_runtime::spawn_blocking` 包装同步操作
- 共享状态通过 `app.manage(store)` + `State<'_, SkillStore>` 注入

### 错误处理
- 后端使用 `anyhow::Result<T>`，通过 `format_anyhow_error()` 转换为字符串传给前端
- 使用特殊错误前缀供前端识别：`MULTI_SKILLS|`、`TARGET_EXISTS|`、`TOOL_NOT_INSTALLED|`
- 前端用 try-catch 捕获，并通过 sonner toast 展示错误

## 编码规范

### TypeScript
- 严格模式：已启用 `noUnusedLocals` 和 `noUnusedParameters` —— 未使用的变量/参数会导致编译错误
- 组件文件：PascalCase（`SkillCard.tsx`）
- Props 类型：`ComponentNameProps`（`SkillCardProps`）
- CSS 类名：kebab-case（`modal-backdrop`、`skill-card`）
- 弹窗条件渲染：`if (!open) return null`（完全卸载，而非 display:none）
- 展示型组件用 `memo()` 包裹
- 所有用户可见文本必须使用 i18n（`t('key')`），翻译 key 定义在 `src/i18n/resources.ts`
- 新增文本时，必须同时提供英文和中文翻译
- DTO 类型定义在 `src/components/skills/types.ts`，必须与 `commands/mod.rs` 中的 Rust DTO 保持同步

### Rust
- 函数/方法：snake_case
- 常量：SCREAMING_SNAKE_CASE
- Tauri 命令参数使用 camelCase（与前端 JS 调用惯例保持一致）
- 使用 `anyhow::Context` 为错误添加上下文
- 新的核心模块必须在 `core/mod.rs` 中导出
- 测试使用 `tempfile` crate 创建临时目录，使用 `mockito` 进行 HTTP 模拟

### 样式
- 组件样式写在 `src/App.css`（不使用 CSS Modules），采用语义化 CSS 类名
- 通过 CSS 变量 + `[data-theme="dark"]` 选择器实现主题化，变量定义在 `src/index.css`
- Tailwind 工具类和自定义 CSS 类可以混用

### UI 设计参考
- 在进行会改变布局、视觉样式、共享组件、导航、浮层、响应式行为或交互反馈的前端工作之前，只加载 `docs/UI-DESIGN-GUIDELINES.md`。
- 对于纯后端、纯数据、纯测试、发布或文档类任务，除非同时改变产品 UI，否则不要加载 UI 指南。

## 开发流程

1. **实现之前**：简要描述方案并列出要修改的文件。等待确认后再编写代码。
2. **完整实现**：对于同时涉及前端和后端的功能，一次性完成两侧修改 —— 包括 Tauri 命令注册、DTO 类型、i18n 翻译（中英双语）和 UI。
3. **变更后验证**：实现完成后务必运行 `npm run check`，确保 lint、构建和所有 Rust 检查通过。在呈现结果之前修复所有错误。
4. **保持改动最小**：只修改需求所必需的内容。不要重构、添加注释或"改进"无关代码。

## 重要说明

- 路径处理必须支持 `~` 展开（后端有 `expand_home_path()`）
- 同步策略采用三重回退：symlink → junction（Windows）→ copy
- Git 使用 vendored-openssl，HTTP 使用 rustls-tls —— 规避系统 SSL 问题
- 版本号必须在 `package.json` 和 `src-tauri/tauri.conf.json` 之间保持同步（用 `npm run version:check` 校验）
- Rust crate 名为 `app_lib`（不是默认包名）—— 导入时使用 `app_lib::...`
- 数据库有 schema 迁移机制（`migrate_legacy_db_if_needed`）—— 修改表结构时需考虑迁移
- 工具适配器列表在 `tool_adapters/mod.rs` —— 新增 AI 工具需要同时添加 `ToolId` 枚举变体和适配器实例
