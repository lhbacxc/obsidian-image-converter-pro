# Obsidian Image Converter Pro AI 升级说明

最近更新：2026-08-18（同步上游 1.4.6 后重构）

## 文档目的

这份文档是写给后续协助维护、升级、适配本插件源码仓库的 AI 的。

目标不是让 AI 重新设计功能，而是让 AI：

- 直接基于当前源码仓库继续维护
- 在上游更新后，**逐项识别并保留本 fork 全部 4 项自定义能力**（见"自定义能力总表"）
- 优先根据当前源码、测试和构建结果完成适配
- 尽量减少升级带来的回归问题

## 根目录约定

`obsidian-image-converter-pro` 本身就是项目根目录。本文档中所有路径均为相对项目根目录的相对路径；不要写死磁盘绝对路径。

## 自定义能力总表（升级时逐项核对）

本 fork 相对上游共有 **4 项自定义能力**。升级后任何一项都不允许丢失。下面按"涉及文件 → 关键实现 → 测试 → 验证要点"归纳：

| # | 能力 | 涉及源码 | 相关测试 |
|---|------|---------|---------|
| 1 | 非图片附件右键菜单增强 | `src/ContextMenu.ts` | `tests/integration/ui/ContextMenu.test.ts` |
| 2 | markdown 相对路径去掉 `./` | `src/LinkFormatter.ts` | `tests/unit/links/LinkFormatter.test.ts`、`LinkFormatSettings.mapping.test.ts` |
| 3 | 图片点击放大预览（`enableImageClickZoom`） | `src/ImageResizer.ts`、`src/main.ts`、`src/ImageConverterSettings.ts`、`styles.css` | `tests/integration/ui/ImageResizer.test.ts` |
| 4 | 命令面板插入附件触发重命名/压缩（create 兜底） | `src/main.ts` | `tests/integration/main/CreateHandler.test.ts` |

> 历史教训：早期文档主体只围绕能力 1 编写，导致升级时 AI 容易漏掉其他 3 项。**每次升级必须按本表逐项确认。**

---

## 能力详情与关键约束

### 能力 1：非图片附件右键菜单增强

目标行为：

- 图片附件继续走原有图片菜单逻辑，**不回归**
- 非图片附件可被右键识别并显示附件菜单，必须包含以下 5 项：
  - `Name`（重命名，保留原始扩展名）
  - `Copy file`（仅 Windows 桌面，PowerShell `Set-Clipboard`）
  - `Show in navigation`
  - `Show in system explorer`
  - `Delete file and link`（移除当前笔记链接 + 文件移入回收站）
- 支持识别的非图片附件目标：`.file-embed`、`.file-embed-title`、`audio`、`video`、`iframe`

关键实现位置（`src/ContextMenu.ts`）：

- `handleContextMenuEvent`：**img/attachment 双分支**——`resolveImageFromTarget` 失败后走 `resolveAttachmentTarget`；img 分支保持无条件 `showAtMouseEvent`（与上游一致），attachment 分支在找不到文件时（`createAttachmentContextMenuItems` 返回 false）不显示
- `resolveAttachmentTarget` / `getAttachmentFile`（路径解析）
- `createAttachmentContextMenuItems` / `addAttachmentRenameInput` / `renameAttachmentFile`
- `copyFileToSystemClipboard` / `showFileInNavigation` / `showFileInSystemExplorer` / `deleteAttachmentAndLinkFromNote`

适配约束（1.4.6 后新增约定）：

- 菜单输入 DOM 创建用 `ownerDocument`（popout 兼容）
- 输入框事件监听用 **`menu.registerDomEvent`**（上游事件治理风格，注册在 transient Menu 上，随菜单隐藏自动清理），不要改回 `this.registerDomEvent` 挂长生命周期

### 能力 2：markdown 相对路径去掉 `./`

约束（`src/LinkFormatter.ts` 的 `formatRelativePath`，接收 `linkFormat` 参数区分分支）：

