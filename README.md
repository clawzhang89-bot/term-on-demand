# term-on-demand 默认终端，按需 UI

> 一个务实的"终端优先、按需可视化"远程开发工作流。

## 核心理念

- **95% 的时间：终端（SSH）** — 纯文本协议，延迟最低，效率最高，零系统开销
- **5% 的时间：按需 HTML** — 需要看东西时，AI agent 现场写一个 HTML，Nginx serve 给你的浏览器
- **输入：迷你键盘 + 语音** — 精确操作靠肌肉记忆，内容输入靠语音，中/英模式硬件切换

## 架构总览

```
你（在空中 / 咖啡馆 / 公园）
  │
  ├─ 迷你蓝牙键盘（Ctrl/C/V/Tab/翻页/中断） → 精确操作
  ├─ 语音（中/英模式硬件切换键） → 内容输入
  │
  └─ XREAL One Pro + Beam Pro
       │
       ├─ Termius SSH → 云端 Ubuntu → 终端交互
       └─ Chrome → Nginx → AI 现场写的 HTML 页面
```

## 核心组成

| 组件 | 选型 | 说明 |
|------|------|------|
| 眼镜 | **XREAL One Pro** | 全彩 1080p 投屏，57° FOV，比 Rokid Glasses 更适合看终端 |
| 计算终端 | **Beam Pro** | 独立的安卓设备，跑 SSH + 浏览器 + 语音输入，不依赖手机 |
| 云端 | **Ubuntu + Nginx** | 终端操作的主机，Nginx 做临时 HTML 投递 |
| AI agent | **Claude Code / Codex CLI / Cloud Code** | 按需写 HTML 页面，serve 到 Nginx |
| 输入 | **迷你键盘 + 语音** | 6-8 键 macropad + 中/英语音模式硬件切换 |

## 为什么不是纯语音

终端操作（路径补全、参数调整、Ctrl 组合键）本质上是**精确交互**，语音不适合。但内容描述（"帮我查一下这个日志"）语音更快。所以方案是：**按物理键做精确操作，按语音键做内容输入，中/英模式硬件级别切换以避免识别歧义**。

## 为什么不是纯投屏终端

终端适合看文本，不适合看图表/图片/结构化数据。所以方案是：**默认终端干活，需要可视化时让 AI 现场写 HTML**——比固定 UI 面板更灵活、零后台开销、用完即焚。

## 快速开始

```bash
# 1. SSH 到你的 Ubuntu 服务器
ssh user@your-server

# 2. 生成一个系统信息页面
curl -s https://raw.githubusercontent.com/term-on-demand/main/scripts/sysinfo | bash

# 3. 在浏览器（眼镜里）打开
# http://your-server:port/sysinfo.html

# 4. 用完了关掉，回到终端
```

## 文档

- [核心理念](docs/01-philosophy.md)
- [硬件选型分析](docs/02-hardware.md)
- [输入方案设计](docs/03-input.md)
- [完整工作流](docs/04-workflow.md)
- [架构总图](docs/architecture.md)
- [AI Prompt 示例](ai/prompt-samples.md)
- [预制 Skill 脚本](scripts/README.md)

## License

MIT
