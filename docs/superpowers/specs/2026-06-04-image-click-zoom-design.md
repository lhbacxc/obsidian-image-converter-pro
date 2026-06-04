# 图片点击放大预览设计

日期：2026-06-04

## 背景

当前插件已经在 `src/ImageResizer.ts` 中集中处理笔记内图片的 hover、拖拽缩放、滚轮缩放和点击抑制逻辑。新增“点击图片放大”功能应复用这套 Markdown 视图生命周期，避免在 `src/main.ts` 中增加新的全局图片监听，减少与现有右键菜单和附件菜单增强的冲突。

## 目标

- 点击笔记内图片后，打开全屏图片预览遮罩。
- 点击遮罩空白区域后，关闭预览并回到笔记。
- 预览打开后，鼠标滚轮可继续放大或缩小图片。
- 预览打开后，可按住鼠标左键拖动图片，查看被放大后的局部细节。
- 预览缩放和拖动只影响临时显示，不修改图片文件，不写回 markdown 链接尺寸，也不写入对齐缓存。
- 提供独立设置开关，便于用户在与 Obsidian 原生行为或其它插件冲突时关闭。

## 非目标

- 不实现键盘快捷键、上一张/下一张切换或工具栏。
- 不改变已有拖拽缩放、滚轮写回尺寸、图片右键菜单和非图片附件右键菜单行为。
- 不为非图片附件提供预览。

## 设置项

新增设置字段 `enableImageClickZoom`：

- 默认值：`true`
- 展示位置：`Drag & scroll resize` 设置分组内
- 作用：控制点击图片是否打开放大预览

该设置虽然放在图片交互分组内，但不依赖 `isScrollResizeEnabled`。也就是说，即使用户关闭“scroll-wheel resize”，仍可以在放大预览中使用滚轮临时缩放，因为预览缩放不会写回笔记。

## 交互设计

1. 用户在 Markdown 预览或 Live Preview 中左键点击图片。
2. 插件校验目标是否为可处理图片：
   - 必须位于当前 Markdown 视图容器内。
   - 跳过 `.image-resize-handle`、`.edit-block-button` 等 resize 或编辑控件。
   - 跳过 `.map-view-main`。
   - 跳过 Excalidraw 图片。
3. 创建 `.image-converter-lightbox-overlay` 遮罩并插入 `document.body`。
4. 在遮罩中创建独立的 `img` 元素，使用原图片的 `src` 和 `alt`。
5. 点击遮罩空白处关闭预览；点击预览图片本身不关闭。
6. 在遮罩中滚动鼠标滚轮时，根据 `deltaY` 调整临时 `scale`。
7. 按住预览图片左键拖动时，根据鼠标位移调整临时 `translate`。
8. 关闭预览时移除遮罩、释放引用、清理 body 状态类。

## 实现位置

主要改动：

- `src/ImageResizer.ts`
  - 新增 click 事件处理，用于打开 lightbox。
  - 新增 lightbox DOM 创建、关闭、滚轮缩放和拖动平移方法。
  - 在 `onunload()`、`onLayoutChange()` 中关闭已有预览，避免视图切换后残留。
- `src/ImageConverterSettings.ts`
  - 新增 `ImageConverterSettings.enableImageClickZoom` 字段。
  - 在 `DEFAULT_SETTINGS` 中设置默认值。
  - 在图片 resize/interaction 设置分组中添加 toggle。
- `styles.css`
  - 添加遮罩、预览图片和打开状态样式。
- `tests/integration/ui/ImageResizer.test.ts`
  - 覆盖点击打开、点击空白关闭、滚轮缩放、拖动平移、设置关闭时不打开。

## 风险与处理

- 与 Obsidian 原生图片点击选择冲突：使用独立设置开关，并跳过 resize 手柄与编辑按钮。
- 与拖拽缩放冲突：点击 resize handle 不打开预览；正在 resize 时不打开预览。
- 与预览拖动冲突：lightbox 内部拖动只更新预览图 `translate(...) scale(...)`，不触发笔记内 drag resize。
- 与右键菜单冲突：仅处理左键 click，不改 `src/ContextMenu.ts`。
- 与附件菜单增强冲突：仅处理 `HTMLImageElement`，不处理 `.file-embed`、`audio`、`video`、`iframe`。

## 验证计划

运行以下命令：

```powershell
npx vitest run tests/integration/ui/ImageResizer.test.ts tests/integration/ui/ContextMenu.test.ts
npm run build
```

验证点：

- 点击图片会创建预览遮罩。
- 点击遮罩空白处会关闭预览。
- 预览中滚轮可改变图片 `transform` 中的 `scale(...)`。
- 预览中左键拖动可改变图片 `transform` 中的 `translate(...)`。
- 预览中的滚轮缩放和拖动平移都不会修改原笔记图片尺寸。
- 关闭 `enableImageClickZoom` 后点击图片不打开预览。
- `ContextMenu` 测试仍通过，说明右键菜单和附件菜单增强没有明显回归。

## 自检

- 未使用 `TBD` 或 `TODO` 占位。
- 设计范围限定在点击预览、临时缩放和拖动平移，不包含切图或工具栏。
- 所有文件路径均为相对项目根目录路径。
- 交互与现有 resize 写回逻辑明确分离。
