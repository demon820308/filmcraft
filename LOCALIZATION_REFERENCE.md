# FilmCraft 简体中文汉化全量架构与修改参考手册 (Chinese Localization Reference)

本文档详细记录了 FilmCraft 项目进行全界面简体中文化改造的所有修改。包含系统架构设计、数据流向、涉及文件清单、每个文件中的具体修改位置（函数名、代码段、行号区间）、汉化内容以及各模块之间的依赖与关联关系。供后续维护、代码审查及与上游同步合并时参考。

---

## 一、 系统架构与关联机制

FilmCraft 的汉化系统遵循轻量、零额外运行时开销、无损回退（Graceful Fallback）的设计原则。其整体分层与数据调用流向如下图所示：

```mermaid
flowchart TD
    subgraph 引擎与配置层 [1. 引擎与配置持久化]
        A["crates/engine/src/settings.rs<br>(AppearancePrefs: language='zh-cn')"] -->|保存/加载| B["preferences.json<br>(配置文件)"]
        A -->|会话同步| C["filmcraft_engine::Session"]
    end

    subgraph 字体扫描与呈现层 [2. 字体扫描与渲染 Fallback]
        D["crates/text/src/fonts.rs<br>(识别中文字体族)"] --> E["crates/ui-egui/src/i18n.rs<br>(system_chinese_font)"]
        E -->|注册到| F["crates/ui-egui/src/theme.rs<br>(egui FontDefinitions)"]
    end

    subgraph 核心字典与翻译路由 [3. 核心字典与翻译路由]
        G["crates/ui-egui/src/i18n.rs"]
        H["Language::ZhCn 枚举"]
        I["UI_CHINESE 词典表 (1145+ 项)"]
        J["CHINESE 菜单命令表 (350+ 项)"]
        G --- H
        H --> I
        I -->|未命中回退| J
        J -->|未命中回退| K["英文原文 (Fallback)"]
    end

    subgraph 界面渲染与调用层 [4. 界面渲染与调用 (UI Panels & Dialogs)]
        L["持有 app 的面板/组件<br>app.tr('Text')"] --> G
        M["仅有 ctx / ui 的闭包/组件<br>crate::i18n::tr_ctx(ctx, 'Text')"] --> G
        N["紧凑循环/高频渲染<br>lang.tr('Text')"] --> G
    end

    C -->|启动读取设置| L
    F -->|字体保障，消除方块| L
```

### 1. 翻译查找核心模型
- **查表函数 `Language::tr(&self, text: &str) -> &str`**：
  1. 如果当前语言为 `Language::En`，直接返回英文原文指针。
  2. 如果当前语言为 `Language::ZhCn`，首先在 `UI_CHINESE`（界面专用快速词典）中线性匹配；
  3. 若 `UI_CHINESE` 未命中，自动回退到 `CHINESE`（菜单与命令词典）查找；
  4. 若仍未命中，原样返回传入的英文字符串（保证未翻译文本永不崩溃、不丢内容）。
- **全局上下文助手 `tr_ctx(ctx: &egui::Context, text: &str) -> &str`**：
  - 从 egui 临时上下文数据中提取当前会话激活的 `Language`（键 `"interface-language"`），无需向深层组件传递 `&FilmcraftApp`，即可安全获取当前语言翻译。

### 2. 字体加载保障机制（防止出现方块/豆腐块）
- 界面在英文系统或默认安装下若无 CJK 字体，中文菜单会变成不可识别的方块（`ToFu`）。
- 我们在 `crates/ui-egui/src/i18n.rs` 中实现了 `system_chinese_font()`，自动按优先级扫描操作系统原生安装的高质量中文字体：
  - Windows: `Microsoft YaHei UI`, `Microsoft YaHei`, `SimSun`, `DengXian` 等；
  - macOS: `PingFang SC`, `Hiragino Sans GB`, `Heiti SC` 等；
  - Linux/跨平台: `Noto Sans CJK SC`, `Source Han Sans SC` 等。
- 在 `crates/ui-egui/src/lib.rs` 的应用初始化阶段，自动将该字体挂载到所有 egui 字体族的备用队列末端，确保任何时刻切换到中文均能正确显示。