1. `markdown + relative` 在子路径或同级路径下**不补 `./`**（输出 `附件/xxx.png`、`image.png`）
2. `markdown + relative` 在父级路径下**必须保留 `../`**（输出 `../images/xxx.png`）
3. `wikilink + relative` 逻辑不变，可保留显式 `./`
4. 路径中的空格仍编码为 `%20`

验证要点：同级不带 `./`、子目录不带 `./`、父目录保留 `../`、wikilink 原行为。

### 能力 3：图片点击放大预览

目标行为：

- 左键点击笔记内图片打开 lightbox 放大预览
- 点击遮罩空白区域关闭预览
- 预览内滚轮缩放、左键拖动平移，**只改变临时显示**（`translate(...) scale(...)`）
- 不修改图片文件、不写回 markdown 链接尺寸、不写入对齐缓存

实现约束：

- 独立设置项 `enableImageClickZoom`，默认开启；**关闭 `isImageResizeEnbaled` 总开关后点击放大仍可独立工作**
- `handleMouseWheel` 必须保留 `isImageResizeEnbaled` 总开关判断（避免只开点击放大时误触笔记内尺寸写回）
- 跳过 Excalidraw 图片、`.map-view-main`、resize handle、编辑按钮、lightbox 自身，以及 **`.cm-table-widget`**（1.4.6 起：表格内图片保留 Obsidian 原生处理，见测试 13.29d）
- 遮罩 `rgba(0, 0, 0, 0.58)`；lightbox 内部拖动不得复用笔记内 resize 写回逻辑
- lightbox 相关 DOM/CSS 类：`.image-converter-lightbox-overlay`、`.image-converter-lightbox-image`、`.image-converter-lightbox-image-dragging`、`.image-converter-lightbox-open`

关键实现位置：

- `src/ImageResizer.ts`：`getImageTargetForClickZoom`（入口过滤，含 `.cm-table-widget` 排除）、`openImageLightbox`、`closeImageLightbox`、`handleLightboxWheel`、`handleLightboxImageMouseDown/Move/Up`、`applyLightboxScale`、`handleMouseWheel`；lightbox 状态字段 12 个（`lightboxScale`/`lightboxPanX`/`lightboxPanY` 等）
- `src/main.ts`：**3 处条件必须同时考虑 `isImageResizeEnbaled || enableImageClickZoom`**——`attachImageResizerToMarkdownView`、`attachImageResizerToActiveView`、`registerImageResizerWorkspaceEvents`，以及 `initializeComponents` 中 ImageResizer 的初始化
- `src/ImageConverterSettings.ts`：接口字段 + `DEFAULT_SETTINGS.enableImageClickZoom: true` + 设置 UI。**UI 位置敏感**：必须放在 Drag & scroll resize section 头部（collapse 逻辑之后、`if (isImageResizeEnbaled)` 之前）；放在 "Disable Obsidian image selection" 与 "Cursor position" 之间会破坏上游测试的 DOM 相邻定位（`ImageConverterSettings.test.ts` 的 toggle 查找逻辑）

验证要点：点击打开 / 空白关闭 / 滚轮缩放 / 拖动平移 / 不写回 / 关闭设置不打开 / 关闭 resize 总开关仍可放大 / **表格内图片不触发 lightbox**。

### 能力 4：命令面板插入附件 create 兜底

背景：Obsidian「命令面板插入附件」走 `vault.createBinary()` 直接建文件，**不触发 `editor-drop`/`editor-paste`**，官方无 insert-attachment 事件，只能靠 `vault.on('create')` 兜底。

实现（`src/main.ts`）：

