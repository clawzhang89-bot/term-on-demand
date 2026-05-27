# 06. 语音交互执行形态

## 0. 这份文档解答什么

承接 [`docs/03-input.md`](03-input.md) 和 [`docs/04-workflow.md`](04-workflow.md):

> 在 AR 眼镜 + Beam Pro + 蓝牙小键盘 + 中英语音 这套硬件约束下,
> **语音 → 命令 → 终端执行** 这条路径具体怎么实施。

也就是把"按一下中文键说话,terminal 里就出现命令"这件事拆到组件、接口、延迟、失败模式的颗粒度,目标是任何工程师拿到这份文档就能开始动手。

---

## 0.5 与 docs/03 的边界关系(重要)

本文(06)与 [`docs/03-input.md`](03-input.md) 讨论的是**同一个交互问题在两个不同场景下的两条路径**,不是替代关系,也不是版本演进关系。读这两份文档时请先看清各自的覆盖范围:

| 维度 | docs/06(本文) | docs/03 |
|---|---|---|
| **目标场景** | SSH + tmux 的远程终端开发(本项目的核心场景) | 非 tmux 通用 App 输入(浏览器地址栏、聊天 App、其他 Android App)|
| **文本"注入"发生在哪** | **云端**(`tmux send-keys` 直接进 SSH session) | **Beam Pro 端**(IME / AccessibilityService 注入到当前焦点 App) |
| **Beam Pro 端是否需要处理文本注入** | **不需要**(只负责按键事件捕获 + 麦克风 + 候选 overlay) | **需要**(IME 注入到 EditText / AccessibilityService 注入到 UI 节点) |
| **能否承载"意图 → LLM 翻译 → shell 命令"** | 是(核心路径) | 否(03 的方案是字面文本注入,不经过 LLM 翻译) |
| **是否需要 Android Accessibility 权限** | 否 | 是(03 的方案二)|

### 为什么 06 不走 IME / AccessibilityService 路径

在 SSH + tmux 场景下,**Beam Pro 上的 SSH client 不是"被注入文本"的目标**,而是"被动接收 tmux 推过来的字节流"的显示器。云端 `tmux send-keys -t dev "命令"` 等价于"有人在隔壁帮你打字",Termius/Termux 那边的 terminal 流里自然出现这一行,完全绕开了 docs/03 讨论的"语音输出的文字应该通过什么渠道送进 App"这个问题。

也就是说:**docs/03 的"三大方案对比"在 SSH+tmux 场景下整段不进入决策空间**;那个对比是为了解决非 tmux 通用 App 输入场景而存在的。

### 两份文档怎么共存

- 如果你 100% 都在 SSH+tmux 里工作:按 06 实施即可,03 的 IME/AccessibilityService 讨论不需要落地
- 如果你也有"在 Beam Pro 浏览器地址栏说话输入网址"这种需求:那部分按 03 实施(IME 或 AccessibilityService),与 06 在同一个 Voice/Key Daemon 进程里共存即可 — 按键事件捕获、PTT 录音、ASR 三个组件可以复用,只是输出去向(tmux send-keys vs 本地 IME/Accessibility)按当前 App 路由

### 如果将来 03 的范围调整

如果 docs/03 后续调整为也覆盖 tmux 场景(比如选了某种本地注入路径与 06 直接竞争),那两份文档需要做一次合并 review,确定 single source of truth。目前(2026-05)按上表的分工读即可。

---

## 1. 设计原则

1. **terminal 主路径不动** — SSH + tmux,语音是 augmentation,不是替代
2. **语音不是键盘替代,是意图通道** — LLM 做"自然语言意图 → shell 命令"的翻译;不试图让 ASR 直接吐出 `/var/log/nginx/access.log` 这种字符级精确字符串
3. **双端架构** — Beam Pro 端负责事件捕获和 UI,云端负责 ASR/LLM/注入;不同关注点分离
4. **优雅降级** — 网络断、ASR 崩、LLM 错,任何时刻能退回纯键盘继续工作
5. **不依赖 root / 越狱** — 所有组件用标准 Android 权限和云端 user shell 权限
6. **场景边界明确**:本设计假设 SSH + tmux 场景;非 tmux 通用 App 输入的方案见 [`docs/03-input.md`](03-input.md) 的 IME/AccessibilityService 讨论(详见 §0.5)