### 3. 设置持久化生命周期
- 用户在菜单栏选择「简体中文」(`app.language.chinese`) 时：
  1. `crates/ui-egui/src/menus.rs` 执行菜单命令，将 `app.ui.language` 设为 `Language::ZhCn`；
  2. 同步写入 `app.session.prefs.appearance.language = "zh-cn".into()`；
  3. `filmcraft_engine` 的自动保存机制将设置写入持久化文件 `preferences.json`；
  4. 软件下次启动时，`FilmcraftApp::new()` 从 `session.prefs.appearance.language` 自动恢复为 `Language::ZhCn`，实现跨重启持久化。

---

## 二、 涉及的模块与文件全景清单 (共 38 个文件)

| 类别 | 模块 / 路径 | 主要职责与修改内容 |
| :--- | :--- | :--- |
| **底层与持久化** | `crates/engine/src/settings.rs` | 偏好设置中增加语言持久化字段 `AppearancePrefs.language` |
| | `crates/engine/src/autosave.rs` | 确保偏好变动触发自动保存配置同步 |
| | `crates/engine/src/settings_tests.rs` | 偏好设置序列化/反序列化单测适配 |
| **文本与字体** | `crates/text/src/fonts.rs` | 识别中文字体族与元数据 |
| | `crates/ui-egui/src/theme.rs` | 注入中文字体 Fallback 到 egui 字体表 |
| **翻译核心与菜单** | `crates/ui-egui/src/i18n.rs` | 中文核心：`Language::ZhCn`、系统字体探测、`UI_CHINESE` 词典 (1145+ 项) |
| | `crates/ui-egui/src/menus.rs` | 顶层菜单栏项与下拉菜单项中文化、增加简体中文切换选项 |
| | `apps/filmcraft/src/native_menu.rs` | macOS 原生菜单栏中文命令绑定 |
| **框架与头部** | `crates/ui-egui/src/header.rs` | 模式标签栏 (导入/编辑/导出)、工作区布局切换、快速导出、全屏/音量提示汉化 |
| | `crates/ui-egui/src/dock.rs` | Dock 面板标签栏占位符与标题中文化 |
| | `crates/ui-egui/src/lib.rs` | 启动时语言恢复、字体初始化、状态栏与快捷键提示中文化 |
| **主工作模式** | `crates/ui-egui/src/panels/import_mode.rs` | 导入模式界面标题、占位引导语汉化 |
| | `crates/ui-egui/src/panels/export_mode.rs` | 导出模式全量参数（格式、预设、视频、音频、高级编码、元数据）汉化 |
| **核心编辑面板** | `crates/ui-egui/src/panels/timeline.rs` | 时间轴顶部控制栏、时间码、轨道头右键菜单、多机位菜单汉化 |
| | `crates/ui-egui/src/panels/monitor.rs` | 监视器走带控制器 (播放/停止/标记/步进) Tooltip 汉化 |
| | `crates/ui-egui/src/panels/monitor_view.rs` | 监视器扳手菜单、缩放比例菜单、安全边框、对比视图汉化 |
| | `crates/ui-egui/src/panels/trim_monitor.rs` | 修剪监视器左右画面提示、动态修剪走带按钮汉化 |
| | `crates/ui-egui/src/panels/effect_controls.rs`| 特效控制台与属性面板（变换、裁剪、音频、速度、参数动画、关键帧）汉化 |
| | `crates/ui-egui/src/panels/graphics.rs` | 基本图形编辑面板（文本、形状、对齐、字距、外观描边、阴影）汉化 |
| | `crates/ui-egui/src/panels/graphics_templates.rs`| 图形模板库、响应式设计时间/位置 (Pin To / Roll) 汉化 |
| | `crates/ui-egui/src/panels/effects.rs` | 效果面板分类树（音频/视频效果、过渡、预设、搜索栏）汉化 |
| | `crates/ui-egui/src/panels/presets.rs` | 预设面板目录、保存预设对话框汉化 |
| | `crates/ui-egui/src/panels/mixer.rs` | 音频轨道混合器读写模式、自动化状态、走带控制汉化 |
| | `crates/ui-egui/src/panels/text.rs` | 文本面板（转录文本、字幕、图形标签页、搜索栏）汉化 |
| **资产与媒体库** | `crates/ui-egui/src/panels/project.rs` | 项目面板状态栏、新建项菜单、预览区、右键上下文菜单汉化 |
| | `crates/ui-egui/src/panels/project_views.rs` | 项目列表视图表头列名、图标视图悬停提示汉化 |
| | `crates/ui-egui/src/panels/media_browser.rs` | 媒体浏览器面包屑、收藏夹、磁盘驱动器树、文件过滤汉化 |
| **功能弹窗与对话框** | `crates/ui-egui/src/panels/dialogs.rs` | 新建序列、重命名、新建项目等基础弹窗汉化 |
| | `crates/ui-egui/src/panels/clip_dialogs.rs` | 剪辑速度/持续时间、多机位创建等对话框汉化 |
| | `crates/ui-egui/src/panels/color_dialogs.rs` | 颜色选择器与色板对话框汉化 |
| | `crates/ui-egui/src/panels/file_dialogs.rs` | 替换素材、查找缺失文件对话框汉化 |
| | `crates/ui-egui/src/panels/media_dialogs.rs` | 链接媒体、设为离线对话框汉化 |
| | `crates/ui-egui/src/panels/project_dialogs.rs`| 序列设置、工程属性对话框汉化 |
| | `crates/ui-egui/src/panels/shortcuts_dialog.rs`| 键盘快捷键编辑器表头、命令分组、搜索汉化 |
| | `crates/ui-egui/src/panels/mod.rs` | 面板右上角汉堡菜单（关闭面板、解除停靠等）汉化 |

