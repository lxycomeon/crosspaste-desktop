# 多端补齐与安卓剪贴板韧性：可行性评估

评估基准：本仓库 `main` @ `36683ed7`（2026-10-25）。只读调研，未改业务代码。

范围只包括合法的产品功能：桌面端已经发布的能力如何映射到尚未在本仓库落地的移动端，以及安卓在系统允许的权限模型内，剪贴板读取、监听、同步能做到哪一步。不讨论、不设计去广告、解锁或绕过 Pro、付费授权、license 或 paywall。

工作量只用三档，不估计日历时间：

| 档位 | 含义 |
| --- | --- |
| 小 | 现有桌面模块内可完成，不改同步协议，不新增应用商店目标 |
| 中 | 要新的平台适配层或系统能力，协议和数据模型可以复用，但要新的测试矩阵 |
| 大 | 新的应用目标、打包和商店链路；大量桌面专有实现不能直接搬；行为还被操作系统策略卡住 |

## 1. 结论

本仓库发布的是 **macOS、Windows、Linux 桌面客户端**，外加同一台机器上的 **命令行** 和 **Chrome 扩展**。Gradle 里没有 Android / iOS target，源码树里没有 `androidMain` / `iosMain`，CI 和发布脚本也不产出 APK、AAB 或 IPA。

协议和文案已经把 Android、iPhone、iPad、HarmonyOS 当成对端身份。`doc/en/PasteCatchUp.md` 写明已经发布的移动客户端可以继续调用旧的拉取接口，`doc/en/I18nStyleGuide.md` 写明移动端会拷贝 `commonMain`。这些客户端的源码、构建和商店包不在本仓库，完成度无法从这里核实。本评估把它们列为 **仓库外、未验证**，不记入“本仓库已支持”。

安卓后台剪贴板的上限由系统决定，不是本仓库缺一段监听代码：

- 进程被杀掉，或用户强制停止：没有监听、没有局域网连接、没有补记。系统不会把这期间的复制排队交给以后再启动的应用。
- Android 10（API 29）及更高版本：不是默认输入法，且当前没有输入焦点时，`ClipboardManager.getPrimaryClip()` 返回 `null`。前台服务、常驻通知、WorkManager 都不提供焦点，因此都不能在后台读到剪贴板内容。
- 应用回到前台并拿到输入焦点后，只能读到 **当时仍留在主剪贴板上的那一条**。中间复制过又被覆盖的内容找不回来。Android 13 起系统还会在一段时间后清空主剪贴板（AOSP 默认约 1 小时，厂商可改）。
- 在不越权、不用无障碍服务做剪贴板监听的前提下，做不到桌面端那种“应用在后台仍持续收录每一次复制”。能做的是：前台窗口监听、回到前台时补读最后一条、用户点一下快捷方式再抓取当前一条，以及在进程还活着时接收其他设备推来的内容并写入本机剪贴板。

因此建议的路线不是“把桌面监听搬到手机后台”，而是两段独立交付：先做 **前台可用的移动同步客户端**（大），再把安卓收口到 **系统允许的后台韧性**（中，且必须接受功能缺口）。iOS 比安卓更严，后台监听同样不可行，且完全不在本仓库。

## 2. 本仓库已经支持哪些端

### 2.1 总表

