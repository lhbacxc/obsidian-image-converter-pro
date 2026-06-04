# Obsidian Image Converter Pro AI 升级说明

日期：2026-05-12

## 文档目的

这份文档是写给后续协助维护、升级、适配本插件源码仓库的 AI 的。

目标不是让 AI 重新设计功能，也不是让 AI 依赖历史目录结构推断实现，而是让 AI：

- 直接基于当前源码仓库继续维护
- 在上游更新后，识别并保留我们已经验证通过的自定义能力
- 优先根据当前源码、测试和构建结果完成适配
- 尽量减少升级带来的回归问题

## 根目录约定

从现在开始，`obsidian-image-converter-pro` 本身就是项目根目录。

后续所有路径、命令、文件说明，都默认基于项目根目录展开：

- 不要假设仓库位于某个固定磁盘路径
- 不要在文档、脚本说明、任务提示中写死绝对路径
- 本文档中出现的文件路径，均为相对项目根目录的相对路径

例如：

- `src/ContextMenu.ts`
- `src/FolderAndFilenameManagement.ts`
- `tests/integration/ui/ContextMenu.test.ts`

## 当前已落地并需持续保留的能力

当前自定义能力包括：

- 非图片附件右键菜单增强
- 图片点击放大预览

### 非图片附件右键菜单增强

目标行为：

- 图片附件继续使用原有图片菜单逻辑
- 非图片附件可被右键识别并显示附件菜单
- 非图片附件菜单必须包含以下 5 项：
  - `Name`
  - `Copy file`
  - `Show in navigation`
  - `Show in system explorer`
  - `Delete file and link`

当前支持识别的非图片附件目标包括：

- `.file-embed`
- `.file-embed-title`
- `audio`
- `video`
- `iframe`

### 图片点击放大预览

目标行为：

- 笔记内图片支持左键点击放大预览
- 点击放大后的遮罩空白区域可关闭预览并回到笔记
- 放大预览中使用鼠标滚轮可继续放大或缩小图片
- 放大预览中可按住鼠标左键拖动图片，查看被放大后的具体细节
- 预览中的滚轮缩放和鼠标拖动只改变临时显示，不修改图片文件、不写回 markdown 链接尺寸、不写入对齐缓存

当前实现约束：

- 由独立设置项 `enableImageClickZoom` 控制，默认开启
- 即使关闭拖拽/滚轮 resize 总开关，点击放大预览仍可独立工作
- 跳过 Excalidraw 图片、`.map-view-main`、resize handle、编辑按钮和 lightbox 自身
- 不处理非图片附件，避免影响附件菜单增强
- 不修改 `src/ContextMenu.ts` 的右键菜单逻辑
- lightbox 遮罩应保持半透明但不要过深，当前为 `rgba(0, 0, 0, 0.58)`
- lightbox 内部拖动使用临时 `translate(...) scale(...)`，不得复用笔记内图片 resize 写回逻辑

当前 lightbox 相关 DOM/CSS 类：

- `.image-converter-lightbox-overlay`
- `.image-converter-lightbox-image`
- `.image-converter-lightbox-image-dragging`
- `.image-converter-lightbox-open`

## 关键源码位置

如果 AI 需要理解、适配或迁移这些能力，优先查看以下文件。

### 右键菜单主入口

- `src/ContextMenu.ts`

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

- `src/FolderAndFilenameManagement.ts`

重点关注的方法：

- `getImagePath`
- `sanitizeFilename`
- `safeRenameFile`

### 相关测试

- `tests/integration/ui/ContextMenu.test.ts`

说明：

- 这里已经覆盖了附件菜单相关测试
- 升级后如果行为变化，应优先修复源码，再按需要调整测试

### 图片点击放大预览与图片交互

- `src/ImageResizer.ts`
- `src/ImageConverterSettings.ts`
- `src/main.ts`
- `styles.css`

重点关注的方法和字段：

- `ImageConverterSettings.enableImageClickZoom`
- `DEFAULT_SETTINGS.enableImageClickZoom`
- `ImageResizer.attachView`
- `ImageResizer.getImageTargetForClickZoom`
- `ImageResizer.handleImageClickCapture`
- `ImageResizer.openImageLightbox`
- `ImageResizer.closeImageLightbox`
- `ImageResizer.handleLightboxWheel`
- `ImageResizer.handleLightboxImageMouseDown`
- `ImageResizer.handleLightboxImageMouseMove`
- `ImageResizer.handleLightboxImageMouseUp`
- `ImageResizer.applyLightboxScale`
- `ImageResizer.handleMouseWheel`

