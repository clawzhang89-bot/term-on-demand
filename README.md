<p align="center">
  <img src="https://img.shields.io/badge/status-active-brightgreen" alt="Status">
  <img src="https://img.shields.io/badge/license-MIT-blue" alt="License">
</p>

<h1 align="center">⌨️ term-on-demand</h1>
<h3 align="center">默认终端，按需 UI</h3>
<p align="center"><i>Terminal-first, on-demand visualization for remote development with AR glasses</i></p>

---

## 一句话

**不跟终端抢键盘，不跟浏览器比渲染。先让终端做它擅长的事，需要看东西时让 AI 现场写一个 HTML。**

---

## 问题

远程开发时，你永远在两个极端之间摇摆：

| 方式 | 优点 | 缺点 |
|------|------|------|
| 🔧 **纯终端 SSH** | 延迟最低、效率最高、零开销、可脚本化 | 不适合看图、图表、结构化比较 |
| 🖥 **图形面板 (Cockpit/Webmin/Portainer)** | 可视化好、交互丰富 | 常驻进程、安全面多、功能固定、为 5% 场景牺牲 95% 资源 |

传统的选择是：要么强行在终端里看 ASCII 表格，要么装一个沉重的面板。**为什么不两个都选，但各管各的？**

---

## 解法

```
默认状态 ──── 终端 (SSH) ──── 95% 的时间在这里
  │                             效率最高，延迟最低，零系统负担
  │
  └── 需要看东西时 ── 一句话让 AI 写一个 HTML
                     │
                     ├── Nginx 直接 serve
                     ├── 浏览器打开看一眼
                     └── 关掉，回到终端
```

- **95% 的时间**：终端交互，纯文本协议，延迟最低，效率最高
- **5% 的时间**：需要可视化时，AI agent 现场写一个 HTML → Nginx serve → 看一眼，关掉
- **零常驻进程**：没有面板在后台吃内存，没有额外的攻击面
- **按需定制**：每张页面都是按你那一刻的需求写的，不是从固定菜单里挑
- **用完即焚**：看完关掉，系统状态回到零

---

## 架构

```
你 (在路上 / 咖啡馆 / 公园)
  │
  ├── 迷你蓝牙键盘 (6-8键)     ← 精确操作: Ctrl/C/V/Tab/Esc/翻页
  ├── 语音 (中/英硬件切换)      ← 内容输入: 说路径、描述需求
  │
  └── XREAL One Pro + Beam Pro
       │
       ├── Termius SSH ──→ 云端 Ubuntu ──→ 终端交互
       │                                        │
       │                                   AI Agent
       │                                   (Claude Code / Codex CLI)
       │                                        │
       └── Chrome ──────────→ Nginx ───────── 写 HTML
                                │
                         浏览器打开看两眼，关掉
```

---

## 硬件选型

| 角色 | 推荐 | 替代方案 | 理由 |
|------|------|---------|------|
| 眼镜 | **XREAL One Pro** (¥4299) | XREAL One (¥2999) | 全彩 1080p，57° FOV，3DoF 屏幕悬停，SDK 生态完善 |
| 计算终端 | **Beam Pro** (¥1299-2999) | 三星手机 DeX | 双 USB-C，独立安卓设备，不占手机 |
| 云端 | **已有 Ubuntu + Nginx** | — | 终端操作主机，Nginx 做临时 HTML 投递 |
| AI agent | **Claude Code / Codex CLI** | Cloud Code | 按需生成 HTML 的引擎 |
| 输入 | **迷你 Macropad + 语音** | TourBox Elite | 6-8 键精确控制，语音做内容输入 |

> 为什么不是 Rokid Glasses？Rokid Glasses 是单色 micro-LED，只适合显示几行提示文字，**无法显示完整的终端或浏览器**。Rokid Max 2 虽然全彩，但没有开发者 SDK，也没有空间悬停能力。

---

## 输入方案

```
┌──────────────┬──────────────┐
│   📝 中文语音  │  🇺🇸 英文语音  │  ← 语音模式硬件切换
├──────────────┼──────────────┤     (根除中英混识别的歧义)
│  翻上 (历史↑) │  翻下 (历史↓) │  ← 命令历史 / 翻页
├──────────────┼──────────────┤
│  确认 (Enter) │  取消 (Esc)   │
├──────────────┼──────────────┤
│  Ctrl (组合键) │  Tab (补全)   │  ← Ctrl+翻上=复制, Ctrl+翻下=粘贴
└──────────────┴──────────────┘
```