| 端 | 本仓库是否构建并发布 | 证据 | 备注 |
| --- | --- | --- | --- |
| macOS | 是，x64 与 arm64 | README 的 OS 徽章只列 Windows / macOS / Linux。`app/build.gradle.kts` 的 Conveyor 依赖有 `macAmd64`、`macAarch64`，打包格式为 DMG。`conveyor.conf` 注入对应 JBR 与 `libcrypto`。`app/jbr.yaml` 有 `osx-x64`、`osx-aarch64`。发布流程在 `macos-26` 上用 Swift 编译 `libMacosApi.dylib`（x86_64 目标 10.15，arm64 目标 11） | Compose 的 Info.plist 写 `LSMinimumSystemVersion` 10.15；`conveyor.conf` 写 14.0。对外最低系统版本以发布配置 14.0 为准，两处不一致 |
| Linux | 是，x64 与 arm64 | Gradle 有 `linuxAmd64`、`linuxAarch64`，格式为 DEB。发布流程另打 AppImage（amd64 / aarch64）和 EL9 RPM（x86_64 / aarch64），并生成 apt / dnf 仓库 | `build-beta-linux.yml` 只覆盖 Linux 测试包 |
| Windows | 是，x64 | Gradle 打 MSI。`build-release.yml` 另打 Microsoft Store MSIX，并检查 `Version x.y.z.0` 与 `crosspaste-cli.exe`。`msstore-publish.yml` 负责商店发布 | 没有 Windows ARM 安装包。`jbr.yaml` 虽有 `windows-aarch64` 下载项，打包只取 `windows-x64`，CLI 也只有 `mingwX64` |
| 命令行 | 是，作为桌面附属而不是新操作系统 | `cli` 模块：`macosArm64`、`macosX64`、`linuxX64`、`linuxArm64`、`mingwX64`。CI 会链接 `linuxX64`、`linuxArm64`、`mingwX64` | 无头模式明确不读系统剪贴板，只读写本机历史并参与同步 |
| Chrome 扩展 | 是 | `web/` 由 `:web:build` 产出，发布流程打成 zip。扩展发现本机桌面端并同步 | 浏览器扩展，不是第六个操作系统 |
| Android | 本仓库否 | 全仓库 Gradle 无 `android()`。无 `androidMain`。ktlint 全部 `android = false`。无 APK/AAB 任务或 workflow | 协议枚举、设备图标、文档中的“已发布移动客户端”见 2.2 |
| iOS（iPhone / iPad） | 本仓库否 | 无 `iosArm64` / `iosSimulatorArm64`，无 IPA、签名或 TestFlight 配置 | 同上 |
| HarmonyOS | 本仓库否 | 只有 `Platform.HARMONY` 和一张图标 | 无实现、无发布 |

README（英文与中文）的特性列表、开发入口 `./gradlew app:run`、路线图都只描述桌面、命令行、Chrome 扩展和插件系统，没有移动端里程碑。`doc/en/Roadmap.md` 的“即将做”只有插件系统。

### 2.2 协议里出现、但本仓库不构建的端

`core` 的 `Platform` 把下面这些字符串当作线上身份（会进同步信息，不能随意改名）：

- `Windows`、`Macos`、`Linux`
- `iPhone`、`iPad`、`Android`、`HarmonyOS`
- `ChromeExtension`

`app` 的 `commonMain` 已经为 Android、HarmonyOS、iPhone、iPad 准备了图标的 `expect` 声明，桌面 `actual` 只是画出来。设备列表可以显示这些对端，不代表本仓库能编出这些对端。

`doc/en/PasteCatchUp.md` 的 Mobile adoption 写道：已经发布的移动客户端可以继续调用旧的 `/pull/pasteBatch`，它们自己的按时间游标不会因为桌面升级而修好；移动端以后可以再移植按 ID 的补拉。`doc/en/I18nStyleGuide.md` 规定 `commonMain` 的文案键被移动端复用，改名要和移动端同一发布周期。提交说明里也预留了 `:robot:`（Android）和 `:green_apple:`（iOS）。

由此能确定的只有：团队把移动端当作 `commonMain` 的拷贝消费方，而且曾有客户端打过旧拉取接口。不能从本仓库确定这些客户端是否仍在维护、覆盖哪些系统版本、后台是否读剪贴板。

### 2.3 桌面剪贴板现在怎么工作

监听实现全部在 `app/src/desktopMain`，不存在安卓或 iOS 分支。

| 桌面 | 监听 | 读取 / 写入 | 进程还在、窗口在后台时 |
| --- | --- | --- | --- |
| macOS | `MacosApi.swift` 读 `NSPasteboard.general.changeCount`，Kotlin 每 280ms 轮询。会跳过 Universal Clipboard 远端标记、本应用标记，以及密码管理器的 concealed / transient 类型 | 写入走 AWT `Toolkit.getDefaultToolkit().systemClipboard` | 可以继续收录。桌面操作系统允许后台进程读剪贴板 |
| Windows | `User32.AddClipboardFormatListener` 收 `WM_CLIPBOARDUPDATE`，用 `GetClipboardSequenceNumber` 合并突发变更，再经 Win32 格式探测和 AWT 做快照 | 同上，AWT 写入 | 可以继续收录 |
| Linux | Wayland 走合成器的 data-control（`ext-data-control-v1` 或 `wlr-data-control-unstable-v1`）。没有该协议的会话（含 GNOME）退回 X11 `XFixes` 选择通知，再用 AWT 读取 | 两种会话的写入都走 AWT | 可以继续收录。Wayland 与 X11 的差异是完整性问题，不是“后台被禁止” |
| 无头 / CLI | `HeadlessPasteboardService` 打日志说明不监听系统剪贴板 | 不写系统剪贴板；条目进本地历史并按同步策略发出 | 不适用 |

