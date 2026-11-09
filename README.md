# dsh-session-delete

[中文](#中文) | [English](#english)

## 预览

![删除会话](docs/images/delete-session.png)

![删除全部会话](docs/images/delete-all-sessions.png)

## 中文

DSH（DeepSeek Harness）工作区侧栏增强插件：补上官方缺失的**会话删除**能力，并增加目录级批量操作。

### 功能

| 入口 | 菜单项 | 行为 |
|---|---|---|
| 会话行 `…` 菜单 | **删除会话** | 永久删除该会话的全部本地记录（`~/.dsh/sessions/<项目目录>/<会话目录>/`，含日志文件），不可恢复 |
| 目录行 `…` 菜单 | **归档全部会话** | 一键归档该工作区下所有未归档会话（记录保留，仅隐藏） |
| 目录行 `…` 菜单 | **删除全部会话** | 批量删除该工作区全部会话（含已归档） |

### 安全设计

- 红色危险确认对话框，明示"不可恢复"及目标数量
- **运行中/已打开的会话一律跳过**（含当前会话），防止误删活跃会话
- 删除命令运行在受限沙箱内（`workspace-write`，root 限定为该会话所属的项目目录）
- 先归档隐藏、再删文件、最后摘工作区账目：删文件失败会自动回滚归档，不会留下"已隐藏但未删除"的半删状态

### 安装

```bash
dsh plugin --profile web add dsh-session-delete
```

（自动加入 profile 的 `dsh.profile.bundles` 层，包内 `cordis.patch.yml` 随层生效，无需手动编辑。`web` 换成你的 profile 名。）

手动兜底（不使用 `dsh plugin` 时）：

```bash
cd ~/.dsh/profiles/web
pnpm add dsh-session-delete --registry https://registry.npmjs.org
```

然后在 `cordis.patch.yml` 添加挂载行（无需 config）：

```yaml
- insert:
    - id: dsh-session-delete
      name: dsh-session-delete
```

重启 DSH 生效。命令菜单文案自动跟随 GUI 语言（中文/英文）。

### 工作原理

- **Host 半**（`index.js`）：注册 `POST /dsh-session-delete/{delete,delete-all,archive-all}` 同源路由；删除经 `ctx.shell` 在按调用沙箱策略下执行（POSIX `rm` / Windows `Remove-Item` 自动分派，路径单引号转义防注入）
- **Client 半**（`client.js`，`dsh.client` web bundle）：点击捕获 + React fiber 读取行数据；MutationObserver 在菜单挂载后克隆现有菜单项 DOM 注入新命令（样式/主题自动一致），操作经同源 `fetch` 调 Host 路由

> 注：浏览器半通过 DOM 增强注入菜单（官方未开放该菜单的扩展 Slot）。DSH 大版本升级若调整侧栏结构，插件可能需要跟进——菜单未注入时功能静默缺失，不会有其他影响。

### 兼容性

- DSH `0.2.1`（已在 0.2.1-rc.2 与 0.2.1-rc.5 验证；`sessionPersistence.list()` 的快照形态、`shell.execute`，以及 rc.5 改用 `data-row-key` 标识侧栏行都已适配）与 `0.1.0-rc.6`（依赖 `webServer` / `shell` / `sessionPersistence` / `workspaceRegistry` / `sessions` 服务）
- Linux / Windows（按会话路径分隔符自动分派删除命令）

## English

A DSH (DeepSeek Harness) sidebar enhancement plugin: adds the missing **session deletion** capability plus directory-level bulk operations.

- Session row menu → **Delete Session**: permanently removes the session's local records (log directory included)
- Workspace row menu → **Archive All Sessions** / **Delete All Sessions**

Safety: danger-styled confirmation dialog; live/opened sessions are always skipped; removal runs in a per-call sandbox (`workspace-write` scoped to the session's project directory); the session is archived before its files are removed and a failed removal rolls the archive back, so a failure never leaves a half-deleted state.

Install: `dsh plugin --profile <name> add dsh-session-delete` (or `pnpm add dsh-session-delete` plus a manual `- insert: dsh-session-delete` row in `cordis.patch.yml`), restart DSH. UI language follows the GUI locale (zh/en).

## 来源与致谢 / Credits

本仓库是 [vtxf/dsh-session-delete](https://github.com/vtxf/dsh-session-delete) 的分支（fork）。原始实现、设计文档与截图均由原作者 **vtxf** 完成，以 MIT 许可发布，版权归原作者所有。感谢原作者提供了这个插件的完整基础实现。

This repository is a fork of [vtxf/dsh-session-delete](https://github.com/vtxf/dsh-session-delete). The original implementation, docs and screenshots were written by **vtxf** and released under the MIT license; the copyright remains with the original author. Thanks to the original author for the complete base implementation.

本分支的改动 / Changes in this fork：

- 适配 DSH `0.2.x`：`sessionPersistence.list()` 改为快照形态（`{ header, revision, … }`）后仍能定位会话；`shell.run()` 移除后改用 `shell.execute(spec)` + `result()`，并保留旧版回退
- 修正删除顺序为「归档 → 删文件 → 摘工作区账目」：`archiveSession` 会校验 `sessionKnown`，原先"先删文件后归档"必然失败，导至会话掉进「未分组」且无法归档；删文件失败时自动 `unarchive` 回滚
- 适配 rc.5 的侧栏结构：行标识改用 `data-row-key`（`session:<id>` / `workspace:<id>`），并识别「会话行菜单改用 `children` 传 slot 项、不再有 `items` 数组」这一变化，否则单会话删除不会出现在菜单里
- 版本号升至 `0.1.3`

## License

MIT，版权归原作者 vtxf 所有；本分支的修改同样以 MIT 发布。

```
Copyright (c) 2026 vtxf

SPDX-License-Identifier: MIT
```

完整许可文本见 [LICENSE](LICENSE)。再分发或基于本项目的修改版本时，请保留 `LICENSE` 中的版权声明与许可声明。