---

## 三、 各文件具体修改位置与汉化内容详解

### 1. 基础架构与配置层

#### (1) `crates/engine/src/settings.rs`
- **修改位置**：`AppearancePrefs` 结构体 (约第 36 行) 及 `Default` 实现。
- **更改代码**：
  ```rust
  pub struct AppearancePrefs {
      pub theme: String,
      pub language: String, // 增加 language 字段持久化，默认为 "en"
      ...
  }
  ```
- **关联关系**：被 `FilmcraftApp::new()` 读取恢复语言状态；当用户切换语言时由 `crates/ui-egui/src/menus.rs` 写入。

#### (2) `crates/text/src/fonts.rs`
- **修改位置**：字体分类与字体族探测函数 (约第 140 行)。
- **更改代码**：增加中文字体特征字形检测，使字体扫描器能识别中文字体元数据。
- **关联关系**：支持 `i18n.rs` 中的 `system_chinese_font()` 精确获取本地 CJK 字形。

---

### 2. 国际化核心与顶层交互层

#### (3) `crates/ui-egui/src/i18n.rs`
- **修改位置**：
  - 第 11–24 行：`enum Language` 增加 `ZhCn` 变体，支持 `parse("zh-cn" | "zh")` 与持久化标识 `"zh-cn"`。
  - 第 61–78 行：`Language::tr()` 改造，加入优先查找 `UI_CHINESE`，未命中回退至 `CHINESE` 的双层路由机制。
  - 第 81–85 行：新增公共辅助函数 `pub fn tr_ctx<'a>(ctx: &egui::Context, text: &'a str) -> &'a str`。
  - 第 270–310 行：实现 `pub fn system_chinese_font() -> Option<Arc<egui::FontData>>`，扫描并缓存操作系统中文字体文件。
  - 第 900–1270 行：`const CHINESE` 菜单命令对照表（350+ 条标准 Premiere Pro 命名）。
  - 第 1276–2420 行：`const UI_CHINESE` 界面高频交互对照表（1145+ 条）：涵盖工作区、占位符、弹窗、表头、按钮、各类参数属性等。
  - 第 2676–2710 行：增加中文单测 `chinese_needs_an_installed_font`，确保无中文字体时安全回退不崩溃。
- **关联关系**：整个 UI 层翻译的基础核心。所有其他面板均通过引用此模块的 `tr` 或 `tr_ctx` 进行翻译。

#### (4) `crates/ui-egui/src/menus.rs`
- **修改位置**：
  - 第 38–45 行：视图菜单中增加「语言 (Language) ▸ 简体中文」命令路由 `app.language.chinese`。
  - 第 120–160 行：遍历顶级菜单栏与下拉项时，使用 `lang.tr(item.label)` 动态呈现中文。
- **关联关系**：响应用户的语言切换点击，调用 `app.session.prefs.appearance.language` 并刷新界面。

#### (5) `crates/ui-egui/src/header.rs`
- **修改位置**：
  - 第 22–30 行：顶部主要工作模式按钮（导入 `Import` ➔ `导入`、编辑 `Edit` ➔ `编辑`、导出 `Export` ➔ `导出`）。
  - 第 42–60 行：主标题项目状态（`Edited` ➔ `已编辑`）、快速导出按键（`Quick Export` ➔ `快速导出`）。
  - 第 75–95 行：工作区布局下拉菜单选项汉化。
- **关联关系**：直接作为应用最上方常驻导航条，使用 `app.tr()` 进行国际化显示。

