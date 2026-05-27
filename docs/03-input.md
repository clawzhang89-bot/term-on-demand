# 输入方案设计

## 问题

终端操作需要"精确输入"，语音输入需要"内容表达"。两个需求无法被单一输入方式覆盖：

- **全尺寸键盘** → 精确、效率高，但带不出去，违背便携初衷
- **纯语音** → 便携，但"/var/log"这种路径在语音里准确率极低，更别提 Ctrl+C、Tab 补全

## 解法

**迷你硬件键 + 语音：各取所长。**

```
精确操作 → 6-8 个实体键（肌肉记忆，盲操作）
内容输入 → 语音（嘴比手快，适合描述需求）

中/英切换 → 物理键硬件级别隔离开关
             避免 speech recognition 的模式漂移问题
```

## 预期键位布局

```
┌──────────────┬──────────────┐
│   📝 中文语音  │  🇺🇸 英文语音  │  ← 按前选好语种，避免混识别
├──────────────┼──────────────┤
│  翻上（历史↑） │  翻下（历史↓） │  ← 命令历史 / less 翻页
├──────────────┼──────────────┤
│  确认（Enter） │  取消（Esc）   │
├──────────────┼──────────────┤
│  Ctrl（组合键） │  Tab（补全）   │
└──────────────┴──────────────┘
```

**Ctrl + 导航键 = 常用组合：**
- Ctrl + 翻上 = 复制
- Ctrl + 翻下 = 粘贴
- Ctrl + 确认 = Ctrl+C（中断）
- Ctrl + Tab = Ctrl+D（退出/EOF）

## 架构方案对比

> **适用场景说明：** 以下三个方案讨论的是**非 tmux 通用 App 输入场景**（浏览器地址栏、聊天 App、其他 Android App 的文本框）。
> SSH + tmux 远程终端场景下，文本注入发生在**云端**（`tmux send-keys`），不需要在 Beam Pro 端做 IME/AccessibilityService 注入。
> 详见 [`docs/06-voice-interaction-execution.md`](06-voice-interaction-execution.md)。

8BitDo 注册为蓝牙键盘后，按键事件直接走 **Android Input Framework → App**，不需要任何软件介入。

核心问题在于**语音输出的文字应该通过什么渠道送进 App**。

以下三种方案的区别在于语音→文字的注入层。

---

### 方案一：IME 注入（不推荐）

**核心思路：** 把语音输出做成一个 Android IME（`InputMethodService`），通过 IME 向当前焦点编辑器提交文字。

**架构图：**

```
8BitDo Micro                    Voice IME (InputMethodService)
  │                                   │
  │  HID key events                   │  STT → InputConnection.commitText()
  ▼                                   ▼
Android Input Framework        Android InputMethodManager
  │                                   │
  │  直接送到 App                     │  仅向焦点文本框提交
  ▼                                   ▼
Termius / 浏览器                当前焦点编辑框
```

**优点：**
- Android 原生机制，无需 AccessibilityService 权限
- 文字提交是系统级 API，稳定性高

**缺点：**
- IME 只能向**焦点文本框**提交文字 —— 终端 App 的核心交互区（虚拟终端）通常不是标准 EditText 控件
- IME 必须接管系统键盘。8BitDo 已是物理键盘，但 IME 无法选择"只处理语音、不管键盘"。两者职责冲突
- 焦点一移，语音输出中断或丢失
- 不适合终端场景，适合聊天/笔记类 App

**结论：不推荐。** 这不是技术难点问题，是**层的错配**——终端场景的"目标"不是一个文本框，而是虚拟终端 buffer。IME 无法也不应该处理这个。

---

### 方案二：AccessibilityService 注入（推荐）

**核心思路：** 不碰 IME。Voice/Key Daemon 跑 Foreground Service 做 STT + 命令路由，文本注入走 `AccessibilityService`。

**架构图：**

```
8BitDo Micro                    Voice/Key Daemon (Foreground Service)
  │                                   │
  │  HID key events                   │  BLE / Beam Pro mic → STT → 解析
  ▼                                   ▼
Android Input Framework        意图解析器（Route Engine）
  │                                 │           │
  │  直接送到 App                     │            │
  ▼                                   ▼           ▼
Termius / 浏览器             AccessibilityService   Intent
                            (文本注入: setText     (系统命令:
                             / performGlobalAction)  startActivity)
```

**优点：**
- AccessibilityService 可以操作任何 UI 元素，不限于文本框
- 8BitDo 独立走 HID，不与 Daemon 交互，零耦合
- 可以同时处理"文本注入"(语音内容)和"系统命令"(打开终端、切换 App)
- Foreground Service 长期存活，不依赖焦点

**缺点：**
- 需要声明 `BIND_ACCESSIBILITY_SERVICE` 权限
- 部分 App 的 WebView/自定义控件可能不支持 AccessibilityNodeInfo
- 注入长文本时不如 IME 的 commitText 流畅

**适用场景：** 终端操作（输入路径、执行命令、粘贴代码片段）、语音控制 App 切换。

---

### 方案三：渐进上手路径（最实用）

**核心思路：** 不一步到位。先跑起来，再逐步取代。

**架构图：**