说明：

- `ImageResizer` 现在不只负责拖拽/滚轮 resize，也负责图片点击放大预览
- `src/main.ts` 中初始化 `ImageResizer` 的条件必须同时考虑 `isImageResizeEnbaled` 和 `enableImageClickZoom`
- `handleMouseWheel` 必须保留 `isImageResizeEnbaled` 总开关判断，避免用户只启用点击放大时误触笔记内图片尺寸写回
- lightbox 内部滚轮缩放和笔记内滚轮 resize 是两条不同路径，不要混在一起
- lightbox 内部鼠标拖动只应更新预览图位移，不应触发笔记内 drag resize 或 markdown 链接尺寸更新

### 图片点击放大预览相关测试

- `tests/integration/ui/ImageResizer.test.ts`

说明：

- 已覆盖点击打开、点击空白关闭、预览滚轮缩放、预览左键拖动平移、设置关闭时不打开、关闭 resize 总开关时仍可点击放大
- 升级后如果图片交互或 `ImageResizer` 生命周期变化，应同步运行并修复这些测试

## AI 工作原则

### 总原则

不要从头重写功能。

你要做的是：

- 识别当前源码仓库中已经存在的附件菜单增强实现
- 对比上游新版本是否已覆盖其中一部分
- 只保留仍然必要的改动
- 基于验证结果继续操作源码，而不是依赖历史背景猜实现

换句话说，目标是“基于当前已验证实现做适配”，不是“重新发明一套功能”。

### 源码优先

长期维护必须优先修改源码文件，而不是把直接修改构建产物作为正式方案。

如需定位功能，请优先查看：

- `src/ContextMenu.ts`
- `src/FolderAndFilenameManagement.ts`
- `src/ImageResizer.ts`
- `src/ImageConverterSettings.ts`
- `tests/integration/ui/ContextMenu.test.ts`
- `tests/integration/ui/ImageResizer.test.ts`

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

如果没有配置 `upstream`，先补充上游仓库地址，再抓取。

推荐流程：

```powershell
git remote add upstream https://github.com/xRyul/obsidian-image-converter.git
git fetch upstream
git branch -a
```

### 第三步：先对比当前源码里已经有什么

升级前不要直接改文件。

先确认当前仓库里，附件菜单增强具体落在哪些源码和测试上，重点看：

- `src/ContextMenu.ts`
- `src/FolderAndFilenameManagement.ts`
- `tests/integration/ui/ContextMenu.test.ts`

如果仓库里已经有对应 commit，优先查看 commit diff。

如果没有 commit，则直接查看当前工作树改动。

### 第四步：再对比上游新版结构

重点确认：

- 上游 `src/ContextMenu.ts` 结构是否变化
- 图片右键菜单逻辑是否重构
- `src/FolderAndFilenameManagement.ts` 的路径解析与重命名逻辑是否变化
- `tests/integration/ui/ContextMenu.test.ts` 的测试结构是否变化

如果上游已经新增了部分附件支持，先识别重叠部分，避免重复实现。

### 第五步：做最小必要适配

适配时遵守以下约束：

- 保持图片原有逻辑不变
- 只在需要的位置保留附件分支
- 尽量复用现有路径解析和重命名逻辑
- 非图片附件新增的 5 个动作必须保留
- 图片点击放大预览必须保持独立设置开关，不要强制依赖拖拽/滚轮 resize 总开关
- lightbox 预览滚轮缩放不得写回 markdown 链接尺寸
- lightbox 预览鼠标拖动不得写回 markdown 链接尺寸，也不得写入对齐缓存
- 测试必须同步验证

### 第六步：验证

适配后至少运行：

```powershell
npx vitest run tests/integration/ui/ContextMenu.test.ts
npx vitest run tests/integration/ui/ImageResizer.test.ts tests/integration/ui/ContextMenu.test.ts
npm run build
```

如果改动范围较大，建议再运行：

```powershell
npm test
```

## 当前已验证通过的命令

以下命令已经在当前仓库中验证通过，可作为后续工作的基线验证：

```powershell
npx vitest run tests/integration/ui/ContextMenu.test.ts
npx vitest run tests/integration/ui/ImageResizer.test.ts tests/integration/ui/ContextMenu.test.ts
npm test
npm run build
```