这三套监听都依赖桌面进程常驻。手机操作系统没有等价能力。

## 3. 技术栈、模块边界和多端缺口

### 3.1 现在的形状

语言和 UI：Kotlin 2.4.21，Compose Multiplatform 1.12.2，Material 3。依赖注入是 Koin。数据库是 SQLDelight 2.4.1（SQLite）。网络是 Ktor 3.5.2。加密在 `core`，桌面用 JDK provider，JS 用 WebCrypto，配对 v3 的 SPAKE2 在桌面通过 JNA 调用 OpenSSL libcrypto。

模块：

| 模块 | 编译目标 | 职责 | 和移动端的关系 |
| --- | --- | --- | --- |
| `core` | `jvm("desktop")`、Kotlin/JS、当前主机的 Kotlin/Native | 平台无关的标识、加密、同步 DTO | 最适合原样复用。JS 已供扩展使用，Native 供 CLI 使用。没有 Android / iOS target |
| `shared` | 同上，但没有 JS | SQLDelight 库、同步运行时、文件与平台工具的 `expect/actual` | 库本身可生成 Android driver，但 Gradle 只挂了 JVM `sqlite-driver` 和 Native driver |
| `shared-ui` | 只有 `jvm("desktop")` | 少量共享 Compose | 未接移动 target |
| `app` | 只有 `jvm("desktop")` | 桌面 UI、同步状态机、剪贴板、发现、托盘、快捷键、OCR、MCP | `commonMain` 约 3.7 万行 Kotlin，`desktopMain` 约 5.1 万行。后者几乎都要按平台重写 |
| `cli` | 五个 Native 目标 | 终端客户端 | 不适合搬到手机 |
| `web` | Vite + Kotlin/JS `core` | Chrome 扩展 | 独立端 |
| `e2e` | `jvm("desktop")` | 桌面端到端 | 移动端要另建 |

`app` 的 `commonMain` 依赖仍挂着桌面库（`kotlinx-coroutines-swing`、`material-desktop`、Ktor server CIO、Compose Desktop）。源码几乎不直接 `import java.*`，和“拷贝到移动工程”的写法一致，但 **这个 Gradle 工程本身编不出 Android 或 iOS**。`ktlint` 的 `android = false` 也说明当前不按 Android 源码检查。

`PasteboardService` 接口在 `commonMain`，三个桌面实现和一个无头实现都在 `desktopMain`。`PasteBonjourService` 同样是接口在 `commonMain`、JmDNS 实现在 `desktopMain`。这是正确的扩展缝，但缝后面没有移动实现。

桌面专有、移动端不能直接复用的部分：

- 剪贴板：AWT、JNA、Swift `NSPasteboard`、Win32、X11、Wayland data-control
- 发现：JmDNS（Java），不是 Android `NsdManager` 或 iOS `NWBrowser`
- 窗口、托盘、全局快捷键、`jnativehook`
- OCR 的 Tesseract 桌面原生包、Sparkle / Conveyor / JBR 打包
- 文件模型以本地路径为主。安卓要用 Storage Access Framework 的 content URI，授权还会过期；iOS 要用 document picker / security-scoped bookmark

可以复用、而且应该保持协议不变的部分：设备身份与配对、剪贴条目模型（文本、颜色、URL、HTML、RTF、图片、文件）、拉取与补拉（`/pull/pasteBatch` 与按 ID 的新路由）、端到端加密。移动端先做文本和图片，文件和 RTF 后做，避免第一期就改协议。

### 3.2 缺口和工作量

