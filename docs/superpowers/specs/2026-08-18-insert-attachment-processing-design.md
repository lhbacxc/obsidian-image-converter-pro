# 设计文档：命令面板插入附件触发重命名/压缩（方案 A：create 兜底）

日期：2026-08-18
状态：已确认，待实现

## 背景

### 问题

用户通过**拖入**图片等附件到 Obsidian 时，本插件会正常执行重命名 + 压缩；但通过**命令面板「插入附件」(Insert attachment)** 插入时，不会触发任何处理。

### 根因（已定位）

1. 插件当前（本地 1.4.3 / 上游最新 1.4.6）只注册了 `editor-drop` 与 `editor-paste` 两个事件（`src/main.ts` `dropPasteRegisterEvents`）。
2. Obsidian「命令面板插入附件」内部走 `vault.createBinary()` 复制文件到默认附件位置 + 插入链接，**不触发** `editor-drop`/`editor-paste`。Obsidian 官方无 insert-attachment 事件可监听，只能靠 `vault.on('create')` 兜底（Obsidian 论坛已确认）。
3. 旧版插件（2025-02-01 `ca5e2c3 refactor` 之前）有 `registerCreateEventAfterAction()` 通过 `vault.on('create')` 兜底处理，该机制在 refactor 中被整体删除，从此命令面板插入不再被处理。上游 1.4.4~1.4.6 未恢复。

### 方案选择

选**方案 A：恢复 `vault.on('create')` 常驻兜底**，而非 monkey-patch Obsidian 内部方法（custom-attachment-location 做法）：

- create 依赖公开 API，Obsidian 升级风险低；
- 本项目处理链（压缩/转换/重命名）比 custom-attachment-location 重得多，monkey-patch 的脆弱性在 Obsidian 1.13 大改版窗口期维护负担高；
- 代价是需要做好来源过滤（区分用户插入 vs 外部同步）与链接回写（Obsidian 已插入链接）。

## 需求决策（已与用户确认）

| 决策点 | 结论 |
|---|---|
| 触发场景 | 只处理外部文件复制进库（create 事件天然覆盖）；库内已有文件插链接不触发 create，不处理 |
| 来源过滤 | 链接证据 + 同步路径黑名单 + 排除插件自建文件 |
| 交互方式 | 与拖拽行为一致（复用 `modalBehavior`：always/ask 弹 PresetSelectionModal，否则默认预设） |
| 链接回写 | 只回写活动笔记中指向旧路径的引用 |
| 平台范围 | 桌面 + 移动都支持（移动端无 editor-drop，create 是主要路径） |
| 代码组织 | 抽取共享管道，drop/paste/create 三路复用 |

## 架构

新增一条事件路径，与现有 drop/paste 并列，三路共用同一套处理管道：

```
editor-drop ──┐
editor-paste ─┼──▶ 共享管道（预设选择 → determineDestination → 压缩 → 建文件 → 处理链接）
vault.create ─┘     （create 路径的"链接处理"= 回写，drop/paste = 光标处插入）
```

## 新增组件

全部位于 `src/main.ts`，不新建文件。

| 组件 | 职责 |
|---|---|
| `registerVaultCreateHandler()` | onload 时注册常驻 `vault.on('create')`（桌面+移动都注册，区别于 drop/paste 仅桌面） |
| `handleFileCreated(file: TFile)` | create 主入口：四重过滤 → 轮询等 metadataCache 链接证据 → 调度共享管道 → renameFile/modifyBinary 写回 |
| `selfCreatedPaths: Set<string>` | 防二次处理：插件自己 `createBinary` 建的文件也会触发 create 事件，建前记录路径、handler 里跳过 |
| `isSyncPath(path)` | 同步路径黑名单（提取旧版 `isExternalOperation` 的 syncPatterns：`.git` / `syncthing` / `remotely-save` / `sync-conflict` 等） |
| `getSelectedPresets()` | 从 handleDrop/handlePaste 抽出的共享预设选择逻辑（modalBehavior 弹窗/默认预设），三路复用 |
| `noteReferencesFile(note, file)` | 链接证据：getFirstLinkpathDest 精确解析，解析为空时降级为链接文本匹配文件名 |
| `processingPaths: Set<string>` | 防并发：同一文件处理中则跳过 |

## create 路径数据流