---

## 2. 整体架构

```
┌──────────────────────────────┐           ┌──────────────────────────────┐
│         Beam Pro             │           │       云端 Ubuntu             │
│  ──────────────              │           │  ──────────────                │
│                              │           │                              │
│  [SSH client]──SSH───────────┼───────────┼──→  tmux session: dev         │
│  Termius/Termux/Blink        │           │      (你的 shell 长期住在这)   │
│       ↑                      │           │                ↑              │
│       │ stdin/stdout         │           │                │ send-keys    │
│       │ (terminal 流)        │           │                │              │
│                              │           │                              │
│  [Voice/Key Daemon]          │           │  [Voice Gateway]              │
│  (Android Foreground Service)│           │  (Python FastAPI + WS)        │
│   - 监听蓝牙 HID 按键事件     │           │   - ASR(faster-whisper /     │
│   - PTT 录音(AudioRecord)   │           │     OpenAI Whisper API)      │
│   - 流式上传音频 ────────────┼─WSS──────→│   - tmux capture-pane 抓上下文│
│   - 接收候选 ←───────────────┼─WSS──────│   - Claude API → top-3 候选    │
│   - SystemAlertWindow overlay│           │   - tmux send-keys 注入       │
│   - 按键选择 ────────────────┼─WSS──────→│   - 会话状态(lang/session)   │
│                              │           │   - 仅监听 127.0.0.1:18800   │
└──────────────────────────────┘           │     (SSH 隧道或 Tailscale 暴露)│
                                            └──────────────────────────────┘

依赖关系:Voice Gateway 进程独立于 Nginx,本设计不复用 nginx 这条路径。
        语音通道走单独的 WebSocket over SSH/Tailscale。
```

3 个核心组件:**Voice/Key Daemon**(Beam Pro)、**Voice Gateway**(云端)、**现成的 SSH client + tmux**(零改造)。

---

## 3. 组件分解

### 3.1 Voice/Key Daemon(Beam Pro,Android Foreground Service)

| 项 | 选择 |
|---|---|
| 语言 | Kotlin(原生)/ React Native(MVP 可用) |
| 形态 | 前台 Service + Notification(确保不被系统杀)|
| 权限 | `RECORD_AUDIO`, `SYSTEM_ALERT_WINDOW`, `BLUETOOTH_CONNECT` |
| 不需要 | root / Accessibility Service |

职责:
- **HID 事件捕获**:Android `InputDevice` API 拿到蓝牙 macropad 按键(QMK 键位 → 自定义 keycode)
- **PTT 录音**:按下"语音键"时 `AudioRecord` 16kHz mono PCM 开始;松开停止
- **音频流式上传**:每 200ms 一个 chunk 通过 WebSocket 发到 Voice Gateway,不等录音结束(降首字延迟)
- **Overlay 渲染**:候选条用 `WindowManager.addView(LayoutParams.TYPE_APPLICATION_OVERLAY)` 浮在 SSH client 之上
- **按键选择反馈**:Tab 切候选,Enter 选定,Esc 取消 → 通过 WS 告知 Gateway

### 3.2 Voice Gateway(云端 Ubuntu,Python FastAPI)

| 项 | 选择 |
|---|---|
| 语言 | Python 3.11+ (FastAPI + uvicorn) |
| 部署 | systemd service,`User=foxer`(不要 root) |
| 监听 | `127.0.0.1:18800`(仅本地,通过 SSH 端口转发或 Tailscale 暴露)|
| 持久化 | 无(无状态,重启不丢任何关键数据) |

