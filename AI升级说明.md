# Obsidian Image Converter Pro AI 升级说明

日期：2026-05-11

## 文档目的

这份文档是写给后续协助升级插件的 AI 的。

目标不是让 AI “重新实现一遍功能”，而是让 AI：

- 基于源码仓库继续维护
- 在上游插件更新后，迁移我们已经做过的附件右键菜单增强
- 尽量减少因为官方升级带来的回归问题

## 当前项目结构

当前目录下有两个和本任务相关的内容：

### 1. 已安装使用的旧 bundle 版本

路径：

- `D:\Desktop\ob插件优化2\image-converter`

说明：

- 这是已构建好的插件产物目录。
- 里面的 `main.js` 是旧的 bundle 文件。
- 这里曾经直接做过附件菜单增强，用来验证需求是否成立。

### 2. 当前维护用的源码仓库

路径：

- `D:\Desktop\ob插件优化2\obsidian-image-converter-pro`

说明：

- 这是 fork 后 clone 下来的完整源码仓库。
- 以后所有升级、迁移、适配，都应该优先基于这里的源码进行。
- 不要再长期直接修改 bundle 版 `main.js`，除非只是临时验证。

## 本次已经完成的源码迁移

附件右键菜单增强已经正式迁移到源码仓库中。

主要修改文件：

- `D:\Desktop\ob插件优化2\obsidian-image-converter-pro\src\ContextMenu.ts`
- `D:\Desktop\ob插件优化2\obsidian-image-converter-pro\tests\integration\ui\ContextMenu.test.ts`

这意味着：

- 相关能力现在已经存在于源码层
- 以后更新插件时，应迁移这部分源码补丁，而不是重新从文档猜测实现

## 当前自定义功能是什么

我们在官方插件基础上增加了“非图片附件右键菜单增强”。

目标行为：

- 图片附件继续使用原有图片菜单逻辑
- 非图片附件新增右键菜单，包含：
  - `Name`
  - `Copy file`
  - `Show in navigation`
  - `Show in system explorer`
  - `Delete file and link`

支持识别的非图片附件目标包括：

- `.file-embed`
- `.file-embed-title`
- `audio`
- `video`
- `iframe`

## 关键源码位置

如果 AI 需要理解或迁移这项能力，优先查看这些文件：

### 右键菜单主入口

- `D:\Desktop\ob插件优化2\obsidian-image-converter-pro\src\ContextMenu.ts`

重点关注的方法：

- `handleContextMenuEvent`
- `resolveImageFromTarget`
- `resolveAttachmentTarget`
- `createAttachmentContextMenuItems`
- `addAttachmentRenameInput`
- `renameAttachmentFile`
- `copyFileToSystemClipboard`
- `showFileInNavigation`
- `showFileInSystemExplorer`
- `deleteAttachmentAndLinkFromNote`

### 路径解析与安全重命名

- `D:\Desktop\ob插件优化2\obsidian-image-converter-pro\src\FolderAndFilenameManagement.ts`

重点关注的方法：

- `getImagePath`
- `sanitizeFilename`
- `safeRenameFile`

### 相关测试

- `D:\Desktop\ob插件优化2\obsidian-image-converter-pro\tests\integration\ui\ContextMenu.test.ts`

说明：

- 这里已经补了附件菜单相关测试
- 以后升级后如果行为变了，应优先修复源码，再更新测试

## AI 应该怎么做

## 总原则

请不要从头重写功能。

你要做的是：

- 识别我们相对官方源码做过哪些补丁
- 判断上游新版是否已经覆盖其中一部分
- 只迁移仍然需要保留的补丁

换句话说，目标是“补丁迁移”，不是“重新设计功能”。

## 推荐工作流程

### 第一步：确认当前仓库状态

先检查：

- 当前分支
- 本地是否有未提交改动
- `origin` 和 `upstream` 配置

推荐命令：

```powershell
git status --short --branch
git remote -v
git log --oneline --decorate -n 10
```

### 第二步：确认上游最新版本

如果已经配置了 `upstream`，先抓取上游最新代码。

如果没有配置 `upstream`，应先补上原作者仓库地址，再抓取。

推荐流程：

```powershell
git remote add upstream https://github.com/xRyul/obsidian-image-converter.git
git fetch upstream
git branch -a
```

### 第三步：先对比我们改过什么

升级前不要直接改文件。

先看当前 fork 中，附件菜单增强究竟改了什么，重点看：

