# ImGui MD3 — 安卓原生 ImGui + Material Design 3 界面框架

用 **Dear ImGui** 在安卓上从零手写一套 **Material Design 3** 风格的界面框架。
所有控件（卡片、滑块、开关、导航栏、对话框、列表、输入框…）都是自己绘制，不依赖任何 Android 原生 View / Compose，
Java 只负责开一个 `GLSurfaceView`、把触摸点和按键转交给 C++，界面 100% 由 C++ / ImGui 渲染。

适合做：游戏内悬浮菜单、调试工具面板、原生性能可视化工具、自绘 UI 实验。

---

## 1. 特性

- **Material Design 3 视觉**：动态取色（种子色）、明暗主题、圆角/阴影/涟漪、Elevation 层次
- **自绘控件集**：Card / FilledButton / OutlineButton / IconButton / Slider / Switch / Checkbox / TextField / Dialog / Snackbar / NavigationBar / Tab / Chip / ListItem / ProgressBar ……
- **动效系统**：页面切换、折叠淡出、运动模糊、阻尼滚动、速度吸附
- **多语言**：内置 i18n 表，运行时可切换
- **图标**：内置矢量图标绘制（ImageVector 风格多段路径），无需图片资源
- **中文字体**：内置思源黑体（`assets/SourceHanSansCN-Bold.otf`），FreeType 后端，中文不再显示 `???`
- **TTS 朗读**：调用系统离线语音引擎，语速/音调多档
- **AI 对话 & MCP**：内置对话页与 MCP（Model Context Protocol）管理器 —— **接口/密钥需要你自己填，见第 5 节**
- **纯 C++ 内存读写实验室（实验风味）**：`/proc/self/maps` + `/proc/self/mem` 的自进程扫描/读/写示例
- **多风味打包**：一个工程同时出多个互不覆盖的 APK

---

## 2. 环境要求（务必对齐版本）

| 组件 | 版本 | 说明 |
|---|---|---|
| Android Studio | Ladybug 2024.2.1 及以上 | 用命令行构建也可以，AS 只是方便 |
| JDK | **17** | AGP 8.x 要求 |
| Gradle | **8.1.1**（工程已带 wrapper） | 不用自己装，`gradlew` 会下载 |
| Android Gradle Plugin | **8.0.0**（根 `build.gradle.kts` 里） | |
| Android SDK | **compileSdk 33 / targetSdk 33 / minSdk 21** | |
| **Android NDK** | **27.1.12297006** | 必须装上这个版本，或与 `app/build.gradle.kts` 里 `ndkVersion` 保持一致 |
| CMake | **3.22.1**（SDK 里自带的即可） | |
| Python | 3.8+（无需，仅打包脚本用） | |

> 关键点：**NDK 版本必须和 `app/build.gradle.kts` 中的 `ndkVersion` 一致**，否则 CMake 配置阶段直接报错。
> 想换 NDK，就同时改 `ndkVersion` 和下面命令里的版本号。

---

## 3. 下载 SDK / NDK（命令行）

先装 Android SDK 命令行工具（`cmdline-tools`），假设 SDK 根目录为 `$ANDROID_HOME`
（Windows 一般是 `C:\Users\你\AppData\Local\Android\Sdk`，macOS/Linux 一般是 `~/Android/Sdk`）。

`sdkmanager` 位于 `$ANDROID_HOME/cmdline-tools/latest/bin/`。

```bash
export ANDROID_HOME=$HOME/Android/Sdk
export PATH=$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$PATH

# 接受许可
yes | sdkmanager --licenses

# 平台与构建工具
sdkmanager "platform-tools" \
           "platforms;android-33" \
           "build-tools;33.0.2"

# ★ NDK（本工程使用的版本）
sdkmanager "ndk;27.1.12297006"

# CMake
sdkmanager "cmake;3.22.1"

# 查看已安装
sdkmanager --list_installed
```

国内下载慢，可先设置镜像（清华 TUNA）：

```bash
# ~/.android/repositories.cfg 不存在就先建一个空文件
touch ~/.android/repositories.cfg
# 在 sdkmanager 支持的环境里设置代理镜像（或使用 Android Studio 里的镜像设置）
# 亦可直接用代理，例如：
# export https_proxy=http://127.0.0.1:7890
```

NDK 装好后检查：

```bash
ls $ANDROID_HOME/ndk/27.1.12297006/    # 应能看到 toolchains/ 、source.properties
```

---

## 4. 构建

### 4.1 准备

```bash
git clone <你的仓库地址> ImGui-MD3
cd ImGui-MD3

# 让 Gradle 找到 SDK（local.properties 已被 .gitignore 忽略，需要自己建）
echo "sdk.dir=$ANDROID_HOME" > local.properties
```

### 4.2 一条命令出包