职责:
- 接收音频 chunk → 调 ASR(策略见 §6.3)
- 调 `tmux capture-pane / display-message / list-windows` 抓 context
- 拼 prompt 调 Claude API,要 JSON 化的 top-3 候选
- 收到 select → 调 `tmux send-keys -t $session "$cmd"`(**不带 Enter**,留给用户手动 Enter)
- 全程 WS 双向消息

### 3.3 SSH client + tmux(零改造)

- SSH client 任选(Termius / Termux + ssh / Blink Shell / Mosh)
- 云端 `~/.bashrc` 或 SSH `RemoteCommand` 自动 `tmux new -A -s dev`,确保你 SSH 进去就在 tmux 里
- tmux session 命名约定:`dev`(每台服务器一个固定名),Voice Gateway 默认操作这个 session

---

## 4. 数据流详解(一次完整交互)

```
T+0ms      用户按下"中文键"(QMK keycode 0xC0)
T+30ms     Daemon: 收到 KeyDown 事件 → AudioRecord.start()
T+30ms..   用户说"看下 nginx 最近的错误"
T+3030ms   用户松开按键
T+3050ms   Daemon: AudioRecord.stop(),flush 最后 chunk
T+30..3050 Daemon → Gateway: WS audio_chunk × N(流式,不等结束)
T+3070ms   Gateway: ASR 处理最后 chunk,产出 final text
T+3170ms   Gateway: `tmux capture-pane -p -S -30` + 当前目录 + 最近命令
T+3270ms   Gateway → Claude API(stream)
T+4500ms   Claude: 首个候选完整 token 出来
T+4550ms   Gateway → Daemon: candidates 消息(top-3)
T+4650ms   Daemon: overlay 渲染候选条
T+4650ms+  用户看候选,按 Tab/Enter/Esc
T+N        Daemon → Gateway: select {index: 0}
T+N+30ms   Gateway: `tmux send-keys -t dev "tail -100 /var/log/nginx/error.log"`
T+N+50ms   命令字面出现在 terminal(用户视角)
T+N+50+    用户按 Enter 执行(或编辑修改)
```

**总体感**:从松开 PTT 到候选出现,**~1.5s(本地 ASR)~3s(云端 ASR + 良好网络)~5s(4G 弱信号)**。

---

## 5. 接口定义

### 5.1 WebSocket 连接

`wss://gateway.local:18800/ws?device=beam_pro_001&session=dev`

`device` 用于多设备区分,`session` 是 tmux session name。

### 5.2 Beam Pro → Voice Gateway(client → server)

```jsonc
// 会话初始化
{ "type": "hello", "device": "beam_pro_001", "session": "dev", "version": "0.1" }

// 流式音频块
{ "type": "audio_chunk", "req_id": "r_abc", "seq": 0, "lang": "zh", "data_b64": "<pcm16 chunk>" }
{ "type": "audio_chunk", "req_id": "r_abc", "seq": 1, "lang": "zh", "data_b64": "..." }

// 录音结束信号
{ "type": "audio_end", "req_id": "r_abc" }

// 候选选择
{ "type": "select", "req_id": "r_abc", "candidate_index": 0 }

// 用户取消(按 Esc)
{ "type": "cancel", "req_id": "r_abc" }

// 心跳
{ "type": "ping" }
```

### 5.3 Voice Gateway → Beam Pro(server → client)