| 缺口 | 工作量 | 主要风险 |
| --- | --- | --- |
| 桌面三端本身 | 已完成 | 维护面已经很大：三套剪贴板后端、签名、商店和 Linux 双仓库 |
| Windows ARM 安装包 | 小到中 | 今天 x64 包或可在 ARM 上模拟运行，但是没有原生包。Kotlin/Native 没有与 `mingwX64` 同级的稳定 Windows ARM CLI 目标。只在产品要求“原生 ARM 安装包”时才值得做 |
| 把本仓库收成真正的 KMP Android / iOS 工程 | 大 | `commonMain` 约 3.7 万行是按“拷贝共享”写的，不是按多 target 编译的。一次性把 `app` 改成多 target，会把桌面 CI 和移动工程绑死。更稳的是继续单独的应用工程，只依赖 `core` / `shared` 的源或制品 |
| Android 前台同步客户端 | 大 | 新应用、新打包、Play 政策、局域网权限、剪贴板焦点限制、OEM 杀进程。协议可复用，UI 和剪贴板适配层要新写。私有移动仓库若仍可用，工作量会小于从零开始，但本仓库无法证明这一点 |
| Android 后台剪贴板韧性 | 中（必须先有前台客户端） | 系统不允许后台读内容。这一期只能做“进程还活着时的接收”和“用户主动抓取当前一条”，不能承诺连续监听。见第 4 节 |
| iOS 前台客户端 | 大 | 无 target、无签名流水线。iOS 16 起读取其他应用放入的剪贴板要用户确认或 `UIPasteControl`。没有后台变更通知。本地网络权限从 iOS 14 起就是运行时授权 |
| HarmonyOS | 大，且单独立项 | 只有枚举值。不能并进 Android 应用 |

最大的结构性风险：若把移动端继续做成 `commonMain` 的源码拷贝，桌面和手机会对同一批文件分叉。`PasteCatchUp.md` 已经在提醒旧客户端不会自动获得新的补拉语义。多端补齐时应先规定哪些模块以制品共享（`core`、数据库 schema、同步 DTO），哪些允许平台各自实现（剪贴板、发现、文件、UI 壳）。

## 4. 安卓：后台、被杀、以及系统允许的边界

### 4.1 本仓库的现状

本仓库没有安卓剪贴板代码，因此没有使用 `ClipboardManager`、`OnPrimaryClipChangedListener`、无障碍、输入法或前台服务。第 2.3 节的桌面 API 不能在安卓上调用。

下面判断的是：**如果在合法权限内为 CrossPaste 增加安卓客户端，系统实际允许什么。** 依据是 Android 公开文档和 `ClipboardManager` API 合同，不是对某个已上架安卓包的反编译。

### 4.2 三种进程状态下能做什么

| 状态 | 读取本机新复制的内容 | 监听“发生了一次复制” | 把其他设备的内容写入本机剪贴板并保持局域网同步 |
| --- | --- | --- | --- |
| 有输入焦点的 Activity（`onWindowFocusChanged(true)`，不只是 `onResume`） | 可以。`getPrimaryClip()` 返回当前主剪贴板 | 可以注册 `OnPrimaryClipChangedListener`。有焦点时回调里能读到内容 | 可以。同步连接随前台进程存在 |
| 进程还在，但没有焦点：压到后台、锁屏、只有前台服务或通知 | **不可以读内容。** API 29+ 上 `getPrimaryClip()`、以及文档所述的无焦点访问，返回 `null` / 读不到 | 公开文档没有保证无焦点时仍投递回调；各版本和厂商不一致。即使回调到了，回调里读内容仍会得到 `null`，不能靠它做快照 | 写入没有 `getPrimaryClip` 那条“必须有焦点”的合同。进程活着时可以 `setPrimaryClip`。Android 13+ 会弹出系统的复制确认。连接能否维持见 4.4，和剪贴板读权限是两件事 |
| 进程被系统杀掉、用户从最近任务划掉后厂商杀进程、或用户强制停止 | 不可以。没有代码在跑 | 不可以。系统不保留“待交给 CrossPaste”的剪贴板事件队列 | 不可以，直到用户再次打开应用。强制停止还会停掉启动广播，用户不手动打开就恢复不了 |

