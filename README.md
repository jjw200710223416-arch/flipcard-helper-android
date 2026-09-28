# 翻牌助手 · Android 版

记忆翻牌小游戏的**全自动辅助 App**：录制屏幕 → 识别每个卡位 → 记住牌面 → 决策 → 模拟点击。
装到手机上就能用，**不需要电脑、不需要 root、不需要连 ADB**。

> **想要 APK？** 直接看 [BUILD.md](BUILD.md)。最省事的方式是推到 GitHub，云端自动出包给你下载，
> 电脑上什么都不用装。

---

## 关于 apk 二进制

**这个仓库里没有 `.apk` 文件，只有完整可编译的工程源码。** 这不是偷懒，是构建环境的硬约束：

开发这套代码的机器上没有 Android SDK、没有 Gradle、也没有网络（`dl.google.com` / Maven Central 全部超时），
所以**无法在那台机器上产出 apk 二进制**。能做到的是：

- ✅ 交付完整工程，`git push` 后 GitHub Actions 自动编译出 apk 供下载
- ✅ 把**算法内核用纯 JDK Java 重写，并在本地用 `javac`/`java` 真跑了回归测试**（56/56 通过，见下）
- ⚠️ Android 那一层（Activity / Service / 布局）**无法在这里编译验证**，首次构建可能需要修个别小问题

---

## 和命令行版（flipcard-helper）的关系

命令行版靠**电脑通过 ADB 遥控手机**。装成 App 以后没有电脑了，ADB 也没了。
手机上一款 App 想点另一款 App，不 root 只有一条路：

| 能力 | 电脑版（ADB） | App 版（本工程） |
|---|---|---|
| 截屏 | `adb exec-out screencap` | **MediaProjection**（录屏 API），需要一次性授权 |
| 模拟点击 | `adb shell input tap` | **AccessibilityService**（无障碍服务），需要用户在设置里手动开 |
| 读屏 | 不需要 | 同左，只靠录屏的图像，不读无障碍节点树 |

架构分层：

```
┌─────────────────────────── App ───────────────────────────┐
│  UI 层    MainActivity（单页卡片） / Calibrate / Settings / Tutorial │
├────────────────────────────────────────────────────────────┤
│  服务层   ScreenCaptureService ──┐                          │
│           （MediaProjection 抓帧）│  AutoEngine              │
│           TapAccessibilityService ┘  （后台线程跑主循环）    │
├────────────────────────────────────────────────────────────┤
│  内核层   core 包：Vision / Board / Brain / GridDetect /     │
│           Autopilot —— 纯 JDK，零 Android 依赖，可独立测试   │
└────────────────────────────────────────────────────────────┘
```

内核层刻意做成**零 Android 依赖**，这样它能在这台没有 SDK 的机器上被真正编译、真正跑测试。
这也是整个交付里唯一能"证明它对"的部分，所以做得比较扎实。

---

## 界面设计

首页就一屏，**单页卡片式**：三个状态灯 + 一个大按钮，别的一律收进「图文教程」和「参数设置」。

```
┌──────────────────────────────────┐
│  翻牌助手                         │
│  记忆翻牌 · 自动配对              │
│                                  │
│  ┌─── 运行前检查 ──────────────┐  │
│  │ ● 无障碍服务    已就绪  重开 │  │
│  │ ● 屏幕录制      已就绪  重开 │  │
│  │ ● 牌阵标定  4x4 @(66,4..) 重标│  │
│  └─────────────────────────────┘  │
│                                  │
│  ┌─────────────────────────────┐  │
│  │      开始自动配对            │  │  ← 58dp 高的大按钮
│  └─────────────────────────────┘  │
│  三步都就绪了：把游戏停在...      │
│                                  │
│  [ 图文教程 ]      [ 参数设置 ]   │
│                                  │
│  ┌─── 运行日志 ────────────────┐  │
│  │ ✓ 记忆配对：吃 #0 + #11      │  │
│  │ ? 探牌：翻 #5                │  │
│  └─────────────────────────────┘  │
└──────────────────────────────────┘
```

标定页做成**四步向导**：抓取画面 → 确认识别结果 → 手动微调 → 保存。
识别结果直接画成标注图（蓝=盖着 / 绿=翻开 / 橙=空位 / 红点=点击位置），肉眼一验就知道准不准。

