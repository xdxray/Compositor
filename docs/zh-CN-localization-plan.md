# Compositor 简体中文本地化方案

> 分支：`zh-cn`（基于 `upstream/main`，跟踪 `origin/xdxray/Compositor`）
> 目标：自用中文界面，本地长期维护，不指望推回上游（作者目前不接受翻译 PR）。

## 1. 现状调研（事实依据）

| 项 | 发现 | 来源 |
|---|---|---|
| 部署目标 | `MACOSX_DEPLOYMENT_TARGET = 26.0` | `project.pbxproj` |
| Swift 版本 | `SWIFT_VERSION = 5.0` | `project.pbxproj` |
| 本地化基础设施 | 零：无 `.lproj`、无 `.strings`、无 `.xcstrings`、无 `NSLocalizedString` | 全仓搜索 |
| 代理语言 | 全局 git 代理已改为 `127.0.0.1:7897`（Clash Verge） | 已配置 |
| UI 文案分布 | 约 43 个 Swift 文件、数百处硬编码字符串 | 全仓统计 |

### 关键机制发现（决定方案形态）

SwiftUI 的 `Text("Undo")` / `Button("新建画布…")` / `Toggle("色调")` 这些**字面量字符串其实是 `LocalizedStringKey`**，SwiftUI 会自动去本地化表查表。因此：

- **A 类（自动本地化，几乎不改代码）**：直接字面量传给 `Text`/`Button`/`Toggle`/`Picker` 的标题。只要建好本地化表并填入翻译，运行在中文环境下即显示中文。
- **B 类（需改代码）**：
  1. `FilterSheet.swift:328` 的 `control(_ title: String, ...)` 参数类型是 `String`（非 `LocalizedStringKey`），经它传入的滑块标题（Exposure/Offset/Gamma/Reds…）**不自动本地化**。
  2. 枚举 `rawValue` 显示：`Filters.swift` 的 `FilterKind`（"Gaussian Blur"等）、`BackgroundQuality`（"Basic"/"Advanced"）——动态字符串，需单独映射。
  3. `.help("...")` tooltip 属 `LocalizedStringKey`，会自动查表，作为次要项处理。
- **不翻译**：动态内容（文件路径 `url.lastPathComponent`、历史名 `undoName` 等）。

## 2. 技术方案

采用 **String Catalog（`Localizable.xcstrings`）**——Xcode 15+ 原生、macOS 13+ 运行时支持，本项目部署 26 完全满足，且 Xcode 能自动管理 key、支持多语言与变体。

### 2.1 基础设施（一次性）

1. 新增 `Compositor/Localizable.xcstrings`，developmentRegion 保持 `en`，添加 `zh-Hans` 语言。
2. `project.pbxproj`：`knownRegions` 由 `(en, Base)` 增加 `zh-Hans`；加入 xcstrings 文件引用。

### 2.2 词条提取（A 类）

用 Apple 官方流程，最准且不会漏：

```
xcodebuild -exportLocalizations -localizationPath ./tmp/en.xcloc -project Compositor.xcodeproj -scheme Compositor
```

导出 `en.xcloc` 即含所有 `LocalizedStringKey` 字面量（A 类全部）。翻译后生成 `zh-Hans.xcloc`，再：

```
xcodebuild -importLocalizations -localizationPath ./tmp/zh-Hans.xcloc -project Compositor.xcodeproj
```

导入即写入 `Localizable.xcstrings` 的 zh-Hans 表。

> 备选（无 Xcode 命令行时）：手工列出所有字面量 key 填入 xcstrings。但 `xcodebuild` 流程更可靠，推荐。

### 2.3 B 类代码改动（少量）

- **`control` 函数**：签名 `String` → `LocalizedStringKey`，使其标题进入本地化表。改动只在 `FilterSheet.swift` 一处函数定义，调用点无需改。
- **枚举 rawValue 本地化**：为 `FilterKind`、`BackgroundQuality` 等加计算属性，例如：
  ```swift
  var localized: String { String(localized: rawValue) }
  ```
  显示处改用 `.localized`。key 用原英文 rawValue，翻译在 xcstrings 中补。
- `.help(...)` 文案：随 A 类一并提取翻译，无需改代码。

## 3. 实施步骤（分阶段，逐步批准）

1. **阶段一（基础设施）**：建 `Localizable.xcstrings` + 改 `pbxproj` 本地化设置。验证：项目可编译、英文默认不变。
2. **阶段二（A 类提取与翻译）**：`xcodebuild` 导出 en → 套用术语库翻译 → 导入 zh-Hans。验证：切换 App 语言为简体中文，菜单/按钮/面板标题变中文。
3. **阶段三（B 类代码改动）**：改 `control` 签名 + 枚举 `.localized`。验证：滤镜滑块标题、滤镜名下拉变中文。
4. **阶段四（收尾）**：构建验证、截图核对、提交到 `zh-cn`。

> 每阶段独立提交，便于上游更新时 `git merge upstream/main` 增量对齐。

## 4. 术语与风格

见 [`zh-CN-terminology.md`](./zh-CN-terminology.md)。核心：对齐 Photoshop 官方简体中文；保留 `Camera Raw`、`.comp`、滤镜专有名词；菜单省略号用 `…`。

## 5. 验收标准

- [ ] 系统/App 语言设为简体中文后，菜单栏、滤镜面板、各 Sheet 标题显示中文。
- [ ] 默认英文（en）界面完全不受影响。
- [ ] 滤镜名（如「高斯模糊」）、滑块标题（如「曝光度」）正确本地化。
- [ ] 项目可正常 `xcodebuild` 构建、运行。
- [ ] 上游 `main` 有更新时，`git merge upstream/main` 可合并且中文不丢。

## 6. 维护流程（已就绪）

```
git fetch upstream                 # 拉上游最新
git merge upstream/main            # 在 zh-cn 上快进/合并
# 若上游改了 UI 文案：A 类重跑 xcodebuild 提取并补译；B 类解冲突
git push origin zh-cn              # 推到自己的 fork
```

remote 现状：`origin` = `xdxray/Compositor`（你的 fork），`upstream` = `robbietilton/Compositor`（原始）。

## 7. 风险与工作量

| 项 | 说明 |
|---|---|
| 上游不接受翻译 PR | 已确认；本地 fork 维护即可，不影响。 |
| 上游改 UI 文案 | A 类 key 变化需重提取；B 类可能合并冲突，局部解。 |
| 工作量 | A 类在「翻译」本身（数百词条，套术语库）；B 类仅 1 处函数签名 + 几个枚举。远低于全量抽离。 |
| 误译风险 | 用 PS 官方术语库统一，降低不一致。 |

## 8. 待你裁决

- 是否按「阶段一 → 四」顺序推进（每阶段单独批准）？
- 翻译范围：先覆盖高频面板（滤镜/图层/导出），还是一次性全量？