```jsonc
// ASR 部分结果(可选,流式展示)
{ "type": "asr_partial", "req_id": "r_abc", "text": "看下 nginx" }

// ASR 最终结果
{ "type": "asr_final", "req_id": "r_abc", "text": "看下 nginx 最近的错误" }

// 候选命令
{
  "type": "candidates",
  "req_id": "r_abc",
  "items": [
    {
      "cmd": "tail -100 /var/log/nginx/error.log",
      "explain": "查看 nginx 错误日志最近 100 行",
      "confidence": 0.92,
      "risk": "read_only"
    },
    {
      "cmd": "journalctl -u nginx --since '10 min ago' --no-pager",
      "explain": "通过 systemd 看最近 10 分钟 nginx 日志",
      "confidence": 0.78,
      "risk": "read_only"
    },
    {
      "cmd": "tail -f /var/log/nginx/error.log | grep -i error",
      "explain": "实时跟踪 nginx 错误",
      "confidence": 0.65,
      "risk": "read_only_blocking"
    }
  ]
}

// 注入完成
{ "type": "injected", "req_id": "r_abc", "candidate_index": 0 }

// 错误
{ "type": "error", "req_id": "r_abc", "code": "asr_empty", "msg": "未识别到内容" }
{ "type": "error", "req_id": "r_abc", "code": "llm_failed", "msg": "LLM 调用失败,可重试" }
{ "type": "error", "req_id": "r_abc", "code": "tmux_no_session", "msg": "tmux session 'dev' 不存在" }

// 心跳响应
{ "type": "pong" }
```

`risk` 字段约定:
- `read_only`:纯读,无副作用
- `read_only_blocking`:read-only 但会卡住 terminal(`tail -f`、`watch` 等)
- `mutating`:会写文件 / 改服务 / 改数据库 — overlay 用红色高亮,Enter 选定后不立即 send-keys,要求二次确认
- `destructive`:`rm -rf`、`drop`、`reset --hard` 等 — 默认不出现在候选里,LLM prompt 显式禁用

---

## 6. 关键技术决策(每个有备选 + 推荐)

### 6.1 命令注入机制

**决策**:`tmux send-keys`。

| 维度 | tmux send-keys | HID emulation |
|---|---|---|
| 通用性 | 仅 tmux 会话 | 任何 input field |
| 权限 | 云端 user shell 已有 | Android Accessibility Service / root |
| 复杂度 | 一行 shell | InputManager 模拟键码序列 |
| 与 form factor 匹配度 | 高(云端开发标配 tmux) | 通用但 overkill |

**风险**:用户必须先 `tmux attach`。
**缓解**:
- SSH client 的 `RemoteCommand` 配置 `tmux new -A -s dev`(`-A` = attach if exists, else new)
- Voice Gateway 启动时确保 named session 存在,不存在则创建:`tmux has-session -t dev || tmux new-session -d -s dev`

### 6.2 候选 UI 形态

**决策**:Beam Pro `SYSTEM_ALERT_WINDOW` overlay。

| 方案 | 工作量 | 体验 | 备注 |
|---|---|---|---|
| **Android overlay** | 中(Kotlin Service) | 像 IME 候选条,符合"AI 浮窗" | 推荐 |
| terminal 内嵌(fzf 风格)| 高(需自造 terminal 或 hack tmux)| 最原生 | 数月级工作量,排除 |
| 独立浏览器 tab | 低 | 打断 terminal focus | 排除 |
| Beam Pro 状态栏通知 | 低 | 选择交互困难 | 排除 |

### 6.3 ASR 位置

**决策**:**MVP 阶段全云端,Phase 3 加本地兜底**。

| 方案 | 延迟 | 网络依赖 | 准确率 | 隐私 |
|---|---|---|---|---|
| OpenAI Whisper API | 1-2s | 强 | 高 | 数据出云 |
| 云端自部署 faster-whisper(large-v3 量化)| 0.5-1s | 弱(SSH 隧道)| 最高 | 自主 |
| **Beam Pro 本地 whisper.cpp(small/base 量化)** | 0.3-0.8s | 无 | 中 | 全本地 |

**MVP**(Phase 1):云端 faster-whisper large-v3,工作量最低、准确率最高。

**Phase 3**:加 Beam Pro 本地 small/base 作为快路径 — 本地秒出粗结果,云端 large 异步精修,差异 > threshold 时弹候选;断网时本地兜底。