**不开悬浮窗权限**：运行期间的控制放在常驻通知上（带「停止」按钮），
切到游戏后不用切回来就能停。少一个权限，少一层麻烦。

---

## 内置图文教程

`app/src/main/assets/tutorial/` 里是一个自包含的 HTML（WebView 加载，不联网），包含：

- 三步准备（开无障碍 / 授权录屏 / 标定牌阵），每步都配图
- **怎么判断标定准不准**：颜色含义表 + 正确/错位对照图 + 验收标准
- 开始运行与停止
- **日志逐行解读**：哪几行说明它聪明，哪几行说明要调参
- 排错手册（现象 → 原因 → 解决）
- 参数速查、原理简述、隐私说明

教程里的插图不是手绘的，是**用同一套渲染器程序化生成**的（`flipcard-helper/tools/make-tutorial-assets.js`），
所以它们和识别逻辑永远一致，改了代码重新生成一次就行。

---

## 验证结果

### 算法内核：56/56 通过

内核用纯 JDK Java 重写后，和 JS 参考实现做了**交叉验证**：
同一批牌局 PNG、同一个随机种子，两边分别完整打一局，翻牌次数必须逐一致。

```
一、自动识别牌阵（Java 读 JS 生成的手机画布 PNG）      16/16 ✅
二、牌面识别与判别力（Java 读 JS 生成的牌局 PNG）      30/30 ✅
三、完整对局交叉验证（JS 与 Java 翻牌次数必须一致）    10/10 ✅
```

第三条最有说服力 —— 同一 seed 下的翻牌次数：

| 牌局 | JS 参考 | Java 内核 |
|---|---|---|
| 4×4 seed1 | 26 次 | **26 次** |
| 4×4 seed3 | 24 次 | **24 次** |
| 4×4 seed5 | 22 次 | **22 次** |
| 3×4 seed11 | 18 次 | **18 次** |
| 4×5 seed11 | 28 次 | **28 次** |
| 6×6 seed11 | 56 次 | **56 次** |
| 6×6 seed12 | 58 次 | **58 次** |

翻牌次数一致，说明「识别 → 记忆 → 决策」整条链路移植后行为完全相同，没有引入任何偏差。

复现（只要有 JDK 17，不需要 Android SDK）：

```bash
mkdir -p build/core build/coretest
javac -encoding UTF-8 -d build/core app/src/main/java/com/flipcard/helper/core/*.java
javac -encoding UTF-8 -cp build/core -d build/coretest tools/coretest/*.java
java -cp build/core:build/coretest coretest.CoreSelfTest tools/coretest/fixtures
```

### 资源引用静态检查

手写 Android 工程最容易死在"id 打错、字符串漏定义"上。没有 SDK 没法编译，
所以写了个静态检查器代替：扫描全部 Java/XML 引用，和 `res/` 里的定义对账。

```bash
node tools/check-resources.js
# ✅ 没有发现未定义的资源引用（230 处引用，98 个字符串，22 个 id）
```

### Android 层的语法/类型体检

没有 SDK 也能做一部分编译检查：直接拿 `javac` 怼那 21 个 Android 源文件，
**语法错误和类型错误在"缺包"之前就会报出来**，可以据此判断代码本身有没有问题。

```bash
javac -encoding UTF-8 -Xmaxerrs 10000 -d build/android-check $(find app/src/main/java -name '*.java')
```

结果：**543 条报错，全部可溯源到缺少 Android SDK** ——

| 类别 | 数量 | 说明 |
|---|---|---|
| `package android.* does not exist` | 263 | 本机没有 SDK，预期 |
| `cannot find symbol` | 259 | 同上，符号都来自那些包 |
| `method does not override...` | 21 | `@Override` 标在 `onCreate`/`onStartCommand` 这类父类未解析的方法上 |

**关键证据**：去重后的 68 个"找不到的符号"里，**没有一个是本工程自己的类**
（`Profile`/`Board`/`Brain`/`Autopilot`/`Prefs`/`AutoEngine`/`Ui`/`Preview`…全部解析成功），
也就是说交叉引用是对的；同时**语法错误 0 条、类型错误 0 条**。

### 没能验证的部分（说清楚）

- Android 层**没有真正编译过**（没有 SDK），只能证明语法/类型/交叉引用/资源引用是对的，
  无法保证和真实 Android API 的签名完全吻合