```
              ┌─────────────────────────────────────────┐
Stage A       │  8BitDo Micro          Gboard （语音输入）│
              │  + Termius 工具栏    （Android 自带语音）  │
              │  零开发，验证核心交互                      │
              └─────────────────────────────────────────┘
                                   │
                                   ▼ 发现 Gboard 不够好
              ┌─────────────────────────────────────────┐
Stage B       │  8BitDo Micro          轻量 Voice Daemon │
              │                        (STT + 剪贴板输出) │
              │  （剪贴板辅助工具自动粘贴）                 │
              └─────────────────────────────────────────┘
                                   │
                                   ▼ 发现剪贴板不够流畅
              ┌─────────────────────────────────────────┐
Stage C       │  8BitDo Micro         完整 Voice/Key     │
              │                        Daemon +           │
              │                        AccessibilityService│
              └─────────────────────────────────────────┘
```

> **与 docs/06 的阶段区分：** 本文的 Stage A/B/C 是**通用 App 输入场景**的上手路径，而 docs/06 的 Phase 0/1/2/3/4 是 **SSH+tmux 场景**的实施路线。两者编号不同、场景不同，不要混淆。

**Stage A — 零开发验证期**
- 8BitDo Micro 到手配好键位映射
- 语音用 Gboard 自带语音输入
- Termius 上方 command bar 补充常用快捷键
- 目标：先感受"物理键 + 语音"的工作流是否真的舒服

**Stage B — 轻量 Voice Daemon**
- 写一个简版 Foreground Service 跑 STT
- 语音转文字后写入系统剪贴板
- 配合剪贴板同步工具（或手动按粘贴键）补上注入环节
- 开始积累实际使用数据和路由规则

**Stage C — 完整 AccessibilityService 版本**
- 在 Stage B 的基础上升级 Daemon + AccessibilityService
- 文本注入从"剪贴板+手动粘贴"升级为 AccessibilityService 自动注入
- 加入系统命令路由（"打开浏览器"→startActivity）

## 实现方式

### 硬件方案一：8BitDo Micro — 推荐物理按键

**目前最适合 AR 终端场景的实体按键选择：**

- **6 个物理按键 + 十字方向键**（可映射最多 10 个功能）
- 蓝牙 HID 模式 → Beam Pro 直接识别为标准键盘，无需额外驱动
- 通过官方软件（Ultimate Software）自定义每个键发出的按键信号
- 口香糖大小（约 44×44×12mm），可贴在 Beam Pro 背面或支架上
- 一周一充，续航稳定
- **注意：** 默认是手柄模式，需刷固件切换到键盘模式

**推荐键位映射：**

| 物理键 | 映射 | 功能 |
|--------|------|------|
| 上 | `↑` | 命令历史上翻 / less 翻上 |
| 下 | `↓` | 命令历史下翻 / less 翻下 |
| 左 | `Ctrl+C` | 中断当前命令 |
| 右 | `Ctrl+D` | EOF / 退出 |
| A | `Enter` | 确认 |
| B | `Esc` | 取消 |
| X | `Tab` | 自动补全 |
| Y | `Ctrl`（修饰键） | 组合：A+翻上=复制，A+翻下=粘贴 |

**价格：** ¥180 左右，性价比极高。

### 硬件方案二：Android 虚拟 Macropad（零额外硬件）

Beam Pro 自带触屏，完全可以不买实体按键。做法是在 Beam Pro 上跑一个**浮动悬浮窗**（`SYSTEM_ALERT_WINDOW` overlay），触摸按钮通过 AccessibilityService 注入键盘事件。

**实现途径：**

| 方式 | 需写代码？ | 特点 |
|------|-----------|------|
| **自写 Android 悬浮窗 APK** | ✅ 是 | 最可控、与 [Voice/Key Daemon](06-voice-interaction-execution.md) 同一套代码、推荐 |
| **Tasker + AutoInput** | ❌ 配 | 能实现，但配置量不亚于写代码 |
| **External Keyboard Helper** | ❌ 配 | 有浮窗插件功能，但自定义程度有限 |
| **Termius 自带工具栏** | ❌ 内置 | Termius 有上方快捷命令条，可放 Ctrl+C/Tab/Esc/方向键 |
| **Termux Extra Keys** | ❌ 内置 | 仅限 Termux 内，Termius 无效 |

自写一个最简单悬浮窗 APK 大约 **半天工作量**，并且可以和 Voice/Key Daemon 合并，一个 APK 搞定按键 + 语音输入。

### 硬件方案三：TourBox Elite（备选）

8 按键 + 旋钮 + 触控环，滚轮+按键手感好。但 ¥1000+ 的价格对纯终端场景功能冗余，仅在已有 TourBox 的情况下可复用。

### 为什么不推荐更大键盘？

AR 眼镜下需要**盲操**。6 个键摸两下就知道位置，全尺寸键盘需要定位、看着按，违背了"不跟终端抢键盘"的便携初衷。

AR 眼镜下需要**盲操**。6 个键摸两下就知道位置，全尺寸键盘需要定位、看着按，违背了"不跟终端抢键盘"的便携初衷。

## 语音识别策略

**中/英模式硬件切换的核心价值：**

假设你说："帮我查一下 /var/log 下面 access.log 最近 5 分钟"

- ❌ **全中文模式**："/var/log" 可能识别成 "瓦尔洛格" 或 "way尔劳格"
- ❌ **全英文模式**："帮我查一下" 可能识别成 "帮 my 茶 一下"
- ✅ **先按中文键说前半句，再按英文键说路径** → 分段清晰，识别率大幅提升

中文语音键 → 说中文描述 → 输出中文文本
英文语音键 → 说英文路径/命令 → 输出英文文本