**为何这样设计：** 终端操作本质上是精确交互（路径补全、参数调整、Ctrl 组合），语音不适合。中英文混说时语音识别最大的坑不是单条不准，而是**切换瞬间的模式漂移**——"/var/log"在中文模式下变成"瓦尔洛格"，"帮我查一下"在英文模式下变成"帮 my 茶 一下"。物理键硬切换是唯一的靠谱解法。

---

## 预制脚本 (开箱即用)

```bash
# 系统状态仪表盘
sysinfo
# → 眼镜浏览器打开 http://your-server/sysinfo.html

# 目录文件浏览器
ls-html /var/log
# → 可折叠文件树，带大小、修改时间、搜索

# 日志查看器
log-view /var/log/nginx/access.log 200
# → 最后一页日志，关键词高亮，错误行标记，搜索
```

**预制 vs AI 生成的分层策略：**

| 层 | 覆盖场景 | 延迟 | 维护 |
|---|---------|------|------|
| 1️⃣ 预制脚本 | 80% 的"看一眼"需求 (sysinfo/ls/log) | ⚡ 零 | 写一次 |
| 2️⃣ 半预制模板 | 15% (表格渲染、图表生成、目录树) | ⚡ 低 | 改参数 |
| 3️⃣ AI 现场写 | 5% (独特需求) | ⏳ 几秒 | 用完就删，好用就收进预制库 |

---

## 完整工作流示例

### 日常调服务器

```
你在咖啡厅坐下
  → Beam Pro 打开 Termius，SSH 到云端 Ubuntu
  → 眼镜里看到终端，迷你键盘敲命令
  → "查一下为什么数据库连接超时"
  → 语音键 → 终端自动键入: grep timeout /var/log/app/error.log
  → 终端返回结果
```

### 需要可视化时

```
"画个过去一小时的连接数趋势"
  → AI agent 采集数据 → 写一个带 Chart.js 的 HTML
  → 放到 Nginx 目录
  → 眼镜从终端切换到浏览器
  → 看到完整的折线图，带时间轴和峰值标注
  → 看完关掉 tab，回到终端
```

### 需要 AI agent 代操作时

```
"帮我看看生产环境有什么异常"
  → OpenClaw SSH 到你的 Ubuntu
  → 执行 kubectl get events --sort-by='.lastTimestamp'
  → 返回结果给你
  → 如果方便可视化 → 生成 HTML 页面
```

---

## 相关资源

| 文档 | 说明 |
|------|------|
| [📖 核心理念](docs/01-philosophy.md) | "默认终端，按需UI" 的详细论述 |
| [🔧 硬件选型](docs/02-hardware.md) | XREAL vs Rokid 完整对比 |
| [⌨️ 输入方案](docs/03-input.md) | 迷你键盘 + 语音输入设计 |
| [🔄 完整工作流](docs/04-workflow.md) | 各种场景的操作流程 |
| [🏗 架构总图](docs/architecture.md) | 系统架构和数据流向 |
| [🤖 AI Prompt 示例](ai/prompt-samples.md) | 让 AI 写 HTML 的 prompt 模板 |
| [📜 预制脚本说明](scripts/README.md) | 内置脚本的使用方式 |
| [🗺 路线图](docs/05-roadmap.md) | 待办和未来方向 |

---

## 灵感来源

- [Hold The Robot: Linux on Android with AR Glasses](https://holdtherobot.com/blog/2025/05/11/linux-on-android-with-ar-glasses) — $636 搭建随身 Linux 工作站
- [Peter Denham: Working with XReal Glasses](https://denham.ie/posts/2024-09-05-working-with-xreal-glasses-as-a-software-developer/) — 每天用 XREAL 编程的全职开发者
- [cc-g2](https://zenn.dev/wmoto_ai/articles/claude-code-even-g2-glasses) — 智能眼镜 + Claude Code / Codex CLI 的交互方案
- [Project Aura](https://www.xreal.com/cn/blog/project-aura-google-io-2026) — XREAL + Google 的 Android XR 眼镜

---

<p align="center">Built for developers who value terminal. 🛠️</p>