- `registerVaultCreateHandler`：注册常驻 `vault.on('create')`（桌面 + 移动都注册）
- `handleFileCreated`：**四重过滤** → 共享管道处理 → 重命名/写回
  1. 支持格式 + 非 neverProcess
  2. `isSyncPath`（排除 git/syncthing/remotely-save 等同步工具路径）
  3. `selfCreatedPaths`（排除插件自身 drop/paste 路径创建的 `createBinary`——**drop/paste 6 处 `createBinary` 调用前必须 `selfCreatedPaths.add(newFullPath)`**）
  4. 链接证据：当前活动笔记是否引用该文件，**轮询等待**（每 150ms 一次、最长 3 秒）
- `noteReferencesFile`：两级链接匹配——`getFirstLinkpathDest` 精确解析，失败降级为链接文本包含 basename/name（中文文件名可命中）
- `getSelectedPresets`：从 drop/paste 抽取的共享预设选择（create 兜底独立调用；drop/paste 仍各自内联等价逻辑）
- 写回：路径未变 → `modifyBinary`；路径变化 → **`fileManager.renameFile`（Obsidian 原生同步所有笔记引用）** + `modifyBinary`
- `processingPaths` 防并发重复处理

关键时序坑（实测踩过）：

- `vault.on('create')` 触发时机**先于** Obsidian 插入链接，metadataCache 更新滞后 → 必须**轮询等待链接证据**，不要用固定延迟
- 处理失败时保留原文件，不删除

lint 注意事项：该方法组内 `setTimeout` 用 `window.setTimeout`；`.obsidian/plugins/...` 字符串需加 `eslint-disable obsidianmd/hardcoded-config-path`（同步工具路径是第三方固定路径，与 vault configDir 无关）。

---

## 升级适配工作流程

### 第 0 步：确认状态 + 备份

```powershell
git status --short --branch
git remote -v
git log --oneline --decorate -n 10
git tag backup-pre-<上游版本>-merge HEAD   # 合并前打备份 tag，可随时回退
```

### 第 1 步：抓取上游并对比

```powershell
git fetch upstream
git branch -a
git log --oneline 1.4.3..upstream/main          # 上游改了什么
git diff --stat 1.4.3 upstream/main -- src/      # 涉及哪些文件
git diff 1.4.3 HEAD -- src/                      # 本地自定义改了什么
```

### 第 2 步：三方文件分类（核心合并策略）

对每个文件判断归属，**不要直接 `git merge`**（两边都改的文件冲突面大，手动移植更可控）：

| 归属 | 处理方式 |
|------|---------|
| 仅上游改 | `git checkout upstream/main -- <file>` 直接取上游版 |
| 仅本地改 | 保留本地版（上游未动，天然无冲突） |
| 两边都改 | **以上游 1.4.6 结构为基底，手动移植本地功能** |

历史冲突面最大的文件（两边都改，需要手动合并）：`src/ContextMenu.ts`、`src/ImageResizer.ts`、`src/main.ts`、`src/ImageConverterSettings.ts`、`styles.css`、`tests/integration/ui/ContextMenu.test.ts`、`tests/integration/ui/ImageResizer.test.ts`。

### 第 3 步：逐能力适配

对「自定义能力总表」中每一项，按顺序执行：

1. 识别当前实现落在哪些方法/测试上（对照"能力详情"章节）
2. 对比上游新版结构是否重构了这些方法
3. 将仍然必要的实现移植到上游结构，**保持图片原有逻辑不变**
4. 同步更新对应测试

适配时遵守的通用约束：

- 保持图片原有菜单/交互逻辑不回归
- lightbox 滚轮缩放与拖动不得写回 markdown 尺寸、不得写入对齐缓存
- create 兜底的四重过滤与 `selfCreatedPaths` 标记必须完整保留
- 每项能力都有独立测试文件，行为变化先修源码，再按需调整测试，不要删测试

### 第 4 步：验证