这说明：

- `ContextMenu` 相关测试已通过
- `ImageResizer` 图片点击放大预览及旧图片缩放相关测试已通过
- 全量测试已通过
- 当前源码可以正常构建

## 升级后必须确认的结果

升级或适配完成后，必须确认以下事项仍然成立：

1. 图片右键菜单仍正常工作
2. 非图片附件仍可被右键识别
3. 附件菜单仍保留以下 5 项：
   - `Name`
   - `Copy file`
   - `Show in navigation`
   - `Show in system explorer`
   - `Delete file and link`
4. 重命名时保留原始扩展名
5. 删除附件时会移除当前笔记中的链接并将文件移入回收站
6. 点击笔记内图片可以打开放大预览
7. 点击放大预览的空白遮罩区域可以关闭预览
8. 放大预览中鼠标滚轮可以临时放大/缩小图片
9. 放大预览中鼠标左键拖动图片可以临时平移查看细节
10. 放大预览缩放和拖动不会修改笔记中的图片尺寸链接
11. 关闭 `enableImageClickZoom` 后点击图片不再打开预览
12. `tests/integration/ui/ContextMenu.test.ts` 相关测试通过
13. `tests/integration/ui/ImageResizer.test.ts` 相关测试通过
14. 项目可以正常构建

## AI 不应该做的事情

请不要做这些事：

- 不要脱离当前源码现状，凭历史印象重写功能
- 不要在没有对比上游结构的情况下重写整个 `src/ContextMenu.ts`
- 不要只为通过测试而移除附件功能
- 不要无故删除现有测试
- 不要修改与本次升级无关的大量模块
- 不要在文档、说明或任务提示里继续依赖旧目录结构
- 不要把图片点击放大预览实现成修改 markdown 尺寸的持久化缩放
- 不要把 lightbox 预览拖动实现成笔记内图片尺寸拖拽
- 不要让 lightbox 内部图片点击再次触发笔记图片点击逻辑

## 发生冲突时的优先级

优先级建议如下：

1. 保证项目能正常构建
2. 保证图片原有菜单不回归
3. 保证附件菜单增强功能仍可用
4. 保证图片点击放大预览不影响拖拽/滚轮 resize 的尺寸写回逻辑
5. 保证相关测试通过
6. 最后再考虑是否需要进一步重构

如果上游改动很大，不要机械套用旧改动。

应先输出：

- 哪些结构发生了变化
- 哪些现有实现可以直接保留
- 哪些逻辑需要重新适配
- 哪些行为需要手动验证

## 交付时应输出什么

完成升级后，请输出：

1. 修改了哪些源码文件
2. 保留了哪些现有能力
3. 哪些逻辑因为上游更新做了适配调整
4. 跑了哪些测试或构建命令
5. 还存在哪些残余风险

## 给 AI 的一句话任务模板

可以直接使用下面这段作为升级任务提示：

```text
请基于当前项目根目录中的源码仓库工作。

目标不是从头重写功能，而是基于当前已经验证通过的实现，保留并适配“非图片附件右键菜单增强”能力。

请按以下顺序执行：
1. 检查当前 git 状态、分支、远程仓库
2. 对比当前仓库与上游最新版
3. 识别 `src/ContextMenu.ts`、`src/FolderAndFilenameManagement.ts`、`tests/integration/ui/ContextMenu.test.ts` 中与附件菜单增强相关的现有实现
4. 将仍然必要的实现适配到上游最新源码结构中
5. 保持图片原有逻辑不变
6. 同步验证 `tests/integration/ui/ContextMenu.test.ts`
7. 运行 `npx vitest run tests/integration/ui/ContextMenu.test.ts`
8. 运行 `npm run build`
9. 输出修改文件、验证结果、风险点

不要依赖旧目录结构、历史 bundle 改法或绝对路径；优先依据当前源码与验证结果完成工作。
```

## 2026-05-12 变更记录：markdown 相对路径去掉 `./`

### 现象

当插件启用“markdown 形式的相对路径”插入附件时，当前实现会生成：

```md
![](./附件/【003】Obsidian_网页访问_202605121005.jpeg)
```

但 Obsidian 默认插入同类附件时，路径通常为：

```md
![](附件/【003】Obsidian_网页访问_202605121005.jpeg)
```