#### (6) `crates/ui-egui/src/dock.rs`
- **修改位置**：
  - 第 24 行、第 78 行：面板容器占位符绘制函数 `placeholder` 支持国际化文本渲染。
  - 第 135 行：Dock 面板默认标签名格式化 `lang.tr(tab.title)`。
- **关联关系**：使停靠窗口在无序列或无剪辑时显示的提示语（如 `(无序列)`、`选择剪辑以查看其属性`）呈现中文。

---

### 3. 专业编辑面板层

#### (7) `crates/ui-egui/src/panels/effect_controls.rs` (特效控制台 & 属性面板)
- **修改位置**：
  - 第 34 行、第 38 行：未选中剪辑或序列时的空状态占位符汉化。
  - 第 79–81 行：视频 / 音频效果列表分组标题汉化。
  - 第 111–130 行：特效开关 `fx` 提示、重置按钮 `Reset Effect` ➔ `重置效果`、右键上下文菜单 `Save Preset…` ➔ `保存预设…`、`Clear` ➔ `清除`。
  - 第 181–186 行：自定义设置行 `Custom Setup` ➔ `自定义设置`、`Edit…` ➔ `编辑…`。
  - 第 239–260 行：关键帧图表切换 `Show graphs` ➔ `显示图表`、秒表动画切换 `Toggle animation` ➔ `切换动画`、参数标签 `app.tr(pd.label)` 汉化。
  - 第 550–553 行：**属性面板行标签 `row`**：`ui.painter().text(..., lang.tr(label), ...)`。汉化 `Position` ➔ `位置`、`Anchor point` ➔ `锚点`、`Scale` ➔ `缩放`、`Rotation` ➔ `旋转`、`Opacity` ➔ `不透明度`、`Left/Top/Right/Bottom` ➔ `左侧/顶端/右侧/底端`、`Level` ➔ `级别`、`Pan` ➔ `声像`。
  - 第 567 行：**属性面板分组折叠标题 `section`**：`ui.painter().text(..., lang.tr(name), ...)`。汉化 `Transform` ➔ `变换`、`Crop` ➔ `裁剪`、`Audio` ➔ `音频`。
  - 第 707 行：底部速度按钮：`format!("{} {:.0}%", lang.tr("Speed"), ...)` ➔ `速度 100%`。
- **关联关系**：直接解决用户在属性面板 (`Properties`) 中看到的各项变换与裁剪属性为英文的问题。

#### (8) `crates/ui-egui/src/panels/graphics.rs` (基本图形编辑面板)
- **修改位置**：
  - 第 1024–1033 行：属性行绘制函数 `Ctx::row`：增加 `trimmed` 与 `indent` 计算，使用 `crate::i18n::tr_ctx(ui.ctx(), trimmed)` 汉化所有属性行（包括带有前置空格缩进的属性）。
  - 第 1082–1090 行：`Ctx::check` 复选框汉化。
  - 第 1101–1119 行：`Ctx::choice` 下拉框当前项与选项列表汉化（`Rectangle` ➔ `矩形`、`Ellipse` ➔ `椭圆`、`Outer/Center/Inner` ➔ `外侧/居中/内侧` 等）。
  - 第 1220–1240 行：对齐图标按钮 `glyph_button` 与字母按钮 `letter_button` 的鼠标悬停 Tooltip 汉化（`Align Left` ➔ `左对齐`、`Distribute Horizontally` ➔ `水平分布`、`Faux Bold` ➔ `仿粗体` 等）。
  - 第 1404 行：文本属性扳手按钮 Tooltip `Text Properties` ➔ `文本属性`。
  - 第 1488 行：`Vertical Text` 复选框 ➔ `竖排文本`。
  - 第 1589–1618 行：`Text Properties` 对话框：弹窗标题、`Text Layer Type` ➔ `文本图层类型`、`Point Text / Paragraph Text` ➔ `点文本 / 段落文本`、`Text Styling` ➔ `文本样式`、`Cancel / OK` ➔ `取消 / 确定` 汉化。
- **关联关系**：与 `crates/project/src/graphic.rs` 定义的图层参数保持对应，提供完整的基本图形本地化交互。