- `src/ContextMenu.ts`
- `tests/integration/ui/ContextMenu.test.ts`

如果仓库里已经有对应 commit，优先查看 commit diff。

如果没有 commit，则直接查看当前工作树改动。

### 第四步：再对比上游新版

重点确认：

- 上游 `ContextMenu.ts` 结构是否有变化
- 图片右键菜单逻辑是否重构
- `FolderAndFilenameManagement.ts` 的路径解析与重命名逻辑是否变化
- 测试文件结构是否变化

如果上游已经新增了部分附件支持，先识别重叠部分，避免重复实现。

### 第五步：迁移补丁

迁移时遵守下面这些约束：

- 保持图片原有逻辑不变
- 只在右键入口增加附件分支
- 尽量复用官方已有路径解析和重命名逻辑
- 非图片附件新增的 5 个动作必须保留
- 测试必须同步更新

### 第六步：验证

迁移后至少运行：

```powershell
npm test
npx vitest run tests/integration/ui/ContextMenu.test.ts
npm run build
```

如果只想先快速验证附件菜单相关改动，至少运行：

```powershell
npx vitest run tests/integration/ui/ContextMenu.test.ts
npm run build
```

## 本次迁移已经验证通过的命令

在当前仓库中，以下命令已经跑通过：

```powershell
npx vitest run tests/integration/ui/ContextMenu.test.ts
npm run build
```

说明：

- `ContextMenu` 相关测试已通过
- 源码构建已通过

## AI 在升级时必须保留的功能点

升级后，请确认以下能力仍然存在：

1. 图片右键菜单仍正常工作
2. 非图片附件可被右键识别
3. 附件菜单仍有以下 5 项：
   - `Name`
   - `Copy file`
   - `Show in navigation`
   - `Show in system explorer`
   - `Delete file and link`
4. 重命名时保留原始扩展名
5. 删除附件时会移除当前笔记中的链接并将文件移入回收站
6. `ContextMenu` 相关测试通过
7. 项目可以正常构建

## AI 不应该做的事情

请不要做这些事：

- 不要直接只改 `image-converter/main.js` 作为长期方案
- 不要绕过源码层，只做 bundle 层修补
- 不要在没有对比上游结构的情况下重写整个 `ContextMenu.ts`
- 不要无故删除现有测试
- 不要为了通过测试而移除附件功能
- 不要修改与本次升级无关的大量模块

## 如果发生冲突，优先级怎么判断

优先级建议如下：

1. 保证插件能正常构建
2. 保证图片原有菜单不回归
3. 保证附件菜单增强功能仍可用
4. 保证测试通过
5. 最后再考虑是否需要进一步重构代码

如果上游改动很大，不要强行机械套 patch。

应该先输出：

- 哪些结构发生了变化
- 哪些补丁可以直接迁移
- 哪些补丁需要重新适配
- 哪些行为需要手动验证

## 交付时应输出什么

完成升级后，请输出：

1. 修改了哪些源码文件
2. 哪些旧补丁被保留
3. 哪些逻辑因为上游更新做了适配调整
4. 跑了哪些测试/构建命令
5. 还有哪些残余风险

## 可参考文档

如果需要了解旧 bundle 阶段的改动背景，可以参考：

- `D:\Desktop\ob插件优化2\image-converter-attachment-menu-change-log.md`

这份文档偏历史复盘和工作流说明。

如果需要理解这次功能目标，请优先以源码现状为准，不要只依赖历史文档猜实现。

## 给 AI 的一句话任务模板

可以直接使用下面这段作为升级任务提示：

```text
请基于 D:\Desktop\ob插件优化2\obsidian-image-converter-pro 这个源码仓库工作。

目标不是从头重写功能，而是把我们已经做过的“非图片附件右键菜单增强”补丁迁移到上游最新版。

请按以下顺序执行：
1. 检查当前 git 状态、分支、远程仓库
2. 对比当前 fork 与上游最新版
3. 识别 ContextMenu.ts 中已有的附件菜单增强补丁
4. 将这些补丁迁移到上游新版源码结构中
5. 保持图片原有逻辑不变
6. 同步更新 tests/integration/ui/ContextMenu.test.ts
7. 运行 npx vitest run tests/integration/ui/ContextMenu.test.ts
8. 运行 npm run build
9. 输出修改文件、验证结果、风险点

不要长期直接修改 bundle 版 main.js，优先维护源码仓库。
```