**Snapdragon 7 Gen 2 跑 whisper.cpp small 量化的实际延迟需要 Phase 0 实测确认**(本设计假设可达 < 1s)。

### 6.4 LLM 选择

**决策**:Claude(Sonnet 4.6 入门,降本可切 Haiku 4.5)。

| 维度 | Claude Sonnet 4.6 | Gemini 2.5 Pro | 本地 qwen2.5-coder 32B |
|---|---|---|---|
| shell 知识深度 | 高 | 中 | 中 |
| 意图理解 | 强 | 强 | 弱(无对话上下文) |
| 工具调用 / JSON 模式 | 成熟 | 成熟 | 需 prompt engineering |
| 延迟(首 token)| 300-600ms | 400-800ms | 取决于硬件 |
| 成本 / 1000 次翻译 | ~$0.5(Sonnet) ~$0.1(Haiku)| ~$0.4 | 自建硬件分摊 |

Claude 的强项是它对常见 Linux/macOS shell 范式的判断更稳(`journalctl` vs `tail` vs `less`,什么时候加 `--no-pager`,什么时候用 `-f`)。

### 6.5 LLM Prompt 设计

System prompt(精简):

```
你是 AR 眼镜 + tmux terminal 的语音助手。
用户用语音表达"想做什么",你输出 3 个 shell 命令候选,JSON 格式。

约束:
- 命令必须在给定的服务器环境上可直接执行
- 优先 read-only 命令;mutating 必须标 risk=mutating
- 禁止 destructive(rm -rf / drop / reset --hard / chmod 777),除非用户明确说"删除/重置"
- 命令尽量短,适合 AR 眼镜阅读(单行 ≤ 80 字符)
- 不要包含 sudo,除非用户明示
- 不要管道到 less / more(会卡住 tmux),用 tail/head 限定行数

输出 JSON schema:
{ "candidates": [{ "cmd": "...", "explain": "...", "confidence": 0..1, "risk": "read_only|read_only_blocking|mutating" }] }
```

User message:

```
服务器: ubuntu-prod-01 (Ubuntu 24.04)
shell: zsh
当前目录: /home/foxer/app
最近 10 个命令:
  git status
  docker ps
  ...
最近 terminal 输出(最后 30 行):
  [tmux capture-pane]
最近一次 ASR 文本(语种=zh): "看下 nginx 最近的错误"
```

### 6.6 中英语种切换的角色

**决策**:保留硬件键,但只作为 ASR 偏置 hint,不作为系统稳定性的核心依赖。

理由(详见 issue 讨论):
- 2026 SOTA 多语种 ASR 对 zh-en 混说 WER 已可控
- 真正难的是 shell 路径/符号 — 语种切换救不了
- 通过 LLM 翻译层吸收多语种波动后,硬切换的边际价值估计 < 5-10%
- 但代价低(键位本来就有,ASR API 的 `language` 参数本来就要传),保留无损

实操:
- 中文键 → ASR 调用时 `language="zh"` + `initial_prompt="以下是中文语音指令"`
- 英文键 → `language="en"` + `initial_prompt="The following is an English voice command"`
- LLM prompt 末尾附 `语种=zh|en` 让 Claude 决定 explain 用什么语言

---

## 7. 延迟预算与网络降级

| 环节 | 本地 ASR | 云端 ASR(良好网络) | 云端 ASR(4G 弱信号) |
|---|---|---|---|
| KeyUp → 录音停止 | 30ms | 30ms | 30ms |
| 音频上传 | 0 | 200-500ms | 1-3s |
| ASR 处理 | 300-800ms | 1-2s | 1-2s |
| ASR text → Gateway | 50ms | 0 | 0 |
| 拉 tmux context | 100ms | 100ms | 100ms |
| Claude API 首 token | 600ms | 600ms | 1-2s |
| Claude 完整 top-3 | +800ms | +800ms | +1-2s |
| 候选下发 | 100ms | 100ms | 300-800ms |
| **总计** | **~2.0s** | **~3.0s** | **~5-8s** |