```
vault.on('create', file)
 ├─ 过滤1：TFile + 支持格式 + 非 neverProcess 模式
 ├─ 过滤2：非同步路径（isSyncPath）
 ├─ 过滤3：非插件自建（selfCreatedPaths）
 ├─ 过滤4（链接证据）：轮询等待活动笔记 metadataCache 出现指向该文件的引用
 │    （create 事件先于 Obsidian 插入链接触发，metadataCache 更新有延迟，
 │     每 150ms 检查一次，最长 3 秒；匹配先 getFirstLinkpathDest 精确解析，
 │     解析为空时降级为链接文本匹配文件名 basename/name）
 │    ├─ 否 → 忽略（外部同步/其他方式创建，不动）
 │    └─ 是 → 共享管道处理：
 │         ├─ 预设选择（与拖拽一致：modalBehavior 弹窗/默认预设）
 │         ├─ determineDestination（新文件名/目录，按预设规则）
 │         ├─ processImage 压缩
 │         ├─ 路径未变 → modifyBinary 写回压缩内容
 │         │  路径变化 → fileManager.renameFile（Obsidian 原生自动更新所有笔记
 │         │              中的引用链接）+ modifyBinary 写回压缩内容
 │         └─ showSizeComparisonNotification
 └─ 全程 try/catch，失败留原文件、Notice 报错
```

## 与 drop/paste 的差异点（需适配）

1. **入参不同**：`FolderAndFilenameManagement.determineDestination` 签名是 `(file: File, activeFile, ...)`，create 路径拿到的是 TFile。处理时从 TFile 读二进制构造 File 壳（含 name/type）再传入。
2. **链接处理不同**：drop/paste 是自己 `insertLinkAtCursorPosition`；create 是 Obsidian 已写入链接 → 用 `fileManager.renameFile` 让 Obsidian 原生同步所有笔记中的引用（比手写链接回写更可靠），随后 `modifyBinary` 覆盖压缩内容。
3. **时序**：create 触发时 Obsidian 尚未插入链接 → 轮询等待 metadataCache 链接证据（最长 3 秒），而非固定延迟。

## 错误处理

- 所有步骤 try/catch，失败给 `Notice` + `console.error`，**不删除原文件**，不阻断 Obsidian 自身流程。
- 处理中的文件加标记（`processingPaths` Set），防止并发重复触发。
- 复用现有 `FolderAndFilenameManagement` 的 determineDestination/ensureFolderExists/handleNameConflicts。

## 测试

新增 `tests/integration/main/CreateHandler.test.ts`，覆盖：

- create 触发且链接证据成立 → 走处理管道，文件被重命名/压缩
- 过滤：非图片 / neverProcess 模式 / 同步路径 / 插件自建文件 → 不处理
- 链接证据不成立（活动笔记无引用）→ 不处理
- 处理完成后文件被 renameFile 到新路径且 modifyBinary 写回压缩内容
- 桌面 + 移动端都注册 create 监听
- 处理失败时原文件保留

复用现有测试基建：`tests/factories/obsidian.ts` 的 `fakeApp` / `fakeVault` / `fakeTFile`，参考 `tests/integration/main/DropPasteHandlers.test.ts` 的写法。

## 验证命令

```powershell
npx vitest run tests/integration/main/CreateHandler.test.ts
npm test
npm run build
```

## 风险与注意

1. **误处理外部同步**：create 事件同样捕获 git/Syncthing/Obsidian Sync 拉入的文件。过滤顺序务必：同步路径黑名单 + 链接证据双重把关，链接证据是"确实是用户插入"的强信号。
2. **防二次处理**：插件自身 `createBinary`（drop/paste 路径）触发 create 事件，`selfCreatedPaths` 必须在 createBinary 前记录路径。
3. **时序竞态**：实测确认 create 事件先于链接插入，必须轮询等待链接证据（最长 3 秒）；若 3 秒内未出现则忽略（宁可漏处理不可误处理）。
4. **共享管道抽取为重构**：需保证 handleDrop/handlePaste 行为不变，抽取后先跑现有 `tests/integration/main/` 与 `npm test` 确认无回归。
5. **链接证据降级匹配**：metadataCache 尚未完成路径解析时，直接匹配链接文本中的文件名 basename/name；实测该降级路径有效（中文文件名也能命中）。