```powershell
npm run build                                   # 必须通过
npx vitest run tests/integration/ui/ContextMenu.test.ts
npx vitest run tests/integration/ui/ImageResizer.test.ts
npx vitest run tests/integration/main/CreateHandler.test.ts
npx vitest run tests/unit/links/LinkFormatter.test.ts tests/unit/links/LinkFormatSettings.mapping.test.ts
npx vitest run                                   # 全量，对照下方"已知基线"
npm run lint                                     # 对照下方"已知基线"
git diff --check
```

## 已知基线（2026-08-18，同步 1.4.6 后实测）

升级验证时对照以下基线判断是否回归：

- `npx vitest run`：**694 通过 / 5 失败**。5 个失败全部在 `tests/unit/pathing/FolderAndFilenameManagement.test.ts` 的 **app:// 资源 URL 测试**——`pathToFileURL` 在 Windows 上给 POSIX 路径加盘符（`app://hash/d:/...`），**上游 Linux CI 才通过，是 Windows 平台固有差异**，涉及文件均为上游原版。**不要为本地通过而改动上游 factories**
- `npm run build`：成功（版本 1.4.6）
- `npm run lint`：**21 个 error 为上游原版代码在 `eslint-plugin-obsidianmd` ^0.3.0 下的遗留**（上游 1.4.6 的 lint 修复本身不完整）。要求：**本次改动不新增 error**

判断回归的黄金法则：先确认失败的文件是上游原版还是本地合并代码——上游原版文件失败多为平台/上游问题，本地合并代码失败才是需要修的回归。

## 升级后必须确认的结果（统一清单）

1. 图片右键菜单仍正常工作
2. 非图片附件仍可被右键识别，5 个动作齐全（Name / Copy file / Show in navigation / Show in system explorer / Delete file and link）
3. 附件重命名保留原始扩展名
4. 删除附件移除当前笔记链接并将文件移入回收站
5. markdown 相对路径：同级/子目录不带 `./`，父目录保留 `../`
6. wikilink 相对路径保持原行为
7. 点击笔记内图片打开放大预览
8. 点击空白遮罩关闭预览
9. 预览内滚轮缩放、左键拖动平移
10. 预览缩放和拖动不修改笔记中的图片尺寸链接
11. 关闭 `enableImageClickZoom` 后点击图片不再打开预览
12. 关闭 `isImageResizeEnbaled` 后点击放大仍可独立工作
13. 表格内图片（`.cm-table-widget`）点击不触发 lightbox，保留 Obsidian 原生处理
14. 命令面板插入附件仍触发重命名/压缩（create 兜底可用）
15. 外部同步工具（git/syncthing/remotely-save）创建的文件不被误处理
16. 相关测试文件全部通过（ContextMenu / ImageResizer / CreateHandler / LinkFormatter）
17. 项目可以正常构建

## 发生冲突时的优先级

1. 保证项目能正常构建
2. 保证图片原有菜单与交互不回归
3. 保证附件菜单增强功能仍可用
4. 保证 lightbox 不影响拖拽/滚轮 resize 的尺寸写回逻辑
5. 保证 create 兜底仍可用（命令面板插入附件被处理）
6. 保证 markdown 相对路径规则不变
7. 保证相关测试通过
8. 最后再考虑是否需要进一步重构

如果上游改动很大，先输出：哪些结构发生了变化 / 哪些实现可直接保留 / 哪些逻辑需要重新适配 / 哪些行为需要手动验证，再动手。

## 交付时应输出什么

1. 修改了哪些源码文件
2. 保留了哪些现有能力（对照能力总表逐项说明）
3. 哪些逻辑因为上游更新做了适配调整
4. 跑了哪些测试或构建命令
5. 还存在哪些残余风险

## 给 AI 的一句话任务模板

