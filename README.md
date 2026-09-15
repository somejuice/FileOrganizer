# 文件整理器 · Android 项目总 README

> 一个纯原生（零第三方依赖）的 Android 文件自动整理应用，以及支撑它开发的轻量安卓构建插件。
> 在本工作区内从零完成：**工具链搭建 → 应用开发 → 签名 APK 交付**。

| | |
|---|---|
| 应用名称 | 文件整理器（File Organizer） |
| 包名 / 版本 | `com.arena.fileorganizer` / **v2.2**（versionCode 4） |
| 系统支持 | Android 5.0（API 21）～ Android 14+（targetSdk 34） |
| 体积 | APK 约 **74KB**（无任何第三方依赖） |
| 安装包 | [`文件整理器-v2.2.apk`](文件整理器-v2.2.apk)（调试签名，可直接侧载） |
| UI | Material 3 + 莫奈（Monet）动态取色 + 完整深色模式 |

---

## 目录

1. [快速开始](#快速开始)
2. [仓库结构](#仓库结构)
3. [功能特性](#功能特性)
4. [界面与设计](#界面与设计)
5. [构建与复现](#构建与复现)
6. [架构与源码导读](#架构与源码导读)
7. [技术要点](#技术要点)
8. [版本历史](#版本历史)
9. [常见问题](#常见问题-faq)

---

## 快速开始

### 安装

1. 下载本仓库根目录的 `文件整理器-v2.2.apk`，传到手机（或直接下载）后点击安装，允许“未知来源/安装未知应用”；
2. 首次打开，在「权限状态」卡片点 **授权**，依次完成：
   - **所有文件访问**（Android 11+ 跳系统设置页；Android 10- 为传统读写权限弹窗）
   - **通知权限**（Android 13+，用于常驻通知）
   - **电池优化白名单**（增强常驻）
3. 点「选择目录」，用内置目录浏览器选定要整理的目录（默认 `Download`）；
4. 按需调整规则（默认已内置 6 类常用规则）；
5. 打开「后台常驻整理」开关 —— 完成。也可先点「立即整理一次」验证效果。

### 一分钟理解它做什么

往监控目录里丢一个 `photo.jpg` → 约 1.5~4.5 秒后（事件防抖 + 3 秒静默保护），
它被自动移动到 `监控目录/图片/photo.jpg`。目录里从此只有文件夹，没有散落文件。

---

## 仓库结构

```
.
├── 文件整理器-v2.2.apk      # ← 最终交付物（签名 APK）
├── README.md                # 本文件（总览）
├── android-plugin/          # 安卓开发插件（轻量无 Gradle 构建链）
│   ├── setup.sh             #   SDK 组件安装（build-tools 34 + platform 34 → /opt/android-sdk）
│   ├── build.sh             #   一键构建：aapt2→javac→d8→zipalign→apksigner
│   ├── debug.jks            #   调试签名密钥（口令 android；升级安装需保持同一密钥）
│   └── README.md            #   插件详细文档
└── FileOrganizer/           # 应用项目
    ├── AndroidManifest.xml  #   清单（权限 / 服务 / 开机接收器）
    ├── src/com/arena/fileorganizer/
    │   ├── MainActivity.java     # 主界面（M3 着色、权限、规则管理、服务开关）
    │   ├── M3Theme.java          # M3 主题中心：莫奈取色三策略 + 控件样式
    │   ├── M3TextField.java      # M3 填充式文本输入框（手写组件）
    │   ├── DirPicker.java        # M3 目录浏览选择对话框（可复用）
    │   ├── OrganizerService.java # 前台常驻服务（FileObserver + 定时扫描）
    │   ├── OrganizerEngine.java  # 整理引擎（扫描/匹配/改名/移动）
    │   ├── RulesStore.java       # 持久化（SharedPreferences + JSON）
    │   └── BootReceiver.java     # 开机自启恢复服务
    ├── res/                  #   资源（M3 布局/四套色板变体/矢量图标）
    ├── icon_512.png          #   图标母版（PIL 绘制）
    ├── FileOrganizer.apk     #   构建产物
    └── README.md             #   应用详细文档（含权限说明）
```

> 工作区约束：全部内容 **432KB / 45 个文件**，远低于 120MB / 10000 文件的限额。
> SDK 与构建中间产物都放在工作区外的 `/opt`、`/tmp`，不占快照空间。

---

## 功能特性

### 核心功能

- 📁 **指定监控目录**：内置可视化目录浏览器（逐级进入/返回上级/粘贴路径回车跳转），默认 `Download`
- 🏷️ **后缀名规则**：`后缀 → 目标文件夹`
  - 默认规则：图片 / 视频 / 音乐 / 文档 / 安装包 / 压缩包（30+ 常见后缀）
  - 一条规则可录多个后缀（`jpg, png` 逗号/空格分隔，自动小写化、去点号）
  - **目标文件夹可视化选择**：规则对话框点「浏览」弹出目录浏览器；选监控目录内的文件夹自动填相对名（📋），选外部目录填绝对路径（📂，列表图标区分）
  - 输入即**实时预览**最终整理到的完整路径
  - 点按编辑 / 长按删除 / 一键恢复默认
- 📦 **自动移动**：
  - `FileObserver`（inotify）实时监控：新建 / 移入 / 写入完成事件，1.5s 防抖合并后整理
  - 定时全量备份扫描：关闭 / 1 / 5 / 15 / 30 / 60 分钟（防止实时监控漏事件）
  - 手动「立即整理一次」
- ♻️ **后台常驻**：
  - 前台服务 + 低优先级常驻通知（`specialUse` 类型，Android 14/15 不受 6 小时前台服务限制）
  - `START_STICKY`：进程被杀后系统自动拉起
  - `BOOT_COMPLETED`：开机自启恢复
  - 引导加入电池优化白名单；通知栏可一键「停止服务」

### 安全细节（不会弄丢/弄乱文件）

| 机制 | 说明 |
|---|---|
| 3 秒静默保护 | 最近 3 秒内仍在写入的文件（如下载中）不会被移动，留待下轮 |
| 重名自动改名 | 目标已存在 `a.jpg` 时依次尝试 `a(1).jpg`、`a(2).jpg`…，绝不覆盖 |
| 跨文件系统回退 | `renameTo` 失败（跨存储/SAF 场景）自动退化为「复制 + fsync + 删除」，复制失败回滚不留残件 |
| 只整理顶层 | 不递归子目录——分类文件夹里的内容永不被二次移动，规则目标=监控目录本身时跳过 |
| 隐藏文件跳过 | `.` 开头的文件与目录一律不碰 |

### 统计与记录

- 常驻通知实时显示「监控中：<目录> · 已整理 N 个文件」
- 底部统计胶囊显示累计整理数，点击查看最近 60 条整理记录（成功/失败、时间、去向）

---

## 界面与设计

**Material 3 + 莫奈动态取色**，纯框架实现（无 AppCompat / Material Components 依赖）：

### 取色策略（按系统版本自动选择）

| 系统版本 | 取色方式 |
|---|---|
| Android 12+ | **系统莫奈调色板**：色彩角色在 `values-v31` 直接映射框架公开资源 `@android:color/system_accent1_*`、`system_neutral1/2_*`——壁纸换色，界面实时跟随（纯资源层，零代码） |
| Android 8 ~ 11 | **自研壁纸取色**：`WallpaperManager` 取壁纸 → 64×64 降采样 → 12 色相桶按饱和度加权选主色 → HSV 变换生成全套 M3 角色 |
| Android 5 ~ 7 | 内置静态 M3 色板（蓝色系） |

### M3 视觉语言

- **布局**：无标题栏 + 24sp 大标题（headline-small）+ 20dp 圆角卡片分层
- **按钮三级体系**：填充式（Filled，主色底）/ 色调式（Tonal，主色容器底）/ 文本式（Text），全部胶囊圆角 + 涟漪反馈
- **对话框**：28dp 圆角、`surfaceContainerHigh` 实底表面、0.32 遮罩、主色 medium 文本按钮
- **输入框**：`M3TextField` 填充式文本字段——`surfaceContainerHighest` 实底、顶 4dp 圆角/底直角、底部指示线（休眠 1dp / 聚焦 2dp 主色）、标签常驻顶端
- **列表**：目录列表为 M3 涟漪列表项 + 着色矢量图标（📁 主色 / ⬆ 次要色）；规则列表为「后缀芯片 → 文件夹」行
- **其他**：后缀名色板芯片、统计胶囊、开关/下拉框/通知全部跟随主题色
- **深色模式**：完整 DayNight 支持（values / values-night / values-v31 / values-night-v31 四套色板 + 系统栏明暗自适应）

---

## 构建与复现

本仓库不使用 Gradle / Android Studio。`android-plugin/` 是为沙盒环境搭建的轻量构建链，
任何有 `bash + curl + JDK 11` 的 Linux 机器都能复现：

```bash
cd android-plugin

bash setup.sh                    # ① 安装 SDK 组件到 /opt/android-sdk（约 124MB，幂等可重跑）
bash build.sh ../FileOrganizer   # ② 构建 + 签名 + 校验
# 产物: FileOrganizer/FileOrganizer.apk（约 74KB）
```

`build.sh` 内部流水线：

| 步骤 | 工具 | 产物 |
|---|---|---|
| 1. 编译资源 | `aapt2 compile` | 二进制资源（.flat） |
| 2. 链接 | `aapt2 link` | resources.arsc + 二进制清单 + `R.java` |
| 3. 编译源码 | `javac -source 1.8`（bootclasspath = android.jar） | .class |
| 4. 转 dex | `d8 --release --min-api 21` | classes.dex（lambda 自动脱糖） |
| 5. 打包对齐 | `zip` + `zipalign -f -p 4` | 对齐 APK |
| 6. 签名校验 | `apksigner`（v1+v2，调试密钥） | 签名 APK |

> 沙盒每次会话重置后 `/opt` 会清空，重跑 `bash setup.sh` 即可恢复（约 5 秒，组件有 170MB/s+ 的下载缓存加速时更快）。
> 修改代码后重新构建即可；**升级安装需保持 `android-plugin/debug.jks` 不变**，否则签名不一致无法覆盖安装。

---

## 架构与源码导读

```
┌─────────────── UI ───────────────┐
│ MainActivity                     │ ← M3Theme.applyM3() 统一着色
│  ├─ DirPicker（目录选择，两处复用）│
│  └─ M3TextField（M3 输入框）      │
└──────────────┬───────────────────┘
               │ 启停 / Intent(ACTION_START/STOP/SCAN)
┌──────────────▼───────────────────┐
│ OrganizerService（前台常驻）      │
│  ├─ FileObserver（inotify 实时）  │── 防抖 1.5s ─┐
│  └─ 定时全量扫描（可选）           │──────────────┤
└──────────────┬───────────────────┘               │
               │ organize()                        ▼
┌──────────────▼───────────────────┐        OrganizerEngine
│ RulesStore（SharedPreferences）   │ ← 规则/目录/统计/日志   扫描→匹配→改名→移动
└──────────────────────────────────┘
        ▲
BootReceiver（开机自启恢复服务）
```

各文件职责与要点：

- **`M3Theme.java`** — 主题单一事实来源：三策略取色（系统莫奈 / 壁纸提取 / 静态回退）、14 个色彩角色、分层表面色（surface → surfaceHigh → fieldFill）、控件样式（卡片/按钮/胶囊/芯片/对话框）
- **`OrganizerService.java`** — 常驻核心：`HandlerThread` 后台执行、`START_STICKY`、`specialUse` 前台类型、通知含累计数与停止按钮、扫描结果经应用内广播（`RECEIVER_NOT_EXPORTED`）回传 UI
- **`OrganizerEngine.java`** — 纯逻辑（已通过桌面 JVM 单元测试）：后缀解析（`extOf`，中文/大小写/无后缀/隐藏文件边界齐全）、目标解析（相对/绝对）、冲突改名（`uniqueDest`）、安全移动（`moveSafe`）
- **`RulesStore.java`** — 规则有序存取（LinkedHashMap ↔ JSON）、默认规则、统计与滚动日志（上限 60 条）

---

## 技术要点

### 无第三方库实现莫奈取色

Android 12+ 框架**公开**了整套莫奈资源（已实证：`system_accent1_0~1000`、`system_neutral1/2_*` 均 PUBLIC 且 ID 稳定，刻度 0=最浅 → 1000=最深——用框架自身取值 `accent_device_default_light = accent1_600`、`background_device_default_dark = neutral1_900` 反推确认）。本项目在 `values-v31/colors.xml` 把 14 个色彩角色映射上去，**资源层自动跟随壁纸**，无需运行时代码。Android 8~11 则由 `M3Theme` 自行从壁纸提取并生成角色色（详见上文取色策略）。

### 常驻服务的四重保障

1. `foregroundServiceType="specialUse"`：Android 14+ 六小时前台服务限制的豁免类型（属性声明 + `PROPERTY_SPECIAL_USE_FGS_SUBTYPE`）
2. `START_STICKY` + `stopWithTask="false"`：被杀/划掉任务后自动恢复
3. `BOOT_COMPLETED` 接收器：开机自启（部分 ROM 需用户在系统设置允许“自启动”）
4. 电池优化白名单引导（`REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`）

### 质量验证（每次构建自动执行）

- `apksigner verify` 签名校验（构建流水线内置）
- `aapt2 dump badging / xmltree` 清单与资源核验（莫奈映射、前台服务类型、权限）
- 桌面 JVM 单元测试：后缀解析（8 组边界用例，含中文文件名）、重名冲突改名、相对/绝对目标解析 —— **全部通过**

---

## 版本历史

| 版本 | 内容 |
|---|---|
| **v2.2** | 修复 v2.1 对话框透明 bug（`surfaceHigh` 漏赋值导致背景为全透明 0）；输入框升级 M3 填充式（实底 + 底部指示线）；对话框遮罩与不透明保险加固 |
| **v2.1** | 对话框全面 M3 化：28dp 圆角表面、`M3TextField` 输入框、M3 图标列表、软键盘自适应 |
| **v2.0** | UI 按 Material 3 重写；莫奈动态取色三策略；完整深色模式；规则目标可视化浏览选择 + 实时路径预览 |
| **v1.0** | 首版：监控目录 + 后缀规则 + 实时/定时/手动整理 + 前台常驻服务 + 开机自启 |

> 详细变更见 [`FileOrganizer/README.md`](FileOrganizer/README.md)。

---

## 常见问题 FAQ

**Q：为什么安装时提示“未知来源”？**
侧载 APK 的正常提示（非应用商店分发）。在系统里允许该来源安装即可。

**Q：后台一段时间后好像不整理了？**
多为国产 ROM（MIUI/EMUI/ColorOS 等）的省电策略：请在系统设置中允许本应用「自启动」+「无限制后台/省电策略无限制」，并确认已加入电池优化白名单（应用内权限卡片会引导）。

**Q：Android 15 上开机不自启？**
部分新 ROM 限制开机拉起前台服务。打开一次 App 即可恢复常驻（开关状态会保留）。

**Q：为什么有的文件没被移动？**
三种可能：① 文件后缀没有对应规则（列表里没有该后缀）；② 文件 3 秒内仍在写入（下载中），等下一轮；③ 该文件是隐藏文件（`.` 开头）。

**Q：移动是复制还是剪切？会不会丢文件？**
优先同卷 `rename`（瞬时）；失败才退化为「复制 → fsync → 删除原文件」，删除失败会回滚副本，任何一步失败都会记录在整理记录里（标 ✗），不会出现覆盖或丢失。

**Q：想改图标/应用名？**
图标母版 `FileOrganizer/icon_512.png`（PIL 生成）与 `res/values/strings.xml` 的 `app_name`，改完重新构建即可。

**Q：如何在本机跑单元测试？**
见「技术要点 → 质量验证」；测试源码为临时工程（沙盒 `/tmp`），核心断言也可参照 `OrganizerEngine` 的方法契约直接编写。

---

## 许可

仅为演示用途的示例项目，可自由学习、修改、二次开发。所使用的 Android SDK 组件遵循其各自许可（Google SDK Terms）。