#### (9) `crates/ui-egui/src/panels/graphics_templates.rs` (图形模板与响应式设计)
- **修改位置**：
  - 第 134–146 行：标签页切换按钮 `Browse` ➔ `浏览`、`Edit` ➔ `编辑` 汉化。
  - 第 175–215 行：搜索输入框提示 `Search templates…` ➔ `搜索模板…`、分类下拉选择框汉化。
  - 第 324–332 行：属性行绘制函数 `row_label`：使用 `crate::i18n::tr_ctx(ui.ctx(), label)` 汉化行标题。
  - 第 474–515 行：响应式时间设置 `responsive_time`：滚动模式 `Roll` 下拉菜单选项（`Off` ➔ `关`、`Roll` ➔ `滚动`、`Crawl Left/Right` ➔ `向左/向右慢移`）、复选框（`Start Off Screen` ➔ `从屏幕外开始`、`End Off Screen` ➔ `在屏幕外结束`）、前卷/后卷/缓入/缓出等时间项汉化。
  - 第 542–555 行：响应式位置设置 `responsive_position`：固定目标 `Pin To` ➔ `固定到`、`None` ➔ `无`、`Video Frame` ➔ `视频帧`、固定边缘 `Pinned Edges` ➔ `固定的边缘` 汉化。
- **关联关系**：被 `graphics.rs` 中的模板控件部分调用，完成 Essential Graphics 的全部汉化。

#### (10) `crates/ui-egui/src/panels/effects.rs` & `presets.rs` (效果与预设面板)
- **修改位置**：
  - `effects.rs`：第 60–140 行：效果分类树根目录（`Video Effects` ➔ `视频效果`、`Audio Effects` ➔ `音频效果`、`Video Transitions` ➔ `视频过渡`、`Audio Transitions` ➔ `音频过渡`、`Presets` ➔ `预设`）及各子目录项翻译呈现。
  - `presets.rs`：第 90–125 行：保存预设对话框 `Save Preset`、类型选择（比例 `Scale`、定位锚点 `Anchor to In/Out Point`）汉化。
- **关联关系**：与渲染引擎预置效果定义相绑定，使用户以熟悉的中文分类检索视频/音频滤镜。

#### (11) `crates/ui-egui/src/panels/export_mode.rs` (导出模式全流程)
- **修改位置**：
  - 第 40–160 行：导出页面侧边栏分类（`Format` ➔ `格式`、`Preset` ➔ `预设`、`Range` ➔ `范围`、`Video` ➔ `视频`、`Audio` ➔ `音频`、`Captions` ➔ `字幕`、`Effects` ➔ `效果`、`General` ➔ `常规`）。
  - 第 220–480 行：编码参数、渲染精度（`Render at Maximum Depth` ➔ `以最大深度渲染`、`Use Maximum Render Quality` ➔ `使用最高渲染质量`）、时间范围（`Entire Sequence` ➔ `整个序列`、`Source In/Out` ➔ `源入点/出点`）等 80 余处导出专业术语汉化。
- **关联关系**：覆盖整个全屏导出交互流程，确保与 Adobe Media Encoder 标准术语一致。

---

### 4. 监视器、时间轴与音频混音层

#### (12) `crates/ui-egui/src/panels/monitor.rs` & `monitor_view.rs`
- **修改位置**：
  - `monitor.rs`：第 486–530 行：底部走带栏所有控制按钮的 Tooltip 汉化：
    - `Add Marker` ➔ `添加标记`
    - `Mark In` ➔ `标记入点`、`Mark Out` ➔ `标记出点`
    - `Go to In Point` ➔ `转到入点`、`Go to Out Point` ➔ `转到出点`
    - `Step Backward 1 Frame` ➔ `后退 1 帧`、`Step Forward 1 Frame` ➔ `前进 1 帧`
    - `Play / Stop` ➔ `播放/停止`
  - `monitor_view.rs`：第 648–700 行：监视器扳手设置菜单（`Safe Margins` ➔ `安全边距`、`Transparency Grid` ➔ `透明网格`、`Show Rulers` ➔ `显示标尺`、`Show Guides` ➔ `显示参考线`、`Snap in Program Monitor` ➔ `在节目监视器中对齐`）汉化；缩放下拉菜单 `Fit` ➔ `适合` 汉化。
- **关联关系**：源监视器 (`Source`) 与节目监视器 (`Program`) 共享这套逻辑，保证监视器界面交互完全中文展示。

#### (13) `crates/ui-egui/src/panels/timeline.rs` (时间轴面板)
- **修改位置**：
  - 第 1098–1125 行：时间轴左上角序列设置菜单、时间码格式化提示汉化。
  - 第 1634–1640 行：多机位上下文菜单汉化。
  - 第 2086–2095 行：时间轴右键空白处/剪辑处菜单指令汉化。