两者差异仅在于前缀 `./`。

### 本次调整

- 修改 `src/LinkFormatter.ts`
- 仅调整 `markdown + relative` 的输出规则
- 当相对路径不需要使用 `../` 回退父目录时，不再强制补 `./`
- 因此子路径和同级路径会输出为：
  - `附件/xxx.png`
  - `image.png`
- 如果路径本来就需要回退父目录，仍然保持：
  - `../images/xxx.png`

### 明确保留的行为

- `wikilink` 的相对路径逻辑不变，仍可保留显式 `./`
- 绝对路径逻辑不变
- markdown 路径中的空格仍继续编码为 `%20`
- 本次不处理插入后文件重命名问题

### 这次改动涉及的文件

- `src/LinkFormatter.ts`
- `tests/unit/links/LinkFormatter.test.ts`
- `tests/unit/links/LinkFormatSettings.mapping.test.ts`

### 升级时的注意事项

以后如果上游再次修改链接生成逻辑，优先检查 `src/LinkFormatter.ts` 中的相对路径格式化分支，确认以下约束仍然成立：

1. `wikilink + relative` 可以保留 `./`
2. `markdown + relative` 在子路径或同级路径下不要补 `./`
3. `markdown + relative` 在父级路径下必须继续保留 `../`

### 本次新增/更新的验证点

- markdown 相对路径指向同级文件时，不带 `./`
- markdown 相对路径指向当前笔记下的子目录附件时，不带 `./`
- markdown 相对路径指向父目录文件时，继续保留 `../`
- wikilink 相对路径保持原行为

## 2026-06-04 变更记录：图片点击放大预览支持拖动平移

### 现象

图片点击放大预览已经支持鼠标滚轮临时缩放，但放大后只能围绕中心查看，无法通过鼠标拖动移动预览图来查看图片局部细节。

同时原 lightbox 遮罩颜色较深，背景被压暗过多。

### 本次调整

- 修改 `src/ImageResizer.ts`
- 修改 `styles.css`
- 更新 `tests/integration/ui/ImageResizer.test.ts`
- lightbox 遮罩从 `rgba(0, 0, 0, 0.86)` 调整为 `rgba(0, 0, 0, 0.58)`
- lightbox 预览图支持鼠标左键按住拖动平移
- lightbox 预览图的临时显示状态统一使用 `translate(...) scale(...)`

### 明确保留的行为

- 点击笔记内图片仍通过 `enableImageClickZoom` 独立控制，默认开启
- 关闭拖拽/滚轮 resize 总开关时，点击放大预览仍可独立工作
- 点击 lightbox 空白遮罩区域仍可关闭预览
- lightbox 内鼠标滚轮缩放仍只改变临时显示
- lightbox 内鼠标左键拖动仍只改变临时显示
- 预览缩放和拖动都不修改图片文件、不写回 markdown 链接尺寸、不写入对齐缓存
- 笔记内图片滚轮 resize 仍必须受 `isImageResizeEnbaled` 总开关控制

### 这次改动涉及的文件

- `src/ImageResizer.ts`
- `styles.css`
- `tests/integration/ui/ImageResizer.test.ts`

### 升级时的注意事项

以后如果上游修改 `src/ImageResizer.ts` 的事件注册或 lightbox 结构，优先确认以下约束仍然成立：

1. lightbox 内部滚轮缩放和笔记内滚轮 resize 是两条不同路径
2. lightbox 内部鼠标拖动平移不触发笔记内 drag resize
3. lightbox 临时 transform 包含位移和缩放，即 `translate(...) scale(...)`
4. lightbox 内部交互不会写回 markdown 图片尺寸链接
5. `.image-converter-lightbox-image-dragging` 仅表示预览图拖动状态

### 本次新增/更新的验证点

- 点击图片后可以打开 lightbox 预览
- 点击 lightbox 空白遮罩区域可以关闭预览
- lightbox 内滚轮缩放会更新预览图 `transform`
- lightbox 内鼠标左键拖动会更新预览图 `transform`
- lightbox 内滚轮缩放和拖动不会修改原笔记图片尺寸
- 关闭 `enableImageClickZoom` 后点击图片不打开预览
- 关闭 `isImageResizeEnbaled` 后点击放大预览仍可独立工作

### 本次已验证命令

```powershell
npx vitest run tests/integration/ui/ImageResizer.test.ts
npm run build
git diff --check
```