回到前台时的补读只有一条：焦点恢复后读当前主剪贴板，用内容哈希或剪贴描述去重，新的才入库。用户在后台连续复制 A、B、C，回到应用时通常只能看到 C；若这期间又过了系统的清空时间，连 C 也没有。

### 4.3 分版本的系统限制

这些限制作用于 **设备系统版本**。把 `targetSdk` 降到 28 不能在 Android 10 以上的设备上恢复后台读取，而且 Google Play 也不接受过期的 target。

| 版本 | 对剪贴板同步的影响 |
| --- | --- |
| Android 9 及更早（API ≤ 28） | 后台可以读剪贴板。新应用不会以这个版本为发布目标，不能当作方案 |
| Android 10–11（API 29–30） | 只有默认输入法，或当前拥有输入焦点的应用，可以读剪贴板。官方表述：Otherwise your app cannot access clipboard data |
| Android 12 / 12L（API 31–32） | 继续要求焦点或默认输入法。应用读取 **其他应用** 放入的剪贴板时，系统会弹 Toast（“已从剪贴板粘贴”一类提示）。这是用户可见的隐私提示，前台同步也会触发，不是故障 |
| Android 13（API 33） | 系统在复制时显示标准预览。敏感内容要用 `ClipDescription.EXTRA_IS_SENSITIVE`（或 `android.content.extra.IS_SENSITIVE`）挡住预览。主剪贴板会在一段时间后被系统清空；公开安全说明确认是 Android 13 起的行为，AOSP 默认时长约 1 小时（3,600,000 ms），厂商配置可以改 |
| Android 14（API 34） | 前台服务必须声明类型，否则调用 `startForeground` 会失败。类型选错会被 Play 拒绝 |
| Android 15（API 35），且应用 target 为 35+ | `dataSync` 前台服务在 24 小时内累计最多约 6 小时，超时必须 `stopSelf`，否则系统抛 `RemoteServiceException`。用户把应用带回前台会重置计时。`BOOT_COMPLETED` 里不允许再启动 `dataSync` 前台服务 |
| Android 16（API 36） | 局域网保护仍处在 opt-in 阶段：不主动打开兼容开关时，仅有 `INTERNET` 仍可访问局域网 |
| Android 17（target 37）起，按官方路线 | 局域网访问改为强制。原始套接字、mDNS、`NsdManager` 都要 `ACCESS_LOCAL_NETWORK`，否则局域网连接失败，公网不受影响。CrossPaste 只有局域网，这条权限是必做项。系统也提供 NSD 设备选择器，用来避免“扫描整个局域网”的宽权限；点对点自动发现全部已配对设备，和“每次选一台”不是同一产品，计划阶段就要选定 |

另外没有第三种公开权限可以申请：不存在给普通应用的 `READ_CLIPBOARD` 运行时权限。`getPrimaryClip()` 的合同就是焦点或默认输入法。

输入焦点要在窗口真正 focused 之后读取。只在 `onResume` 里读，可能还没有焦点，结果仍是 `null`。官方讨论区给出的时机是 `onWindowFocusChanged(true)`。

### 4.4 进程还活着时，同步连接本身也有上限

即使完全不读剪贴板，要在后台接收桌面端推来的条目，进程也必须活着，并且能进局域网。

- Android 8 以后，长期后台工作需要前台服务和一条用户看得到的通知。
- `dataSync` 能表达“正在同步”，但 API 35+ 有 6 小时 / 24 小时上限，不适合当作“全天在线”。
- `connectedDevice` 的文档覆盖蓝牙、NFC、USB 或网络连接，清单里可以声明 `CHANGE_NETWORK_STATE`、`CHANGE_WIFI_STATE` 或 `CHANGE_WIFI_MULTICAST_STATE`。它没有 `dataSync` 那条 6 小时上限。用它承载局域网同步有文档上的空间，但是否被 Play 接受为“外部设备连接”要单独做政策预审，不能直接当实现结论。
- 官方把 Companion Device Manager 列为 `connectedDevice` 的替代：用户关联一台设备后，设备在附近时应用更容易保持运行。它 **不** 授予剪贴板读取。桌面电脑是否算合格的 companion，需要在真机和 Play 预审里验证。
- 国产 ROM 在用户划掉最近任务、省电策略、自启动限制下会杀掉带通知的前台服务。产品文案只能引导用户把应用加入电池白名单，不能从技术上保证。
- 用户强制停止之后，在用户再次手动打开之前，启动广播和后台拉起都不可用。

