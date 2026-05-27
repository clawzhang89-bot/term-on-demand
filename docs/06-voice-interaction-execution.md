# 06. 语音交互执行形态

## 0. 这份文档解答什么

承接 [`docs/03-input.md`](03-input.md) 和 [`docs/04-workflow.md`](04-workflow.md):

> 在 AR 眼镜 + Beam Pro + 蓝牙小键盘 + 中英语音 这套硬件约束下,
> **语音 → 命令 → 终端执行** 这条路径具体怎么实施。

也就是把"按一下中文键说话,terminal 里就出现命令"这件事拆到组件、接口、延迟、失败模式的颗粒度,目标是任何工程师拿到这份文档就能开始动手。

本文给出**两套方案**:
- [§1 当下主推方案:剪贴板桥接 + Claude Code](#1-当下主推方案剪贴板桥接--claude-code) — 工程量最小、复用现成 agent、推荐先实施
- [§2 备选方案:Voice Gateway 双端架构](#2-备选方案voice-gateway-双端架构保留) — 完整自研,在主推方案不适用时考虑

读这份文档的顺序建议:**§0.5 → §1 → 决定按 §1 实施 → §2 留作未来 reference**。

---

## 0.5 主推 vs 备选 — 当下为什么选简化方案

| 维度 | §1 主推(剪贴板 + Claude Code) | §2 备选(Voice Gateway) |
|---|---|---|
| 新增组件 | 1 个 Beam Pro Service(录音 + 豆包 + 剪贴板) | 1 Beam Pro Service + 1 云端 Gateway + WebSocket |
| 文本注入位置 | Beam Pro 端(overlay 预览 + Enter 触发剪贴板粘贴) | 云端(`tmux send-keys`) |
| LLM 意图理解 | Claude Code 本身(它本来就是 agent) | Voice Gateway 拼 prompt 调 Claude API |
| 候选展示 | Claude Code plan 模式(主建议 + alternative) | overlay 显示 top-3,Tab 切换 |
| 工程量(到 MVP) | ~1-2 周 | ~3-6 周 |
| 依赖 Claude Code | 是 | 否(可换 Codex / 本地 LLM) |
| 多端共享 voice 服务 | 否(每个 Beam Pro 独立) | 是(同一 Gateway 多客户端) |
| 音频可不上云 | 否(豆包要上云) | 是(本地 faster-whisper) |
| 依赖 tmux | 否(任意 shell) | 是 |

**主推选择的根因**:Claude Code 本身已经是成熟的 agent —— 它做意图理解、工具调用、context 管理、plan/confirm 流程,几乎覆盖了 Voice Gateway 设计要做的 90%。补上"接受语音输入"这 10%(剪贴板桥接),剩下的让 Claude Code 接管,工程量直接砍 2-3 倍。

**备选保留的原因**:在"不用 Claude Code / 要本地 ASR / 多设备共享 / 工程团队要细粒度可控中间层"这些场景下,Voice Gateway 仍是合理设计 —— 见 [§2.0 何时重新评估](#20-何时重新评估)。

---

# § 1. 当下主推方案:剪贴板桥接 + Claude Code

## 1.1 一句话拓扑

```
[按住 F13 中文 / F14 英文] → Beam Pro overlay 立即激活 → 录音 → 松开
                                                            │
                                                            ▼ 豆包 ASR(境内)
                                                Beam Pro overlay 显示 ASR 文本(预览)
                                                            │
                              ┌─────────────────────────────┼─────────────────────────────┐
                              ↓                             ↓                             ↓
                          [按 Enter 确认]                [按 Esc 撤销]              [再按 F13/F14 重说]
                              │                             │                             │
                       写剪贴板 + 触发粘贴              关闭 overlay               关闭旧 overlay,
                       到 Termius(机制见 §1.3.1.1)     (不发送任何文本)            立即激活新 overlay
                       + 关闭 overlay                                              并重新录音
                              │
                       SSH → 海外服务器
                              │
                       Claude Code(plan 模式)
                              │
                       看建议 → Enter 确认 / 编辑 → 执行 → 输出回流 → Termius 显示
```

整套系统**只有一个新组件**:Beam Pro 上的 Voice Daemon(按键 → 录音 → 豆包 → overlay 预览 → 写剪贴板 + 模拟粘贴)。其他全部复用现成的(8BitDo / Termius / Claude Code / 你的 SSH server)。

**两层防误**:
- **第一层(Beam Pro 端)**:ASR 文本对不对 → overlay 上一眼看清楚 → Esc 撤销,Termius 永远不会被污染
- **第二层(服务器端)**:意图理解对不对 → Claude Code plan 给出建议 → Ctrl+C 拒绝

## 1.2 数据流详解(一次完整交互)

```
T+0ms        用户按下 F13(中文)
T+30ms       Voice Daemon 收到 HID KeyDown
             → 立即创建/重建 SYSTEM_ALERT_WINDOW overlay,显示"🎤 录音中..."
             → AudioRecord.start()
T+30..3030ms 用户说"看下 nginx 最近的错误"
T+3030ms     用户松开 F13
T+3050ms     AudioRecord.stop() → overlay 显示"识别中..."
T+3060ms     音频(Opus ~10KB)→ POST 豆包 ASR(带[动态热词表](03-input.md#热词持续优化))
T+3060..3550ms 豆包 ASR(端到端 ~500ms)
T+3550ms     豆包返回 text: "看下 nginx 最近的错误"
T+3580ms     overlay 切换显示:┌──────────────────────────────────┐
                              │ 🎤 看下 nginx 最近的错误            │
                              │                                  │
                              │  Enter 发送   Esc 撤销   F13 重说  │
                              └──────────────────────────────────┘

[分支 A]:用户按 Enter 确认
T+N          Enter KeyDown(被 overlay 焦点拦截,不落到 Termius)
T+N+10ms     Voice Daemon 写 ClipboardManager.setPrimaryClip(text)
T+N+30ms     触发粘贴到 Termius(具体机制见 §1.3.1.1,Stage A 必须验)
T+N+50ms     overlay 关闭 → 焦点还给 Termius
T+N+100ms    Termius 收到字符 → 把字符通过 SSH 流推到服务器
T+N+200ms    Claude Code 在 server stdin 收到这段文本
T+N+200ms..  Claude Code 思考 + 调 bash 工具
T+N+3-6s     Claude Code plan:"我打算运行 `tail -100 /var/log/nginx/error.log`"
T+N+P        用户按 Enter 确认 → 执行 → 输出回流

[分支 B]:用户按 Esc 撤销
T+N          overlay 关闭,什么都不发送
             → Termius 保持原状,Claude Code 完全没看到这次失败的 ASR

[分支 C]:用户按 F13 重说
T+N          立即关闭旧 overlay
T+N+10ms     创建新 overlay,显示"🎤 录音中..."
T+N+10ms     AudioRecord.start()
             ...回到 T+30 流程重新走一遍
```

**整体时延**:
- 从松开 PTT 到 overlay 显示预览:**~500ms**(豆包 ASR)
- 从 Enter 确认到 Claude Code plan 出来:**~3-6s**(SSH 传输 + Claude Code 思考)
- 总:**4-7s**

比 §2 的 top-3 方案略长,但**多了一个 0 延迟的 ASR 撤销机制** — 误识别的代价从"污染 Claude Code session 要解释/撤销"降为"按一下 Esc 当无事发生"。

## 1.3 组件分解

### 1.3.1 Voice Daemon(Beam Pro 上)

唯一需要新写的组件。

| 项 | 选择 |
|---|---|
| 语言 | Kotlin / React Native(Stage A 可 Tasker + 脚本) |
| 形态 | Android Foreground Service + 通知 |
| 权限 | `RECORD_AUDIO`, `BLUETOOTH_CONNECT`, `SYSTEM_ALERT_WINDOW`(overlay) |
| **不需要** | Accessibility Service、root |

职责(按状态机组织):

```
状态: IDLE
  ↓ F13/F14 KeyDown
状态: RECORDING(overlay 显示"🎤 录音中...")
  ↓ F13/F14 KeyUp
状态: ASR_PENDING(overlay 显示"识别中...")
  ↓ 豆包返回 text
状态: PREVIEWING(overlay 显示 text + 三个操作提示)
  ├─ Enter → COMMITTING(写剪贴板 + 触发粘贴到 Termius,机制见 §1.3.1.1)→ overlay 关闭 → 回 IDLE
  ├─ Esc → overlay 关闭 → 回 IDLE(不发送任何文本)
  └─ F13/F14 → 关 overlay → 回 RECORDING(重说)
```

**关键设计**:Enter/Esc 复用 Termius 标准键,Voice Daemon 只在 overlay 显示时通过焦点机制拦截(`SYSTEM_ALERT_WINDOW` focusable=true)。overlay 关闭后焦点立刻还给 Termius,Enter/Esc 恢复正常 Termius 语义。这样**新增的物理键只有 F13/F14 两个**。

核心操作:
- **HID 按键监听**:`InputDevice` 拿 8BitDo 的 F13/F14(全局监听);Enter/Esc 通过 overlay focusable 拦截
- **PTT 录音**:`AudioRecord` 16kHz mono PCM,Opus 实时编码
- **豆包 ASR 调用**:HTTPS POST,带[动态热词表](03-input.md#热词持续优化)
- **Overlay 渲染**:`SYSTEM_ALERT_WINDOW` 浮在前台 App 上方,简单文本框
- **剪贴板 + 模拟粘贴**:`ClipboardManager.setPrimaryClip(...)` + 用 `Instrumentation.sendKeyDownUpSync` 或 dispatchEvent 发 Ctrl+Shift+V 给前台 App
- **错误处理**:豆包失败 → overlay 显示错误信息 + 仍允许 Esc / F13 重说

### 1.3.1.1 "Enter 一键发送"的技术约束 ⚠️ 最大未知项

主推方案的简化目标是 **overlay 显示时 Enter = 确认 + 发送 + 关 overlay**,不引入额外按键。但这要求 Voice Daemon 在按 Enter 时完成:

1. 写剪贴板 — ✓ 标准 ClipboardManager 即可
2. 把文本"塞进"Termius — ⚠️ 这一步在 Android 安全模型下需要权限
3. 关闭 overlay 并把焦点还给 Termius — ✓ 标准 WindowManager 即可

第 2 步是关键卡点。`SYSTEM_ALERT_WINDOW` 允许浮窗 + 抢键,**但不允许跨 App 注入按键事件**(无法模拟 Ctrl+Shift+V 给 Termius)。可行路径列举:

| 路径 | 权限要求 | "Enter 一键"能否实现 | 备注 |
|---|---|---|---|
| Voice Daemon 内部模拟 Ctrl+Shift+V(`Instrumentation`)| 无 | ❌ | 只能作用于 Voice Daemon 自己,Termius 收不到 |
| **AccessibilityService.performAction(ACTION_PASTE 或 setText)** | Accessibility | ✅ | Termius 输入区不一定支持 ACTION_PASTE,需实测 |
| **Termius 接收 Intent 推送文本** | 无 | ✅ | 取决于 Termius 是否暴露此 Intent;**Termux 支持**(`am start -a android.intent.action.SEND -t text/plain --es Intent.EXTRA_TEXT "..."`)— Termius 待验 |
| **Shizuku** + `input keyevent` | 用户激活 Shizuku(一次性 ADB) | ✅ | 零运行时权限,但需要 Shizuku 框架已安装并激活 |
| **8BitDo 把 Enter 物理键直接配成 Ctrl+Shift+V** | 无 | ⚠️ | Voice Daemon 拦不到 Enter 这个语义,只能用别的键(或物理键发组合序列)做 overlay 确认。或者:Enter 物理键由 Voice Daemon 监听抢键 + 不传给 Termius,接受后 Voice Daemon 用上面任一机制再触发粘贴 |

**Stage A 必须按优先级实测**:
1. 先验 **Termius Intent 接收文本** — 如果可行,零权限拿下,最干净
2. 其次验 **AccessibilityService.ACTION_PASTE on Termius** — 牺牲一个权限拿稳定
3. 备选 **Shizuku** — 用户能接受 ADB 激活时

**如果 1/2/3 都不通**,有两条退路:
- 回到 docs/03 §架构方案二的 AccessibilityService 完整路径,接受权限代价
- **降级 Enter 语义**:overlay 显示时按 Enter 只做"关 overlay + 写剪贴板",用户再按一次物理粘贴键(违背"Enter 一键"的简化精神,但保证可用)

这是简化方案的**最大单点不确定性**,Stage A 必须解决,否则全局架构要调整。

### 1.3.2 8BitDo Micro(物理按键)

按 [`docs/03-input.md`](03-input.md) 推荐的键位映射,**语音交互只新增两个键**(F13、F14),Enter/Esc 复用 Termius 的标准键:

| 物理键 | 键码 | overlay 关闭时(普通终端)| overlay 显示时(语音交互)|
|---|---|---|---|
| 语音键 1 | F13 | (空,无功能)| **"录(再录)中文"** — 关旧 overlay,立即开新 overlay 进入 PTT |
| 语音键 2 | F14 | (空,无功能)| **"录(再录)英文"** — 同上,语种 = en |
| Enter | `Enter` | Termius 普通 Enter(命令执行 / Claude Code plan 确认)| **确认提交** — 见 §1.3.1.1 的实际机制 |
| Esc | `Esc` | Termius 普通 Esc | **撤销** — 关 overlay,不发送 |
| 其他终端键 | Ctrl/Tab/↑↓/Ctrl+Shift+V ... | 见 docs/03 键位表 | (Voice Daemon overlay 模态下不响应)|

**核心简化**:语音相关的新键只有 F13、F14;Enter/Esc 复用 Termius 已有的键,Voice Daemon 通过 overlay 的 modal 状态决定是否拦截。这样:
- overlay 不显示 = Beam Pro 一切如常,Termius 正常工作
- overlay 显示 = Voice Daemon 接管 Enter/Esc,处理完后释放回 Termius

**关键依赖(Stage A 必须验)**:
- 8BitDo Micro 的 Ultimate Software 是否支持发 F13 / F14 键码(普通 App 不占用、Android 能识别)— 如果不能,退到 Ctrl+Alt+1 / Ctrl+Alt+2 组合键
- Voice Daemon 的 SYSTEM_ALERT_WINDOW overlay 设 focusable=true 时能否抢键 + 关闭后焦点正确还给 Termius
- Termius 是否对 Voice Daemon 关 overlay 后的瞬间"接收剪贴板"有兼容性问题(可能需要短暂 sleep)

### 1.3.3 Termius(SSH client,零改造)

- 启用 SSH `RemoteCommand` 或在 `~/.zshrc` 末尾自动启动 Claude Code:
  ```
  tmux new -A -s dev "claude code --resume"
  ```
- Termius 工具栏放好 Esc / Ctrl+C / Tab 等高频键

### 1.3.4 Claude Code(服务器端,零开发)

启动配置:

```bash
# ~/.config/claude/config.toml  (或 claude code 实际配置文件位置)
# 建议配置:
permission_mode = "plan"        # 默认 plan,看到建议再执行
auto_compact = true             # 长会话自动压缩 context
```

或启动时:
```bash
claude code --permission-mode plan --resume
```

`plan` 模式让 Claude Code "先说我打算做什么,等用户 Enter 确认再执行" — 这是用户安全感的关键。

## 1.4 与 docs/03 的关系

**§1 主推方案 = docs/03 方案二/三(本地注入 + 渐进上手)+ Claude Code 作为意图理解层**。

具体对应:
- docs/03 §硬件方案一(8BitDo Micro)→ 本文 §1.3.2
- docs/03 §架构方案二(本地注入)的"剪贴板路径" → 本文核心机制
- docs/03 §架构方案三(渐进路径)的 Phase 0/1 → 本文 [Stage A/B](#15-实施-stage-a--b--c) 的简化形式
- docs/03 §热词持续优化 → 本文 §1.3.1 直接复用

**不冲突,不重复设计** — docs/03 讨论的是"按键 + ASR + 本地注入"这一组通用机制,本文是"把这组机制 + Claude Code 组合起来支撑 SSH 终端开发场景"的具体落地。

PR #6 写在原 §0.5 里"docs/06 = tmux send-keys 不绕开本地注入,docs/03 = 本地注入"的边界,在新主推方案下已**不再准确** — 主推方案恰好就走本地注入路径。新边界:
- 主推方案 §1 ≈ docs/03 的本地注入 + Claude Code(命中本项目核心场景)
- 备选方案 §2 ≈ 原 docs/06 设计(tmux send-keys 路径,在不用 Claude Code 等场景下保留价值)

## 1.5 实施 Stage A / B / C

(用 Stage A/B/C 避免与 docs/03 Phase 0/1/2 或 §2 备选方案的 Phase 0-4 编号撞车)

### Stage A:可行性硬验证(1 周,纯手动 + 1 个最小脚本)

**不写完整 app**,先验证两个核心假设:
1. **8BitDo Micro 真的能配出 PTT 键** — Ultimate Software 给"按住时持续发某键码,松开停"的能力
2. **豆包 ASR 对你的常用中英混说指令准确率够用** — 准确率门槛 ≥ 70% 才进 Stage B

具体:

```bash
# A1: 录 30 句日常远程开发指令(自然说,中英混)
# A2: 调豆包 ASR API,得到 30 条 text
# A3: 把每条 text 当作 prompt 喂给 server 上的 Claude Code(--permission-mode plan)
# A4: 人工标注:Claude Code 给出的 plan 是不是"对的命令"
# A5: 计算可执行率
```

`Voice Daemon` 这阶段不存在 — 你手动 curl + paste 验证整条 pipeline 是否工作。门槛过了才值得投入开发。

### Stage B:MVP(1-2 周)

写 Voice Daemon 最小可用版:
- Tasker + AutoTools 调录音 + curl 豆包 + 写剪贴板(零原生代码,1-2 天)
- 或 Kotlin Foreground Service + AudioRecord + Retrofit 调豆包(更正经,1-2 周)
- 不做 overlay,识别结果通过 Android Notification 显示

跑通端到端"按键 → 说话 → 粘贴 → Claude Code plan → 确认 → 执行"。

### Stage C:体验优化(持续)

- 加 overlay visual confirm("已识别:XXX")
- 加[动态热词表](03-input.md#热词持续优化)
- Voice Daemon 自启动 + 后台稳定性优化
- 多设备同步剪贴板(可选,laptop + Beam Pro 共享 ASR 服务)

## 1.6 失败模式与降级

| 故障 | 检测 | 用户体感 | 恢复 |
|---|---|---|---|
| 豆包 ASR 识别错(高频)| overlay 显示明显不对的文本 | 一眼看清 | **按 Esc 撤销** — Termius 永远不会被污染,这是 overlay 设计的核心价值 |
| 豆包 API 失败 / 限流 | HTTP 非 200 / 超时 | overlay 显示"ASR 失败" | 自动重试 1 次;再失败按 Esc / F13 重说 |
| 网络断 | 上传超时 | overlay 显示"网络断" | overlay 不关,网络恢复可按 F13 重录 |
| 8BitDo 蓝牙断 | Daemon 收不到 HID 事件 | 按键无响应 | Notification "蓝牙断开" + Daemon 后台重连 |
| Claude Code 进程死 | SSH 上敲 Enter 无响应 | shell prompt | 手动 `claude code --resume`,session 不丢 |
| 注入到 Termius 失败 | Enter 后 Termius 无字符进入 | 看到 overlay 关了但 terminal 没动 | 内容仍在系统剪贴板,用户手动 Ctrl+Shift+V 兜底(见 §1.3.1.1 降级) |
| 整套语音崩 | — | 都不响应 | 退回 Termius 直接打字 + Claude Code 文本对话,完全可用 |

**核心降级承诺**:Voice Daemon / 豆包 / 8BitDo 任何一个挂了,**Termius + Claude Code 继续可用** — 你打字跟 Claude Code 对话是 fallback 路径。

## 1.7 关键技术选择

### 1.7.1 ASR:豆包(火山引擎 ASR)

为什么:
- 中英混说能力强(国内 ASR 厂商在中英混领域已经做得不错)
- 支持热词偏置(配合 docs/03 §热词持续优化)
- 境内厂商境内调用,延迟低(Beam Pro 在国内,豆包也在国内,一跳)
- 价格便宜(0.0015 元/秒,1 小时音频 ~5 元)

trade-off:
- 音频必上云(豆包是云服务,无本地版本)
- 锁定豆包 API 协议(切换 provider 要改 Daemon)

如果隐私是硬需求,见 [§2 备选方案](#2-备选方案voice-gateway-双端架构保留) 的本地 ASR 选项。

### 1.7.2 LLM 在哪调

**不需要单独调** — Claude Code 自己调,Voice Daemon 完全不碰 LLM。

Beam Pro 端只做"录音 + ASR + 写剪贴板",所有意图理解和命令翻译由服务器侧的 Claude Code 接管。这意味着:

- Anthropic API key 只在服务器上(Beam Pro 不存)
- LLM 出网走 VPS 国际出口(比 Beam Pro 4G 出海稳)
- Claude Code 的 session 持续性、context 管理、工具调用等能力都自动复用

### 1.7.3 中英语种切换

按 docs/03 现有方案 — 两个 PTT 键分别对应中文 / 英文 ASR 模式,通过 Voice Daemon 给豆包 API 传不同 `language` 参数。

实际重要性:在豆包 ASR 中英混说已不错的情况下,硬切换的边际价值约 5-10%。键位代价低,保留无损。

### 1.7.4 服务器位置

推荐**海外服务器 + mosh**,理由见与本设计配套的部署考量:

- SSH 字符回显在海外链路下用 mosh(local echo + 差量同步)体感接近本地
- 服务器侧的 Claude Code 直连 Anthropic API 最稳(VPS 国际出口比 Beam Pro 4G 强)
- `git pull` / `docker pull` 等海外工具链顺畅

音频不绕服务器 —— Beam Pro 直接到豆包(境内一跳),不污染国际出口。

---

# § 2. 备选方案:Voice Gateway 双端架构(保留)

> **状态**:未选为当前主推方案。原文档(本节即原 docs/06 v1 的设计)完整保留,在以下场景重新评估时仍是合理设计。

## 2.0 何时重新评估

回到 Voice Gateway 方案值得在以下任一情况触发:

- **不再使用 Claude Code 作为主 agent** — 比如换 Codex CLI(其 plan/confirm 体验更弱)或自建 LLM agent
- **音频必须不出云** — 隐私合规要求 ASR 本地化(需要 faster-whisper on server)
- **多个 Beam Pro / 多设备共享同一语音服务** — Voice Gateway 是天然的多 client 服务,主推方案是单设备绑定
- **需要 top-3 候选 + Tab 切换的体验** — 主推方案的 plan 模式给 1 个建议,要在 N 个 plausible 实现间挑时不如 top-3 高效
- **不用 tmux 或 Claude Code 无法常驻** — Voice Gateway 不依赖 tmux/agent 常驻
- **要细粒度的命令 risk 标记 + 强制二次确认** — Voice Gateway 在 prompt 层强制,主推方案靠 Claude Code 的权限模式(较粗)

如果上面有任意一条命中,回头读 §2.1 - §2.11 评估是否值得切换。

## 2.1 设计原则

1. **terminal 主路径不动** — SSH + tmux,语音是 augmentation,不是替代
2. **语音不是键盘替代,是意图通道** — LLM 做"自然语言意图 → shell 命令"的翻译;不试图让 ASR 直接吐出 `/var/log/nginx/access.log` 这种字符级精确字符串
3. **双端架构** — Beam Pro 端负责事件捕获和 UI,云端负责 ASR/LLM/注入;不同关注点分离
4. **优雅降级** — 网络断、ASR 崩、LLM 错,任何时刻能退回纯键盘继续工作
5. **不依赖 root / 越狱** — 所有组件用标准 Android 权限和云端 user shell 权限

## 2.2 整体架构

```
┌──────────────────────────────┐           ┌──────────────────────────────┐
│         Beam Pro             │           │       云端 Ubuntu             │
│                              │           │                              │
│  [SSH client]──SSH───────────┼───────────┼──→  tmux session: dev         │
│       ↑                      │           │                ↑              │
│       │ stdin/stdout         │           │                │ send-keys    │
│                              │           │                              │
│  [Voice/Key Daemon]          │           │  [Voice Gateway]              │
│  (Android Foreground Service)│           │  (Python FastAPI + WS)        │
│   - 监听蓝牙 HID 按键事件     │           │   - ASR(faster-whisper /     │
│   - PTT 录音                 │           │     OpenAI Whisper API)      │
│   - 流式上传音频 ────────────┼─WSS──────→│   - tmux capture-pane 抓上下文│
│   - 接收候选 ←───────────────┼─WSS──────│   - Claude API → top-3 候选    │
│   - SystemAlertWindow overlay│           │   - tmux send-keys 注入       │
│   - 按键选择 ────────────────┼─WSS──────→│   - 会话状态(lang/session)   │
└──────────────────────────────┘           └──────────────────────────────┘
```

3 个组件:**Voice/Key Daemon**(Beam Pro)、**Voice Gateway**(云端 Python 服务)、**SSH client + tmux**(零改造)。

## 2.3 关键技术决策

### 2.3.1 命令注入:`tmux send-keys`

云端 Voice Gateway 拿到 LLM 翻译的命令后直接 `tmux send-keys -t dev "..."`(不带 Enter,留给用户最后确认)。Termius 那边的 terminal 流自然出现这一行字符。

不需要在 Beam Pro 端处理任何文本注入。

### 2.3.2 候选 UI:Beam Pro `SYSTEM_ALERT_WINDOW` overlay

LLM 返回 top-3 候选,Beam Pro overlay 显示:
- Tab 切候选
- Enter 选定
- Esc 取消

### 2.3.3 ASR 位置

| 方案 | 延迟 | 网络 | 隐私 |
|---|---|---|---|
| 云端 faster-whisper | 0.5-1s | SSH 隧道 | 自主 |
| OpenAI Whisper API | 1-2s | 强网络 | 数据出云 |
| Beam Pro 本地 whisper.cpp | 0.3-0.8s | 无 | 全本地 |

MVP 全云端,Phase 3 加本地兜底。

### 2.3.4 LLM Prompt

Voice Gateway 抓 context(tmux capture-pane + history + cwd)拼 prompt,要求 Claude 返回 JSON 化的 top-3:

```jsonc
{
  "candidates": [
    { "cmd": "tail -100 /var/log/nginx/error.log", "explain": "...", "confidence": 0.92, "risk": "read_only" },
    { "cmd": "journalctl -u nginx --since '10 min ago'", "explain": "...", "confidence": 0.78, "risk": "read_only" },
    { "cmd": "tail -f /var/log/nginx/error.log | grep -i error", "explain": "...", "confidence": 0.65, "risk": "read_only_blocking" }
  ]
}
```

`risk` 字段约定:
- `read_only`:纯读
- `read_only_blocking`:read-only 但卡终端(`tail -f` / `watch`)
- `mutating`:写文件 / 改服务 — 二次确认
- `destructive`:`rm -rf` / `drop` — 默认不出候选

### 2.3.5 中英语种键

降级为 ASR 的 `language` hint + LLM prompt 中的 `语种=zh|en` 提示。不是核心特性。

## 2.4 WebSocket 接口定义(摘要)

```jsonc
// client → server
{ "type": "hello", "device": "...", "session": "dev" }
{ "type": "audio_chunk", "req_id": "r_abc", "seq": 0, "lang": "zh", "data_b64": "..." }
{ "type": "audio_end", "req_id": "r_abc" }
{ "type": "select", "req_id": "r_abc", "candidate_index": 0 }
{ "type": "cancel", "req_id": "r_abc" }

// server → client
{ "type": "asr_final", "req_id": "r_abc", "text": "..." }
{ "type": "candidates", "req_id": "r_abc", "items": [ ... ] }
{ "type": "injected", "req_id": "r_abc", "candidate_index": 0 }
{ "type": "error", "req_id": "r_abc", "code": "asr_empty", "msg": "..." }
```

## 2.5 延迟预算

| 环节 | 本地 ASR | 云端 ASR(良好网络)| 云端 ASR(4G 弱信号) |
|---|---|---|---|
| KeyUp → 录音停止 | 30ms | 30ms | 30ms |
| 音频上传 | 0 | 200-500ms | 1-3s |
| ASR | 300-800ms | 1-2s | 1-2s |
| 拉 tmux context | 100ms | 100ms | 100ms |
| Claude API 首 token | 600ms | 600ms | 1-2s |
| Claude 完整 top-3 | +800ms | +800ms | +1-2s |
| 候选下发 | 100ms | 100ms | 300-800ms |
| **总计** | **~2.0s** | **~3.0s** | **~5-8s** |

## 2.6 分阶段实施(原 Phase 0-4)

- **Phase 0**(1 周):录 30 句做可行性硬验证,可执行率 > 70% 才进 Phase 1
- **Phase 1**(2-3 周):MVP,Tasker + AutoApps 监听按键 + 录音 + 上传;云端 200 行 FastAPI;单候选,无 overlay
- **Phase 2**(2-3 周):Kotlin Service + SYSTEM_ALERT_WINDOW overlay,LLM top-3 + Tab 切换 + Enter 选定
- **Phase 3**(2-4 周):本地 ASR(whisper.cpp)+ 流式 LLM
- **Phase 4**(持续):语种、边缘情况、prompt 调优

## 2.7 失败模式(摘要)

- 蓝牙断 / 麦克风占用 / WS 断:Daemon 端状态栏提示,自动重连
- ASR 模型挂:systemd 重启
- Claude API 限流:降级到 ASR 文本直接 send-keys,用户自己改
- tmux session 不存在:Gateway 自动创建

---

# § 3. 与 repo 其它部分的关系

## 3.1 新增 / 修改

主推方案 §1 落地涉及:
- **新增**:`android/voice-daemon/` — Beam Pro 上的 Voice Daemon 代码(Stage B 开始)
- **不动**:`scripts/sysinfo`、`ls-html`、`log-view` 等 — 继续作为 UI 多样性补丁的第 1 层
- **不动**:Nginx serve HTML 那条路径 — 与本设计正交

备选方案 §2 落地涉及(若启用):
- **新增**:`gateway/` Python Voice Gateway
- **新增**:`android/voice-key-daemon/` Beam Pro Daemon(含 overlay)

## 3.2 与 docs/03 的衔接

主推 §1 直接使用 docs/03 §硬件方案一(8BitDo)+ §架构方案二(本地注入,剪贴板路径)+ §热词持续优化。如果将来 docs/03 的硬件选型或注入方案改了,本文 §1 跟随调整。

## 3.3 与 docs/05-roadmap 的对齐

`docs/05-roadmap.md` 的"近期"建议按本文 §1 Stage A 重新排序:**先验证 8BitDo PTT + 豆包 ASR 准确率,再决定后续投入**。

---

# § 4. 未解决问题(进 Stage A 前需明确)

按优先级排:

1. ⚠️ **"Enter 一键发送"的注入机制**(见 §1.3.1.1)— 优先验 Termius Intent 接收、其次 Accessibility ACTION_PASTE、备选 Shizuku。如果都不通,要么接受 Accessibility 权限,要么降级 Enter 语义(只关 overlay + 用户再按粘贴键)
2. **8BitDo Micro 是否支持发 F13/F14 键码** — 不支持时退到 Ctrl+Alt+1/2 组合键
3. **8BitDo Micro 的 PTT 持续键码能力** — Ultimate Software 是否支持"按住时持续发某键码",决定 PTT 是否可行
4. **`SYSTEM_ALERT_WINDOW` overlay focusable=true 时的焦点切换** — 关 overlay 后焦点能否干净地还给 Termius;Termius 是否兼容这种瞬态焦点切换
5. **豆包 ASR 对你的真实指令准确率** — Stage A 的核心人工评估项,门槛 ≥ 70%
6. **Claude Code plan 模式的稳定性** — Claude Code session 长跑(>4h)是否会因 context 累积而行为漂移
7. **Beam Pro 后台稳定性** — Foreground Service 在长时间 idle 时是否被 Android 杀(NebulaOS 的具体策略未知)

---

# § 5. 总结一句话

**主推**:Beam Pro 上一个最小 Voice Daemon — 监听 F13/F14 (PTT) → 录音 → 豆包 ASR → `SYSTEM_ALERT_WINDOW` overlay 显示预览 → 用户按 Enter(发送)/ Esc(撤销)/ F13-F14(重说)→ 通过 Termius Intent 或 Accessibility 把文本送进 Termius → SSH → Claude Code plan → 用户确认执行。**只新增 F13/F14 两个物理键 + 一个 Android Service**,其他复用现成的。

**备选**:不用 Claude Code、要本地 ASR、多设备共享、top-3 候选体验等场景下,回到 §2 的 Voice Gateway 双端架构。
