# 怎么把它变成能装的 APK

我交付的是**完整可编译的工程源码**，不是现成的 `.apk` 文件（原因见 [README](README.md#关于-apk-二进制)）。
下面三种方式任选一种，最推荐 **方式 A**：不用装任何开发环境，云端自动出包。

| 方式 | 需要装什么 | 难度 | 出包时间 |
|---|---|---|---|
| **A. GitHub Actions 云构建** | 只要一个 GitHub 账号 | ⭐ | 首次约 5 分钟 |
| B. Android Studio 本地构建 | Android Studio（约 3GB） | ⭐⭐ | 首次约 15 分钟 |
| C. 命令行 Gradle 构建 | JDK 17 + Android SDK + Gradle | ⭐⭐⭐ | 首次约 10 分钟 |

---

## 方式 A：GitHub Actions 云构建（推荐）

原理：把工程推到 GitHub，GitHub 免费给你一台机器编译，编完把 APK 作为"产物"提供下载。
**你不需要在电脑上装任何安卓开发工具。**

### A1. 注册 GitHub 账号

打开 https://github.com/signup ，用邮箱注册一个免费账号（如果已经有就跳过）。

### A2. 新建一个仓库

1. 登录后点右上角 **+** → **New repository**
2. **Repository name** 填 `flipcard-helper-android`
3. 选 **Public**（私有仓库也能用，但 Actions 有免费额度限制；公开仓库完全免费）
4. **不要**勾选 "Add a README file"
5. 点 **Create repository**

### A3. 把代码传上去

建完仓库后页面会给你几种上传方式，选**最简单的那种**：

**方式一：网页拖拽（不用装 git）**

1. 在刚建好的空仓库页面，点 **uploading an existing file** 链接
2. 把 `flipcard-android` 文件夹**里面的所有内容**（不是文件夹本身）拖进上传框
   - 包括 `.github`、`app`、`gradle`、`tools`、`build.gradle`、`settings.gradle` 等
3. 下方 Commit changes 填个说明，点 **Commit changes**

> ⚠️ 注意：`.github` 和 `.gitignore` 是**隐藏文件夹/文件**，在 Windows 资源管理器里要先打开
> 「查看 → 显示 → 隐藏的项目」才能看到并拖进去。漏了 `.github` 就没有自动构建了。

**方式二：命令行（熟悉 git 的话更快）**

在 `flipcard-android` 目录下打开终端：

```bash
git init
git add .
git commit -m "翻牌助手 Android 版"
git branch -M main
git remote add origin https://github.com/<你的用户名>/flipcard-helper-android.git
git push -u origin main
```

### A4. 等它自动编译

推送完成后：

1. 打开你的仓库页面，点上方 **Actions** 标签
2. 会看到一条叫 **Build APK** 的工作流正在跑（黄色转圈）
3. 点进去可以看到两个任务：
   - **算法内核回归测试** —— 先跑，几秒钟
   - **构建 APK** —— 后跑，约 3~5 分钟
4. 两个都变成绿色 ✅ 就成功了

> 如果 Actions 标签页提示需要手动开启，点一下 **I understand my workflows, go ahead and enable them** 即可。
> 也可以随时点 **Run workflow** 按钮手动触发一次构建。

### A5. 下载 APK

1. 在 Actions 页面点进**最近一次成功的**运行记录
2. 拉到页面最底部 **Artifacts** 区域
3. 点 **flipcard-apk** 下载（是个 zip，里面有两个 apk）
   - `app-debug.apk` —— 调试版，包名带 `.debug`，可以和正式版共存
   - `app-release.apk` —— **推荐装这个**（已用调试签名签过，可以直接安装）
4. 解压得到 apk 文件

### A6. 装到手机

见下面 [安装到手机](#安装到手机)。

---

## 方式 B：Android Studio 本地构建

1. 下载安装 **Android Studio**：https://developer.android.com/studio
   安装时一路下一步，让它自动装好 SDK 就行。
2. 启动 Android Studio → **Open** → 选中 `flipcard-android` **文件夹**（不是里面的文件）
3. 首次打开会自动 **Gradle Sync**，右下角有进度条。
   工程里带了 `gradle/wrapper/gradle-wrapper.properties`，Android Studio 会按它自动下载 Gradle 8.9。
   - 如果提示 "Gradle wrapper not found / 找不到 gradle-wrapper.jar"，在弹窗里选
     **Use Gradle from: gradle-wrapper.properties file**，或者选 **Specified location** 后
     在终端里跑一次 `gradle wrapper --gradle-version 8.9` 生成 wrapper。
4. Sync 完成后：菜单 **Build → Build App Bundle(s) / APK(s) → Build APK(s)**
5. 完成后右下角弹提示，点 **locate** 就能找到 apk，一般在：
   ```
   app/build/outputs/apk/debug/app-debug.apk
   app/build/outputs/apk/release/app-release.apk
   ```

---

## 方式 C：命令行 Gradle 构建

需要：JDK 17、Android SDK（含 `platforms;android-34`、`build-tools;34.0.0`）、Gradle 8.9。

```bash
# 1. 确认 JDK 是 17（不要用 21+/25，AGP 8.5 可能不兼容）
java -version

# 2. 指定 SDK 路径（或用环境变量 ANDROID_HOME）
echo "sdk.dir=/path/to/android-sdk" > local.properties

# 3. 直接构建（本工程不带 gradle-wrapper.jar，所以用系统 gradle）
gradle assembleDebug assembleRelease --no-daemon

# 4. 产物在这里
ls app/build/outputs/apk/*/*.apk
```

如果想用 `./gradlew`，先生成一次 wrapper：

```bash
gradle wrapper --gradle-version 8.9
./gradlew assembleDebug
```

---

## 安装到手机

1. 把 apk 传到手机（微信文件传输助手 / QQ / 数据线 / 网盘都行）
2. 在手机上点开这个 apk
3. 系统会提示「禁止安装未知应用」→ 点**设置** → 打开「允许来自此来源的应用」
4. 返回继续安装
5. 装完桌面会出现 **翻牌助手** 图标

> 如果提示「解析包出现问题」，通常是下载不完整或 Android 版本低于 8.0（本工程最低支持 Android 8.0）。

---

## 首次使用（装完之后）

打开 App，主页就一张卡片 + 一个大按钮，按顺序点亮三个绿灯：

1. **无障碍服务** → 点右边「去开启」→ 在系统列表里找到「翻牌助手」并打开开关
2. **屏幕录制** → 点右边「去授权」→ 系统弹窗点「允许 / 立即开始」
3. **牌阵标定** → 点右边「去标定」→ 按页面上的四步走

三个都变绿之后，把游戏停在「所有牌都盖着」的画面 → 回 App 点**开始自动配对** →
提示"5 秒后自动开始" → 立刻切回游戏 → 看着它自己翻。

**完整的图文步骤在 App 里点「图文教程」就有**，和手机屏幕一样宽，配了插图。

---

## 构建相关常见问题

| 问题 | 原因与解决 |
|---|---|
| Actions 里没有 Build APK 这个 workflow | `.github/workflows/build.yml` 没上传成功。注意 `.github` 是隐藏目录，Windows 资源管理器要开「显示隐藏的项目」 |
| Actions 报 `Could not resolve com.android.tools.build:gradle:8.5.2` | 网络问题，重跑一次即可；国内网络偶尔会失败 |
| 本地构建报 `Unsupported class file major version 69` | 你用的是 JDK 25 之类的新版本。换成 JDK 17（Android Studio 自带的就行） |
| 本地构建报找不到 Android SDK | 在工程根目录建 `local.properties`，写 `sdk.dir=你的SDK路径` |
| 提示 `gradle-wrapper.jar not found` | 本工程没带这个二进制。见方式 B 第 3 步，或方式 C 里先跑 `gradle wrapper` |
| 装的时候提示「应用未安装」 | 手机上已经有同包名但签名不同的版本，先卸载旧的 |
| release 包能装吗 | 能。工程里 release 用了 debug 签名，就是为了让 CI 产物直接可装。**但不要拿它上架应用商店** |

---

## 工程结构

```
flipcard-android/
├── .github/workflows/build.yml     云端构建流水线
├── app/
│   ├── build.gradle                模块配置（namespace / SDK 版本 / 依赖）
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── java/com/flipcard/helper/
│       │   ├── core/               ★ 纯 JDK 算法内核（本机 javac/java 已验证）
│       │   ├── data/Prefs.java     配置档案存取
│       │   ├── service/            录屏服务 / 无障碍服务 / 引擎
│       │   └── ui/                 主页 / 标定页 / 设置页 / 教程页
│       ├── res/                    布局、主题、字符串、图标
│       └── assets/tutorial/        内置图文教程（HTML + 插图）
├── tools/
│   ├── coretest/                   Java 内核回归测试 + 测试数据
│   └── check-resources.js          资源引用静态检查
├── build.gradle / settings.gradle / gradle.properties
└── gradle/wrapper/gradle-wrapper.properties
```