所以“后台同步”若指 **接收并写入剪贴板**，在进程存活且前台服务策略允许的时间窗内可行。若指 **在后台继续发现并上传用户在其他应用里的每一次复制**，不可行。

### 4.5 可行方案

下列做法使用公开 API，用户能理解自己授权了什么，也不把无障碍或输入法当成剪贴板后门。

1. **前台监听（主路径）**  
   Activity 获得输入焦点后注册 `OnPrimaryClipChangedListener`，在回调和 `onWindowFocusChanged(true)` 里调用 `getPrimaryClip()`。失去焦点立即停止把 `null` 当成空剪贴板。只同步文本、HTML 和带持久授权的 URI。第一期可以不做 RTF。敏感条目写入时带上 `EXTRA_IS_SENSITIVE`。

2. **回到前台补读最后一条**  
   再次获得焦点时读当前主剪贴板，与上次入库的哈希比较。相同则跳过。向用户说明：离开应用期间只保留最后一条，且可能被系统清空。这是合规上限，不是暂时的实现缺口。

3. **用户点一下，抓取当前一条**  
   快捷设置磁贴、应用快捷方式、小部件或通知动作启动一个透明 Activity，在拿到焦点后读取、去重、同步，然后 `finish()`。每次点击只处理当时的主剪贴板。磁贴服务本身没有剪贴板焦点，不能在 `TileService` 里直接读。

4. **用户显式送入**  
   接收 `ACTION_SEND` 和 `ACTION_PROCESS_TEXT`。这不是监听，是用户把选中文本交给 CrossPaste。适合作为后台监听的替代入口，应在第一期就有。

5. **接收方向，与读取分开**  
   应用在前台，或在已经向用户说明的前台服务存活期间，用 `setPrimaryClip` 写下其他设备推来的文本。预期 Android 12+ 的读取提示和 Android 13+ 的复制预览会出现。不要为了静默写入去申请特殊权限。

6. **不依赖剪贴板的历史同步**  
   桌面端已经收录的历史，手机打开后按现有拉取接口补齐。这条路径不受剪贴板焦点限制，只受局域网权限和进程是否被用户打开的限制。它应该是移动端“可靠”的定义：打开即可对齐历史；实时收录只覆盖手机应用自己处于前台的时段。

7. **局域网权限按 Android 17 的强制模型设计**  
   清单和运行时都规划 `ACCESS_LOCAL_NETWORK`（在强制实施前用官方的 opt-in 开关做测试）。发现用 `NsdManager`，不要把 JmDNS 搬进安卓进程。若产品接受“系统选择器逐台授权”，可以少要宽权限；若产品要求自动看到所有已配对设备，就要宽权限和被拒绝时的说明页。

### 4.6 不可行边界

下面这些做法能在技术上碰到剪贴板，但超出本次允许的产品边界，或者会直接违反商店政策。评估不提供实现步骤。

| 做法 | 为什么不做 |
| --- | --- |
| 无障碍服务（`AccessibilityService`）只为了监听剪贴板或窗口文字 | Play 要求无障碍 API 用于帮助残障用户。用它绕过剪贴板焦点限制属于滥用。本评估明确排除 |
| 空壳输入法，只为了取得“默认 IME 可读剪贴板” | 系统确实给默认输入法开了读权限。一个不提供真实输入能力的 IME 是欺骗性权限提升，Play 对输入法有单独要求。不做。若将来单独做一款真实键盘，那是另一个产品，不在这条路线里 |
| 用自动填充服务、设备管理员、设备所有者身份读剪贴板 | 这些身份不是剪贴板同步的正当用途，普通用户也无法、也不应授予 |
| 透明悬浮窗、`SYSTEM_ALERT_WINDOW` 抢焦点后轮询 | 用焦点规则的空子，Android 12 以后对不可信触摸和悬浮窗焦点更严，Play 也会视为干扰 |
| 反射或隐藏 API 调用 `ClipboardService` | 非 SDK 接口会被平台屏蔽，也过不了政策 |
| root、Shizuku、无线调试 shell | 用户机器上的高权限通道，不是应用的正常权限。不作为产品方案 |
| 降低 `targetSdk`，或只支持 Android 9 | 恢复不了 Android 10+ 设备上的后台读取，也无法上架 |
| WorkManager 定期读剪贴板 | 后台任务没有输入焦点，`getPrimaryClip()` 为 `null` |
| 开机广播里启动监听 | 没有焦点，读不到；API 35+ 还禁止从 `BOOT_COMPLETED` 启动 `dataSync` 前台服务 |
| 把通知栏文字当作剪贴板 | 通知监听是另一项敏感能力，文本也不等于剪贴板。不做 |