- **关联关系**：作为核心非编剪辑区域，协调剪辑菜单与时间轴播放头行为。

#### (14) `crates/ui-egui/src/panels/mixer.rs` (音频轨道混合器)
- **修改位置**：
  - 第 212–260 行：各轨道音频读取/写入模式下拉选项（`Read` ➔ `读取`、`Touch` ➔ `触碰`、`Latch` ➔ `锁定`、`Write` ➔ `写入`）汉化。
  - 第 838–845 行：混合器独立走带控制器（循环、录制就绪等）Tooltip 汉化。
- **关联关系**：与底层音频混合引擎保持一致，提供专业调音台中文支持。

---

### 5. 项目资产、媒体库与对话框层

#### (15) `crates/ui-egui/src/panels/project.rs` & `project_views.rs`
- **修改位置**：
  - `project.rs`：第 380–430 行：项目面板底部状态统计（`items` ➔ `个项目`、`selected` ➔ `已选中`）、新建项菜单（`Sequence…` ➔ `序列…`、`Bin` ➔ `素材箱` 等）汉化。
  - `project_views.rs`：第 203–285 行：详细信息列表表头（`Name` ➔ `名称`、`Frame Rate` ➔ `帧速率`、`Media Start` ➔ `媒体开始`、`Media End` ➔ `媒体结束`、`Media Duration` ➔ `媒体持续时间`、`Video Info` ➔ `视频信息`、`Audio Info` ➔ `音频信息` 等）汉化；素材右键上下文菜单汉化。
- **关联关系**：资产管理核心，为项目树形列表和多列属性呈现提供中文。

#### (16) `crates/ui-egui/src/panels/dialogs.rs` & 其它功能弹窗
- **修改位置**：
  - `dialogs.rs`：新建序列对话框（时基、视频帧大小、像素长宽比、音频采样率）、新建项目对话框全量汉化。
  - `clip_dialogs.rs`：速度与持续时间对话框（速度百分比、倒放速度、保持音频音调、波纹编辑移动后方剪辑等）汉化。
  - `media_dialogs.rs`：链接媒体文件（包含匹配文件名、文件扩展名、媒体开始时间等检索条件）汉化。
  - `shortcuts_dialog.rs`：快捷键设置弹窗搜索栏、分类筛选、重置按钮汉化。
- **关联关系**：弹出式模态窗口，在用户执行高级操作时提供完整的母语指引。

---

## 四、 字典规范与后续维护同步指南

### 1. 字典结构说明
`crates/ui-egui/src/i18n.rs` 包含两个核心静态表：
1. **`const UI_CHINESE: &[(&str, &str)]`**：
   - 存储所有的 UI 按钮、标签、弹窗说明、属性名称、状态栏提示。
   - 匹配规则：完全精准字符串匹配。
   - 要求：**不得出现重复的英文字符串 Key**（由测试用例 `scratch/verify_no_dups.py` 进行静态校验）。
2. **`const CHINESE: &[(&str, &str)]`**：
   - 与 Spanish / Portuguese 保持相同的菜单项与全局命令 ID 对照表（350 条），必须与 `MENUS` 完全对齐，不能任意删减键名，否则会触发单元测试报警。

### 2. 向上游同步代码时的合并规范
当上游主仓库增加新功能或修改已有代码时：
1. **针对 UI 面板代码**：
   - 如果上游在某个面板增加了新文本渲染，只需按照本项目约定将其包装为 `app.tr("New Text")` 或 `crate::i18n::tr_ctx(ui.ctx(), "New Text")`。
   - 尽量避免引入深度嵌套闭包导致的借用冲突（Borrow Conflict），若出现借用冲突，可提前提取 `let lang = app.ui.language;` 并直接调用 `lang.tr(...)`。
2. **针对词典 `UI_CHINESE`**：
   - 在 `crates/ui-egui/src/i18n.rs` 的 `UI_CHINESE` 列表中追加对应的 `("New Text", "新文本")`。
   - 运行检查工具确保无冲突、无重复。

### 3. 本地自动化验证命令
在每次修改提交前，执行以下步骤确保 100% 稳定性：
```powershell
# 1. 校验 UI_CHINESE 是否存在重复键
python scratch/verify_no_dups.py

# 2. 编译检查 (语法与借用检查)
cargo check -p filmcraft-ui-egui

# 3. 运行全量单元测试 (包括多语言持久化与字体探测单测)
cargo test -p filmcraft-ui-egui --lib
```
全部测试通过后，即可安全提交并合并。
