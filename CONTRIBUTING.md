# 参与贡献

感谢你抽出时间为 Skills Hub 做贡献！

## 开发环境要求

- Node.js 18+（推荐 20+）
- Rust（stable）
- Tauri 系统依赖（按照 Tauri 官方文档针对 macOS/Windows/Linux 安装）

## 本地运行

```bash
npm install
npm run tauri:dev
```

## 质量检查

```bash
npm run lint
npm run build
```

## 运行单元测试

Rust 单元测试位于 `src-tauri/src/core/tests/` 目录下。

```bash
cd src-tauri
cargo test
```

## 提交 PR 之前

- 确保 `npm run lint` 和 `npm run build` 通过
- 确保 `cd src-tauri && cargo test` 通过
- 保持改动小而聚焦（不要提交本地配置/缓存/构建产物）
- UI 改动请附上截图或简短录屏

## 报告问题

请在 issue 中包含以下信息：

- 操作系统版本（macOS/Windows/Linux）
- Skills Hub 版本
- 复现步骤，以及期望行为与实际行为的差异
- 相关日志（请对本地路径和敏感信息做脱敏处理）