iOS 不在本仓库，但路线图不应假设它比安卓松。iOS 9 起就只有前台能读通用粘贴板；应用在后台收不到 `UIPasteboardChangedNotification`，只会在再次激活时发现 `changeCount` 变了。iOS 14 对未经用户意图的读取给出系统提示，iOS 16 起通常要用户确认，或使用系统提供的 `UIPasteControl`。没有公开的后台监听 API。

## 5. 分阶段路线图

每一期都只做合法功能。验收标准写成可以在真机上打勾的行为，不把“后台连续收录”写成目标。

### 阶段 0：边界写进计划（本文件）

交付物：本评估。不改协议、不改应用代码。

验收：

- 阅读的人能区分“本仓库已发布的三端”和“协议里出现、源码不在本仓库的移动端”。
- 安卓后台读剪贴板被标成做不到，而不是待办功能。
- 后续实现不包含无障碍监听、空壳输入法、隐藏 API。

### 阶段 1：安卓前台同步客户端

工作量：大。

交付物：

- 独立的 Android 应用工程（或在确认私有移动仓库仍是主线之后，在那个工程上补齐），通过制品或受控源码依赖 `core` 与 `shared`，而不是把整个 `app/commonMain` 和桌面 Gradle 绑在一起。
- `PasteboardService` 的安卓实现：只在拥有输入焦点时读取和监听。
- `NsdManager` 发现，Ktor 客户端走现有配对和拉取协议。手机前台时可以连上桌面端。
- 文本、URL、纯文本化的 HTML。图片若剪贴板里是本应用能持久读取的 URI 再做，否则放到阶段 3。
- 分享入口与“处理文本”入口。
- 局域网权限的设置说明；用 Android 16 的 opt-in 限制做一次失败路径测试。

验收：

- 同一局域网内与一台已发布桌面端完成配对。手机 CrossPaste 处于前台时复制一段文本，桌面端出现同一条。桌面端复制一段文本，手机前台历史里出现，并且可以写入手机剪贴板。
- 手机切到其他应用之后，在其他应用里再复制两条。手机 CrossPaste 不产生新纪录。回到 CrossPaste 并获得焦点后，最多新增仍留在主剪贴板上的那一条。
- 过程中不出现无障碍服务、输入法和无障碍权限申请。
- 最低覆盖 API 29、31、33、35。API 36 至少跑通局域网 opt-in 被打开时的失败和授权恢复。

### 阶段 2：安卓后台韧性，停在系统允许的地方

工作量：中。依赖阶段 1。

交付物：

- 用户可开关的前台服务，通知文案写明“保持与已配对设备的连接”，不写“后台监听剪贴板”。
- 选定一种前台服务类型并完成 Play 声明预审。若用 `dataSync`，实现 `onTimeout` 并在超时后停止，界面说明每天的在线时间有限。若用 `connectedDevice` 或 Companion Device，先拿到政策预审结论再写进发布说明。
- 快捷设置磁贴或等价的一键入口：透明 Activity 取得焦点、读取当前一条、同步、关闭。
- 电池优化说明，明确列出无法克服的厂商杀进程和强制停止。

验收：

- 服务运行中，桌面端发来的文本能写入手机剪贴板；系统复制预览可以出现。
- 服务运行中，用户在其他应用复制的内容 **不会** 被自动上传。这是通过，不是失败。
- 划掉最近任务或强制停止之后，不再接收、不再读取，直到用户手动打开应用或点击磁贴。磁贴点击只上传当时那一条。
- `dataSync` 方案：模拟超时后服务停止且进程不崩溃。
- 真机至少包含一台原生 Android 15+ 和一台会杀后台的国产 ROM，结果写入测试记录。