```bash
# 主 App（MD3 界面）
./gradlew assembleFullDebug

# 所有风味一起打
./gradlew assembleDebug
```

产物位置：

```
app/build/outputs/apk/full/debug/app-full-debug.apk
```

Windows 用 `gradlew.bat assembleFullDebug`。

### 4.3 三种产物风味（productFlavors）

| 风味 | 包名 | 用途 |
|---|---|---|
| `full` | `com.u3dexample` | **主界面**：MD3 UI 本体 |
| `proctest` | `com.u3dexample.proctestapp` | 原版 ImGui 皮肤的内存读写/进程测试页 |
| `range` | `com.u3dexample.practice` | “靶场”App：一块带标签的靶子内存，供上面那个页面对照练习 |

只想保留主界面，把 `app/build.gradle.kts` 里不想要的 `create("xxx") { ... }` 整段删掉即可。

### 4.4 在 ARM64 设备/ARM64 Linux 上构建（进阶，可选）

本项目作者是在 **aarch64 安卓设备**上构建的，需要的额外处理：

1. **AAPT2 架构问题**：Google 提供的 `aapt2` 只有 x86_64 版本，ARM64 上需要自己编译或找 arm64 版 aapt2，
   然后在 `gradle.properties` 里指定：
   ```properties
   android.aapt2FromMavenOverride=/你的/aapt2/绝对路径
   ```
   （本工程里的这一行默认已注释掉，x86_64 机器上不需要它。）
2. **不要在 FUSE 挂载目录（如 `/storage/emulated/0/...`）里构建**，
   Gradle/CMake 对 FUSE 的 `exec` 和文件锁支持很差，请把工程拷到原生文件系统（如 `/root/imgui_md3`）再构建。
3. **NDK 工具链**：如果 NDK 里的 `clang` 也是 x86_64，同样需要 qemu-user 包装
   （已验证可用 qemu 包装的 NDK 工具链完整编译本工程），或者用 Termux 里的 arm64 NDK。

---

## 5. ★ 必须自己填的 AI 接口（重要）

本仓库**不包含任何可用的 AI 密钥 / 服务地址**，所有密钥位都已被替换为占位符。
想体验 AI 相关功能，请自己申请账号并填写：

### 5.1 对话接口

打开 `app/src/main/java/com/u3dexample/AiChatBridge.java`，你会看到：

```java
private static final String[] API_KEYS = new String[] {
        "YOUR_API_KEY_HERE",      // ← 换成你自己的 key
};
```

同文件内还有 **接口地址（BASE_URL / endpoint）** 与 **模型名（model）**，按你使用的服务商填写，
只要求 **兼容 OpenAI 的 `/v1/chat/completions` 协议**（绝大多数服务商都兼容）：

```java
// 形如：
// https://你的服务商/v1/chat/completions
// 请求头：Authorization: Bearer <你的 key>
// 请求体：{"model":"你的模型名","messages":[...],"stream":false}
```

多把 key 会自动轮询（`API_KEYS` 数组可以放多个，用于限流/均衡）。

### 5.2 TTS 语音

TTS 走的是系统自带离线语音引擎，不需要密钥，在
**设置 → AI 人格/模式/知识库 → 朗读** 里启用并试听即可（语速、音调各 5 档）。

### 5.3 MCP 服务器

界面里 **设置 → MCP** 可以添加 MCP 服务。需要鉴权的，点「编辑」，
在 Header 里填 `Authorization: Bearer <token>`。这些内容只存在本地，不会上传。

> 注意：任何硬编码在客户端里的密钥都能被反编译提取，正式产品请把密钥放在**自己的后端**，客户端只调你的后端。

---

## 6. 常见改动怎么做

### 6.1 改包名 / 应用名 / 图标 / 版本号

`app/build.gradle.kts`：

```kotlin
android {
    defaultConfig {
        applicationId = "com.你的包名"     // 包名
        versionCode = 1
        versionName = "1.0"
    }
    productFlavors {
        create("full") { applicationId = "com.你的包名" /* ... */ }
    }
}
```

应用名：`app/src/main/res/values/strings.xml` 里的 `app_name`；
各风味可单独覆盖：`app/src/<风味>/res/values/strings.xml`。

图标：替换 `app/src/main/res/mipmap-*/ic_launcher.png`
（或用 Android Studio 的 Image Asset 生成 mipmap-anydpi-v26 自适应图标）。

### 6.2 改主题 / 配色

- 主题与取色：`app/src/main/cpp/md3_theme.cpp`
- 默认是黑白极简种子色，想换成品牌色，改 `md3_theme.cpp` 里的种子色常量即可，派生色会自动重算。
- 深色/浅色：设置页有开关，也可以改默认值。

### 6.3 改字体 / 加中文字体

`app/src/main/assets/` 放 `.otf`/`.ttf`，然后在 JNI 初始化字体处改成你的文件名
（字体加载在 `md3_jni.cpp` 里，通过 `AAssetManager` 读进内存后 `AddFontFromMemoryTTF`）。