加上用户按 Tab/Enter 的反应 + tmux send-keys 注入 ~300ms。

**降级策略**:

| 触发 | 行为 |
|---|---|
| ASR 延迟 > 5s | overlay 提示"网络慢,Esc 取消" |
| ASR 返回空 | 红点提示"未识别,请重说",不打扰 terminal |
| LLM 调用失败 | 自动重试 1 次;再失败则降级:把 ASR 文本直接 send-keys 到 terminal,用户自己改 |
| LLM 返回非法 JSON | 同上 |
| 完全离线 | Daemon 进入"local fallback":只显示 ASR 文本,不出候选;用户手动改 |
| tmux session 不存在 | Gateway 自动创建,提示一次"已重建 dev session" |

---

## 8. 失败模式清单

| 模式 | 检测 | 用户体感 | 恢复 |
|---|---|---|---|
| 蓝牙断开 | Daemon 收不到 KeyEvent | 按键无反应 | Daemon 后台尝试重连;状态栏图标变红 |
| 麦克风被其他 app 占用 | `AudioRecord.startRecording` 失败 | overlay 弹"麦克风被占用" | 用户手动关闭占用 app |
| WS 连接断 | onClose 事件 | 按键时 overlay 弹"未连接" | 自动重连(指数退避);Daemon 状态栏显示连接状态 |
| ASR 模型加载失败 | Gateway 启动日志 | 静默,Gateway 不响应 hello | systemd 自动重启;运维介入 |
| Claude API 限流 | HTTP 429 | overlay 弹"LLM 限流,改用本地兜底" | 用 ASR 文本直接 send-keys |
| tmux pane 已被关闭 | `tmux send-keys` 退出码非 0 | 命令出不来 | 创建新 session,提示用户重新 attach |
| 用户说错了 | — | overlay 出现明显不对的候选 | 按 Esc → 整条对话取消;按 Tab 看其他候选 |

---

## 9. 分阶段实施

### Phase 0:可行性硬验证(1 周,纯手动)

**不写任何 app**。在你自己的开发机上:

```bash
# 录 30 句典型指令
ffmpeg -f avfoundation -i ":0" -ar 16000 -ac 1 -t 5 utterance_01.wav
# ...重复 30 次

# 批量 ASR
for f in utterance_*.wav; do
  curl https://api.openai.com/v1/audio/transcriptions \
    -F file=@$f -F model=whisper-1 -F language=zh > "${f%.wav}.txt"
done

# 批量 LLM 翻译
for t in utterance_*.txt; do
  claude -p "$(cat prompt-template.txt) 用户说:$(cat $t)" > "${t%.txt}.cmd.json"
done

# 人工标注:每条命令"能不能在我服务器上直接跑通"
# 测三个指标:WER / 可执行率 / 端到端时间
```

**门槛**:可执行率 > 70% 才进 Phase 1。否则:换模型 / 改 prompt / 调整意图表达方式,再测。

### Phase 1:MVP(2-3 周)