### 阶段 3：iOS 前台客户端

工作量：大。与阶段 2 无依赖，但不要和阶段 1 并行到共享层还没定稿的时候。

交付物：

- 独立 iOS 工程，复用同一套同步 DTO 和加密，不从本桌面仓库直接出 IPA。
- 仅在前台比较 `UIPasteboard.general.changeCount` 并读取。iOS 16+ 使用系统粘贴确认或 `UIPasteControl`，不尝试静默读取。
- 本地网络权限说明。不注册任何后台模式来读粘贴板。

验收：

- 应用在前台时，与桌面端双向同步文本。
- 应用进入后台或被杀掉后不产生新的本机复制纪录；再次打开时最多补到当前仍在的那一条，且系统粘贴提示的行为与文档一致。
- App Store 隐私说明不声明后台剪贴板访问。

### 阶段 4：文件、图片和富文本

工作量：中到大。桌面端这些类型已经可用，移动端是新适配。

交付物：安卓 SAF、iOS 安全作用域书签；图片缩略图；明确不做或降级 RTF。不改现有条目的线上字段，移动端不认识的类型显示为不可预览但仍可删除。

验收：一张图片和一个用户主动选出的文件能从手机前台传到桌面并打开；桌面发来的文件在手机上以内容 URI 或书签保存，重启后行为符合授权是否仍在。后台不新增读取能力。

### 建议不要排进这条线的项

- 用无障碍或输入法补齐“后台每一次复制”。
- 为了移动端改动桌面已经发布的身份字符串或加密格式。
- 在本仓库的桌面 CI 里同时编 Android / iOS。移动应用保持独立工程，桌面仓库只提供它已经稳定的 `core` / `shared` 契约。
- HarmonyOS，除非单独评估，不作为安卓应用的附带目标。
- Windows ARM 原生包。与剪贴板多端无关；只有在明确要原生 ARM 安装包时另立小到中的桌面任务。

## 6. 证据索引

- 端支持声明：`README.md`、`README.zh-CN.md`、`doc/en/Roadmap.md`
- 工程目标：`settings.gradle.kts`，各模块 `build.gradle.kts` 的 `kotlin { jvm("desktop") }`，`cli/build.gradle.kts` 的五个 Native 目标
- 打包：`conveyor.conf`，`app/build.gradle.kts` 的 `nativeDistributions`，`app/jbr.yaml`
- 发布：`.github/workflows/build-release.yml`、`publish-release.yml`、`msstore-publish.yml`、`build-beta-linux.yml`，以及 `.github/scripts/package_appimage.sh`、`package_rpm.sh`
- 移动端只存在于契约和文档：`core/.../Platform.kt`，`doc/en/PasteCatchUp.md`，`doc/en/I18nStyleGuide.md`
- 桌面剪贴板：`MacosPasteboardService.kt`，`app/src/desktopMain/swift/MacosApi.swift`，`WindowsPasteboardService.kt`，`LinuxPasteboardService.kt`，`WaylandClipboardMonitor.kt`，`HeadlessPasteboardService.kt`
- 安卓系统行为：
  - [Android 10 剪贴板访问限制](https://developer.android.com/about/versions/10/privacy/changes#clipboard-data)
  - [ClipboardManager.getPrimaryClip](https://developer.android.com/reference/android/content/ClipboardManager#getPrimaryClip())
  - [复制与粘贴（Android 12 Toast、Android 13 预览与敏感标记）](https://developer.android.com/develop/ui/views/touch-and-input/copy-paste)
  - [安全剪贴板处理（Android 13 起定时清除）](https://developer.android.com/privacy-and-security/risks/secure-clipboard-handling)
  - [前台服务超时](https://developer.android.com/develop/background-work/services/fgs/timeout)
  - [前台服务类型](https://developer.android.com/develop/background-work/services/fgs/service-types)
  - [局域网权限](https://developer.android.com/privacy-and-security/local-network-permission)
- iOS 对照：[UIPasteboard](https://developer.apple.com/documentation/uikit/uipasteboard)、[changeCount](https://developer.apple.com/documentation/uikit/uipasteboard/changecount)