### 6.4 加一门语言 / 文案

`app/src/main/cpp/md3_i18n.cpp`，按现有表结构往里加 `key → 文案` 即可，运行时切换。

### 6.5 加一个页面

1. 在 `md3_pages.cpp` 里仿照已有页面写绘制函数；
2. 在页面枚举/路由表里注册；
3. 侧栏或底部导航里加一项。

### 6.6 加一个新控件

`md3_controls.cpp`（实现）+ `md3_uicon.hpp`（声明）；图标在 `md3_icons.cpp` 里用矢量路径画。

---

## 7. 目录结构

```
.
├── app/
│   ├── build.gradle.kts            # 模块配置：NDK 版本、风味、CMake 接线
│   └── src/
│       ├── main/
│       │   ├── AndroidManifest.xml
│       │   ├── assets/             # 字体等
│       │   ├── java/com/u3dexample/
│       │   │   ├── CjImGuiActivity.java     # 主界面壳（GLSurfaceView + 触摸/按键转发）
│       │   │   ├── AiChatBridge.java        # ★ AI 接口（密钥需要你自己填）
│       │   │   ├── McpManager.java          # MCP 管理
│       │   │   ├── ProcTestActivity.java    # 原版 ImGui 测试页 / 内存实验室
│       │   │   └── RangeActivity.java       # 靶场
│       │   └── cpp/
│       │       ├── CMakeLists.txt
│       │       ├── md3_jni.cpp          # JNI 入口、帧循环、字体、触摸
│       │       ├── md3_pages.cpp        # 页面
│       │       ├── md3_controls.cpp     # 控件
│       │       ├── md3_theme.cpp        # MD3 取色/主题
│       │       ├── md3_icons.cpp        # 矢量图标
│       │       ├── md3_i18n.cpp         # 多语言
│       │       ├── md3_anim.cpp / md3_uianim.cpp / md3_motionblur.cpp   # 动效
│       │       ├── md3_net.cpp / md3_ai.cpp / md3_mcp.cpp               # 网络与 AI
│       │       ├── md3_tts.cpp / md3_log.cpp / md3_fx.cpp
│       │       ├── proctest.cpp         # 原版 ImGui 测试页
│       │       ├── rangevalue.cpp       # 靶场靶子内存
│       │       ├── imgui-1.91.8/        # Dear ImGui 源码
│       │       ├── freetype/            # FreeType（中文字体）
│       │       └── cubism/ live2d/      # Live2D Cubism（立绘渲染，可整块删掉）
│       ├── proctest/AndroidManifest.xml
│       └── range/AndroidManifest.xml
├── build.gradle.kts
├── settings.gradle.kts
├── gradle.properties
├── gradlew / gradlew.bat
└── gradle/wrapper/
```

第三方库（imgui / freetype / cubism / live2d）各保留自己的许可证，二次分发请遵守其协议。

---

## 8. 常见问题

| 现象 | 原因 / 解决 |
|---|---|
| `No version of NDK matched the requested version` | `ndkVersion` 与已安装 NDK 不一致。装 `sdkmanager "ndk;27.1.12297006"` 或改 `ndkVersion` |
| `AAPT2 ... error ...` / `aapt2: not executable` | 架构不匹配（ARM64 上跑 x86_64 aapt2）。按 4.4 处理 |
| CMake 配置报 `Failed to find CMake` | `sdkmanager "cmake;3.22.1"` |
| 构建卡住 / 文件锁错误 | 工程在 FUSE 目录（`/storage/emulated/0`）。换到原生文件系统 |
| 界面中文全是 `???` | 没加载中文字体。确认 `assets` 里的字体文件存在且参数 `IMGUI_ENABLE_FREETYPE` 打开 |
| 触摸点有偏移 | 触摸要挂在 `GLSurfaceView.setOnTouchListener`（View 相对坐标），不要挂在 `Activity.onTouchEvent`（含状态栏偏移） |
| AI 功能不工作 | 第 5 节：密钥/地址没填，或模型名不对 |

---

## 9. 免责声明

本项目仅用于 **学习与研究**（图形界面、ImGui 渲染、安卓 JNI、性能优化）。
其中的“内存实验室 / 靶场”风味只对 **本工程自己的示例进程** 做读写演示，
**不包含**也不应被用于绕过任何游戏或软件的授权、修改他人进程数据等用途。
请勿将本项目用于任何违反当地法律法规或第三方服务条款的场景。使用者自行承担全部后果。

---

## 10. 许可

代码以你选择的许可证开源（建议 MIT / Apache-2.0；请自行补一个 `LICENSE` 文件）。
`imgui-1.91.8`、`freetype`、`cubism`、`live2d` 目录中的代码版权归各自作者所有，遵循其原始许可证。