- MediaProjection 抓帧、无障碍手势在你的具体机型上的实际行为：**没有真机验证**
- 不同厂商 ROM 对模拟点击的限制（小米「USB 调试（安全设置）」、OPPO「禁止权限监控」等）：文档里写了，但没实测

---

## 权限与隐私

| 权限 | 用途 |
|---|---|
| `FOREGROUND_SERVICE` + `FOREGROUND_SERVICE_MEDIA_PROJECTION` | 录屏服务常驻，Android 14 起强制要求 |
| `POST_NOTIFICATIONS` | 显示运行状态和「停止」按钮 |
| `WAKE_LOCK` | 避免运行中断 |
| 无障碍服务（不是普通权限，需手动开启） | 执行点击手势 |

- **没有 `INTERNET` 权限**，App 无法联网，不存在上传数据的可能
- 录屏画面只存在内存里当场识别，不落盘、不导出
- 无障碍服务不读屏幕内容（`canRetrieveWindowContent="false"`），只用点击手势能力

**使用边界**：这类工具属于外部自动化操作，可能违反小游戏的用户协议，也可能被判定为异常操作。
请只在你自己做的游戏、单机游戏或已获授权的场景里使用。

---

## 已知局限

1. 屏幕录制授权**每次开启 App 都要重新点一次**（Android 系统强制，14 起尤其严格）
2. 有 `FLAG_SECURE` 的界面截不到（银行类 App 常见，游戏一般没有）
3. 部分厂商 ROM 默认禁止模拟点击，需要额外开关（教程里列了各品牌的做法）
4. 牌阵位置随关卡变化的游戏，需要每关重新标定（可以存不同档案，但 App 版目前只存一份）
5. 抓帧节流固定 25fps；手机很卡的话要到设置里把 `pollMs` 调大

---

## 目录结构

```
flipcard-android/
├── README.md                       本文件
├── BUILD.md                        ★ 怎么出 apk（三种方式，含 GitHub Actions 图文步骤）
├── .github/workflows/build.yml     云端构建流水线
├── app/
│   ├── build.gradle
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── java/com/flipcard/helper/
│       │   ├── App.java
│       │   ├── core/               ★ 纯 JDK 算法内核（可独立编译测试）
│       │   │   ├── RgbaImage.java  DRect.java  Rect.java
│       │   │   ├── Vision.java     灰度化 / 重采样 / 16x16 描述子 / 距离度量
│       │   │   ├── Profile.java    网格几何 + 阈值 + 时序
│       │   │   ├── Board.java      卡位状态判定 / 卡背模板学习 / 网格自对位
│       │   │   ├── Brain.java      记忆库 + 四条决策规则
│       │   │   ├── GridDetect.java 自动识别牌阵
│       │   │   └── Autopilot.java  主循环（通过 Device 接口与平台解耦）
│       │   ├── data/Prefs.java
│       │   ├── service/
│       │   │   ├── ScreenCaptureService.java    MediaProjection 抓帧前台服务
│       │   │   ├── TapAccessibilityService.java 无障碍手势
│       │   │   ├── AutoEngine.java              后台主循环
│       │   │   └── Calibration.java             标定中间结果
│       │   └── ui/
│       │       ├── MainActivity.java      单页卡片式主页
│       │       ├── CalibrateActivity.java 四步标定向导
│       │       ├── SettingsActivity.java  参数设置（代码生成，免一堆 XML）
│       │       ├── TutorialActivity.java  内置图文教程
│       │       ├── Preview.java           标注图渲染
│       │       └── Ui.java                界面小工具
│       ├── res/                    布局 / 主题 / 字符串 / 图标
│       └── assets/tutorial/        内置图文教程（HTML + 程序化生成的插图）
└── tools/
    ├── coretest/                   Java 内核回归测试 + 测试数据（约 0.5MB）
    └── check-resources.js          资源引用静态检查器
```

---

## 核心算法（一句话版）

记忆翻牌**不需要认出图案是什么**，只需要判断"这两张是不是同一张"。
所以把每个卡位裁出来压成 16×16 灰度 + 平均色，用三个判据（灰度差 / 余弦距离 / 颜色距离）
同时成立来判定同图案——没有任何模型、没有图标库、没有联网。

决策四条规则：**免费配对** → **信息优先**（只翻没见过的牌）→ **顺手补刀**（翻开后若搭档已见过就补上）
→ **终局排除**。探牌时两张牌同时亮着的那一帧会把两张都读下来，这是省步数的关键细节。