**Beam Pro 端**:用 [Tasker](https://tasker.joaoapps.com/) + AutoApps 监听蓝牙键盘 + 录音 + curl 上传(零原生代码)。

**云端 Voice Gateway**:200 行 FastAPI:

```
gateway/
├── main.py              # FastAPI + WS endpoint
├── asr.py               # OpenAI Whisper API 封装
├── llm.py               # Claude API 封装,prompt + JSON parse
├── tmux.py              # send-keys, capture-pane wrapper
├── prompt.txt           # LLM system prompt
└── systemd/
    └── voice-gateway.service
```

**简化**:不做 overlay,LLM 只返回 1 个候选,直接 send-keys 到 terminal(不带 Enter),用户在 terminal 上看命令,按 Enter 执行 / Ctrl+C 否决。

**目标**:端到端跑通"按键 → 说话 → 命令出现在 terminal"。

### Phase 2:候选 + Overlay(2-3 周)

- Beam Pro 端写 Kotlin Foreground Service + `SYSTEM_ALERT_WINDOW` overlay
- LLM 返回 top-3
- Tab 切换、Enter 选定、Esc 取消
- 加 tmux capture-pane context
- 加 risk 字段 + mutating 二次确认

### Phase 3:本地 ASR + 流式(2-4 周)

- whisper.cpp 集成到 Android(JNI)
- 本地 small 模型快路径,云端 large 异步精修
- 音频流式上传,LLM 流式输出,首 token 延迟 < 1s

### Phase 4:语种 + 边缘 + prompt 调优(持续)

- 中英语种键的 ASR / LLM 联动
- 各种 mode(命令模式 / 描述模式 / 路径模式)
- LLM prompt 持续根据失败案例调优
- 多 tmux session 切换支持

---

## 10. 与现有 repo 的关系

### 新增

- 本文 `docs/06-voice-interaction-execution.md`
- 后续 PR(单独):
  - `gateway/` — Voice Gateway Python 源码
  - `android/` — Voice/Key Daemon Kotlin 项目
  - `prompts/` — LLM prompt 模板

### 不动

- `scripts/sysinfo`, `ls-html`, `log-view` 等 — 继续作为 UI 多样性补丁的第 1 层(高频固定视图)
- Nginx serve HTML 那条路径 — 与本设计正交,不冲突

### 建议修改(单独 PR,不在本 PR 内)

- `docs/03-input.md` 在"架构方案对比"开头加一句限定:"以下三个方案讨论的是非 tmux 通用 App 输入场景;SSH+tmux 场景的语音→命令路径见 docs/06"(配合本文 §0.5 的边界声明)
- `docs/03-input.md` 的 Phase 0/1/2 上手路径与本文 Phase 0/1/2/3/4 编号重叠且含义不同,建议改名为 "Stage A/B/C" 之类避免读者混淆
- `docs/05-roadmap.md` 的"近期/中期"按本文 Phase 0-2 重新组织
- `README.md` 的"输入方案"段落里,在引用 03 的三方案对比时补一句"以下方案适用于非 tmux 场景;SSH+tmux 场景请见 docs/06"

---

## 11. 未解决问题(进 Phase 0 前需明确)

1. **Beam Pro 是否能跑 whisper.cpp small 量化在 < 1s 延迟内?** Phase 0 必须实测,否则 Phase 3 路径要重设计。
2. **LLM context 截断策略** — 当 tmux pane 输出很长(`cat large.log`),context 怎么裁?最近 30 行?按 token 截?还是让 LLM 摘要?
3. **多 tmux session 切换** — 你在 dev / prod / staging 几个 session 间切换时,Voice Gateway 怎么跟踪 active session?(候选方案:从 SSH client 通过 OSC 序列上报当前 session,或 Gateway 周期 `tmux list-clients` 探测)
4. **Privacy 模型** — audio 是否上云?如果不上云,本地 ASR 是硬需求,Phase 3 不能延后
5. **多设备并发** — 你 Beam Pro 和笔记本可能同时 attach 到同一 tmux session,语音注入时该向哪边显示候选?
6. **Cost 上限** — 假设每天 50 次语音翻译,Sonnet 月成本约 ~$5;意外触发(误按)会不会失控?需要客户端 rate limit

---

## 12. 总结一句话

**Voice/Key Daemon(Beam Pro Android Service)+ Voice Gateway(云端 Python)+ tmux send-keys**,SSH client 和 terminal 不动,通过 WebSocket 把"按键事件 + 麦克风"和"ASR + LLM + 命令注入"连起来。本设计不依赖 nginx、不依赖 root、不依赖任何 SSH client 的内部 API,所有改造点集中在两个新组件上。