```text
请基于当前项目根目录中的源码仓库工作。目标不是从头重写功能，而是在上游更新的基础上，
保留并适配本 fork 全部 4 项自定义能力（非图片附件右键菜单、markdown 相对路径去 ./、
图片点击放大预览 enableImageClickZoom、命令面板插入附件 create 兜底）。

请执行 `AI升级说明.md` 中的「升级适配工作流程」：先备份（git tag backup-pre-<版本>-merge），
抓取上游并对比，按三方文件分类法合并（仅上游改→checkout 上游；仅本地改→保留；
两边都改→以上游结构为基底手动移植本地功能），然后对「自定义能力总表」逐项核对并移植，
最后运行第 4 步验证命令并对照「已知基线」判断是否回归。完成后按「交付时应输出什么」汇报。
```

---

## 变更记录（简史，细节以 git commit 为准）

| 日期 | 改动 | 涉及文件 | 验证 |
|------|------|---------|------|
| 2026-05-12 | markdown 相对路径去掉 `./` | `src/LinkFormatter.ts` + 2 个链接测试 | ContextMenu/ImageResizer 测试 + build |
| 2026-06-04 | lightbox 支持拖动平移；遮罩改浅（0.86→0.58） | `src/ImageResizer.ts`、`styles.css`、`ImageResizer.test.ts` | ImageResizer 测试 + build |
| 2026-08-18 | 命令面板插入附件 create 兜底（四重过滤 + 轮询链接证据） | `src/main.ts`、`tests/integration/main/CreateHandler.test.ts` | 全量 645 测试 + build |
| 2026-08-18 | 同步上游 1.4.6：popout 支持 / Obsidian 1.13 交互恢复 / Windows MAX_PATH 安全 / 事件生命周期治理；4 项能力全部保留 | ContextMenu、ImageResizer、main、ImageConverterSettings、styles、manifest/package/versions 等 43 个文件 | 全量 694 过 / 5 失败（Windows app:// 平台差异）+ build 1.4.6 |

### 2026-08-18 同步 1.4.6 详情（本次合并要点备忘）

上游 1.4.3 → 1.4.6 带来的新内容：

- **popout 窗口支持**：`document` → `activeDocument`/`ownerDocument`，跨窗口 `instanceOf()`，ContextMenu 多 document 监听（`window-open`）
- **Obsidian 1.13 图片交互恢复**：ImageResizer 生命周期重构为 workspace 事件驱动（`file-open`/`active-leaf-change` → attach/detach）、原生点击抑制、表格图片 widget 保留原生处理
- **Windows 路径安全**：`FolderAndFilenameManagement.getWindowsPathLengthViolation()` + `main.ts validateAttachmentPath()`（MAX_PATH 260）
- **事件生命周期治理**：ContextMenu 事件注册移到 transient Menu 上、`menu.onHide`、图片复制 `onload/onerror` 清理
- **新测试**：ImageResizerLifecycle、WindowsPathLimit、ImageAlignment、NativeImageSelectionStyles 等
- 依赖升级：`eslint-plugin-obsidianmd` ^0.1.9 → ^0.3.0

本次合并中的适配调整（保持自定义能力的关键点）：

- `src/main.ts`：上游 `activeDocument`/生命周期重构/`validateAttachmentPath` + 本地 create 兜底 + `selfCreatedPaths` 6 处标记 + `enableImageClickZoom` 条件 3 处
- `src/ContextMenu.ts`：上游 popout/`menu.registerDomEvent` 结构 + 本地附件菜单（DOM 用 `ownerDocument`，监听用 `menu.registerDomEvent`；img 分支保持上游无条件 `showAtMouseEvent`，attachment 分支保留 hasItems 门控）
- `src/ImageResizer.ts`：上游生命周期重构 + 本地 lightbox。**`getImageTargetForClickZoom` 必须排除 `.cm-table-widget`**（否则上游 13.29d 测试失败）
- `src/ImageConverterSettings.ts`：`enableImageClickZoom` 设置 UI 放 section 头部（见能力 3 的位置敏感说明）
- 版本：manifest/package `1.4.6`，品牌标识（Image Converter Pro / xRyul (lhbacxc)）不变
