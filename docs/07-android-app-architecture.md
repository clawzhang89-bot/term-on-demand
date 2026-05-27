# 07. Android App 架构:WebView + sshj + Voice Daemon 单 app 闭环

> **状态**:这份文档是 [`docs/06-voice-interaction-execution.md`](06-voice-interaction-execution.md) 经过多轮迭代后的最终方案。06 的多种变体(双端 Voice Gateway、剪贴板桥接 + Claude Code、Termux Intent 中转等)在本设计下被取代;06 仍有保留价值作为"未选方案的历史记录"和"特定场景下的备选"参考。

## 0. 这份文档解答什么

> 在 AR 眼镜 + Beam Pro + 蓝牙小键盘 + 中英语音 这套硬件约束下,**语音 + 终端交互**这条路径以最小工程量、最干净架构怎么落地。

答:**写一个 Android App,把 SSH client + 终端 UI + 语音输入全部塞进同一个进程**。一切跨 App 注入难题、双端通信复杂度、Voice Gateway 服务端额外组件——全部消失。

---

## 1. 设计原则

1. **单 Android App 闭环** — SSH 协议 / 终端渲染 / 按键事件 / 录音 / ASR / overlay,全部在一个 APK 内。没有跨 App 通信、没有 IME 注入、没有 Accessibility,没有 SYSTEM_ALERT_WINDOW
2. **零服务端增量** ⭐ — 服务端只跑你**已有的 tmux + Claude Code**。无 ttyd、无 nginx、无 Voice Gateway、无 tmux-send-keys daemon。从服务端运维角度,这个 App 跟一个普通 SSH client 没区别
3. **UI 完全 WebView 实现** — terminal 用 xterm.js,候选/状态 overlay 是 WebView 里的 HTML 元素。Android 端只管 SSH 字节流 + 按键事件 + 音频
4. **Voice 直写 SSH channel,跳过 xterm.js** — Voice Daemon 拿到 ASR 文本后直接写 SSH 输出流,字符通过 shell echo 在远端回显,xterm.js 渲染 — Voice Daemon 不需要知道 xterm.js 存在
5. **优雅降级** — App 挂了,你照样可以用 Termius / Termux / Blink 等任何 SSH client 连同一台服务器继续工作

---

## 2. 整体架构

```
┌─ Beam Pro 上的一个 APK ────────────────────────────────────┐
│                                                            │
│  ┌─ WebView(全屏 immersive)─────────────────────────┐   │
│  │  xterm.js + WebGL renderer + 自定义 CSS 主题      │   │
│  │  HTML overlay(Voice 预览框,纯 DOM 元素)         │   │
│  └──────────────────────────────────────────────────────┘   │
│         ↑ JS:term.write(bytes)     ↓ JS:onData(bytes)     │
│         │                           │                       │
│  ┌─ JSBridge(Base64 编码双向)───────────────────────┐    │
│  │  Kotlin ↔ JavaScript                              │    │
│  └────────────────────────────────────────────────────┘    │
│         ↑                           ↓                       │
│  ┌─ SSH 模块(sshj 0.39+)──────────────────────────┐     │
│  │  TCP socket → SSH 协议 → PTY                       │     │
│  │  inputStream.read()   outputStream.write()         │     │
│  └────────────────────────────────────────────────────┘    │
│         ↑                                                   │
│         │ Voice Daemon 直接调 outputStream.write(text)     │
│         │                                                   │
│  ┌─ Voice Daemon(Foreground Service)──────────────┐      │
│  │  HID 按键监听(F13/F14)                          │      │
│  │  AudioRecord → Opus → 豆包 ASR                    │      │
│  │  WebView.evaluateJavascript("showOverlay('...')") │      │
│  │  Enter 确认 → sshSession.outputStream.write(text) │      │
│  └────────────────────────────────────────────────────┘    │
│                                                            │
└────────────────┬────────────────────────────────────────────┘
                 │ Raw SSH (port 22)
                 ▼
       海外 Ubuntu 服务器
       └─ tmux: dev session → claude code --resume
       (没有 ttyd / nginx / Voice Gateway —— 跟标准 SSH 接入完全一样)
```

**3 个内部模块**(WebView + SSH + Voice Daemon),**1 个出口**(SSH 到云端)。

---

## 3. 关键组件

### 3.1 WebView + xterm.js(UI 层)

**职责**:terminal 渲染、用户输入捕获、overlay UI。

**实现**:
- `assets/terminal.html` 静态打包 [xterm.js](https://github.com/xtermjs/xterm.js) + addons:
  - `xterm-addon-fit` — 自动适配 WebView 尺寸
  - `xterm-addon-webgl` — 60fps GPU 渲染,大输出不卡
  - `xterm-addon-unicode11` — CJK 宽字符正确对齐
- 自定义 CSS 主题(暗色 + JetBrains Mono + 圆角 + 行间距等)
- HTML overlay 元素(`<div id="voice-overlay">`)用 JS 控制 show/hide,语义由 Kotlin 端通过 JSBridge 触发

**完全离线启动**:WebView `loadUrl("file:///android_asset/terminal.html")`,不需要任何网络拉取资源。

**关键代码骨架**(`assets/terminal.html`):

```html
<!doctype html>
<html><head>
  <link rel="stylesheet" href="xterm.css">
  <style>
    body { margin: 0; background: #11131a; }
    #term { height: 100vh; }
    #voice-overlay {
      position: fixed; bottom: 30px; left: 50%; transform: translateX(-50%);
      background: rgba(20, 22, 30, 0.95); backdrop-filter: blur(20px);
      padding: 16px 24px; border-radius: 12px; color: #e6e6e6;
      font-family: 'JetBrains Mono', monospace; min-width: 360px;
      box-shadow: 0 10px 40px rgba(0,0,0,0.5); display: none;
    }
  </style>
</head><body>
  <div id="term"></div>
  <div id="voice-overlay">
    <div id="overlay-status">🎤 录音中...</div>
    <div id="overlay-text" style="margin-top: 8px; color: #94e0b2;"></div>
    <div style="margin-top: 12px; font-size: 12px; color: #888;">
      Enter 发送 · Esc 撤销 · F13/F14 重说
    </div>
  </div>
  <script src="xterm.js"></script>
  <script src="addon-fit.js"></script>
  <script src="addon-webgl.js"></script>
  <script>
    const term = new Terminal({
      fontFamily: '"JetBrains Mono", monospace', fontSize: 14, lineHeight: 1.2,
      theme: { background: '#11131a', foreground: '#e6e6e6', cursor: '#94e0b2' },
      cursorBlink: true, scrollback: 10000,
    });
    const fitAddon = new FitAddon.FitAddon();
    term.loadAddon(fitAddon);
    term.loadAddon(new WebglAddon.WebglAddon());
    term.open(document.getElementById('term'));
    fitAddon.fit();

    // 用户敲键 → Kotlin
    term.onData(data => Bridge.onInput(btoa(data)));
    term.onResize(({cols, rows}) => Bridge.onResize(cols, rows));

    // 提供给 Kotlin 调用的 API
    window.writeToTerm = (b64) => term.write(Uint8Array.from(atob(b64), c => c.charCodeAt(0)));
    window.showOverlay = (status, text) => {
      document.getElementById('overlay-status').innerText = status;
      document.getElementById('overlay-text').innerText = text || '';
      document.getElementById('voice-overlay').style.display = 'block';
    };
    window.hideOverlay = () => document.getElementById('voice-overlay').style.display = 'none';
  </script>
</body></html>
```

### 3.2 SSH 模块([sshj](https://github.com/hierynomus/sshj))

**选型**:**sshj 0.39+** 主推。

**为什么 sshj**:
- Apache 2.0,Java/Kotlin 通用
- API 现代,异步 + Channel 模型清晰
- 历史上 Android 兼容性有 BouncyCastle 问题,**PR #636 已显著改善**,2026 版本 Stage A.2 实测确认即可

**Fallback**:如 sshj 在 Beam Pro / Android 14 上仍有 BouncyCastle 问题,退到:
- **[sshlib](https://github.com/connectbot/sshlib)** — ConnectBot 从 trilead-ssh2 fork 的版本,专为 Android 维护,无 BouncyCastle 依赖,**最稳的 Android fallback**
- **Apache MINA SSHD client** — 也成熟,但代码量大

**关键代码骨架**(Kotlin):

```kotlin
class SshConnection(
    private val host: String, private val port: Int,
    private val user: String, private val privateKeyPath: String
) {
    private lateinit var client: SSHClient
    private lateinit var session: Session
    private lateinit var shell: Session.Shell

    fun connect(cols: Int, rows: Int) {
        client = SSHClient(DefaultConfig()).apply {
            addHostKeyVerifier(OpenSSHKnownHosts(File("$filesDir/known_hosts")))
            connect(host, port)
            authPublickey(user, privateKeyPath)
        }
        session = client.startSession().apply {
            allocatePTY("xterm-256color", cols, rows, 0, 0, emptyMap())
        }
        shell = session.startShell()
    }

    fun outputStream(): OutputStream = shell.outputStream
    fun inputStream(): InputStream = shell.inputStream

    fun resize(cols: Int, rows: Int) {
        session.changeWindowDimensions(cols, rows, 0, 0)
    }

    fun disconnect() {
        runCatching { shell.close(); session.close(); client.disconnect() }
    }
}
```

**关键依赖**(`build.gradle.kts`):
```kotlin
dependencies {
    implementation("com.hierynomus:sshj:0.39.0")
    implementation("org.bouncycastle:bcprov-jdk18on:1.78.1")  // sshj 需要
    implementation("net.i2p.crypto:eddsa:0.3.0")              // Ed25519 key 支持
}
```

### 3.3 JSBridge(WebView ↔ Native 双向桥接)

**职责**:WebView 内 JS 跟 Kotlin Activity 之间双向传字节流。

**主方案:Base64 over `evaluateJavascript` / `@JavascriptInterface`**

- SSH 输出方向(Kotlin → JS):
  - Kotlin 后台线程读 `shell.inputStream`,每读到一个 chunk 调 `webView.evaluateJavascript("writeToTerm('$b64')")`
- SSH 输入方向(JS → Kotlin):
  - JS 端 `term.onData(data => Bridge.onInput(btoa(data)))`
  - Kotlin 端 `@JavascriptInterface fun onInput(b64: String)` 解码后写 `shell.outputStream`

**Base64 对 SSH 性能足够**:
- SSH 输出典型 < 100 KB/s(`top` / `tail -f` 等),Base64 开销 +33% 仍在 KB/s 量级
- 单次 `evaluateJavascript` 调用 < 1ms,60fps 输出无压力
- 大输出(`cat huge.log`)走 `less` / `head` 限定,本来就该这么用

**Fallback:localhost WebSocket**(如 Base64 在你的实测下抖)
- Kotlin 起一个 `127.0.0.1:0`(随机端口)WebSocket server,WebView 内 JS `new WebSocket('ws://127.0.0.1:PORT')`
- 二进制帧直传,零编码开销
- 多加 ~30 行代码,在 Stage A.3 如果发现 Base64 60fps 卡顿就切

**关键代码骨架**:

```kotlin
class TerminalBridge(private val ssh: SshConnection) {
    @JavascriptInterface
    fun onInput(b64: String) {
        val bytes = Base64.decode(b64, Base64.NO_WRAP)
        ssh.outputStream().write(bytes); ssh.outputStream().flush()
    }
    @JavascriptInterface
    fun onResize(cols: Int, rows: Int) { ssh.resize(cols, rows) }
}

// 主 Activity 里:
webView.addJavascriptInterface(TerminalBridge(ssh), "Bridge")
webView.settings.javaScriptEnabled = true
webView.loadUrl("file:///android_asset/terminal.html")

// 后台线程:SSH 输出 → WebView
thread {
    val buf = ByteArray(4096)
    while (true) {
        val n = ssh.inputStream().read(buf); if (n <= 0) break
        val b64 = Base64.encodeToString(buf.copyOf(n), Base64.NO_WRAP)
        runOnUiThread { webView.evaluateJavascript("writeToTerm('$b64')", null) }
    }
}
```

### 3.4 Voice Daemon(同 app 内 Foreground Service)

**职责**:监听 F13/F14 → 录音 → 调豆包 → overlay 显示 → Enter 时直接写 SSH。

**状态机**:

```
IDLE
  ↓ F13/F14 KeyDown
RECORDING(JS overlay 显示"🎤 录音中...")
  ↓ F13/F14 KeyUp
ASR_PENDING(JS overlay 显示"识别中...")
  ↓ 豆包返回 text
PREVIEW(JS overlay 显示 text + 操作提示)
  ├─ Enter → COMMIT:ssh.outputStream.write(text)  → hideOverlay → IDLE
  ├─ Esc → hideOverlay → IDLE
  └─ F13/F14 → hideOverlay 立即重进 RECORDING
```

**关键设计**:Voice Daemon **直接写 SSH outputStream**,**不经过 xterm.js**。
- 文本被 SSH 推到远端 shell
- 远端 shell 默认开启 echo,把收到的字符回送
- 字节流回到本地 xterm.js,渲染出文字
- xterm.js 完全不知道"语音"存在,行为跟"用户敲键"完全一样

这消除了一整类 "voice 怎么注入 xterm.js" 的集成问题。

**关键代码骨架**:

```kotlin
class VoiceDaemon(
    private val webView: WebView, private val ssh: SshConnection,
    private val doubao: DoubaoAsrClient
) {
    enum class State { IDLE, RECORDING, ASR_PENDING, PREVIEW }
    private var state = State.IDLE
    private var currentText: String? = null
    private var recorder: AudioRecord? = null

    fun onKeyDown(keyCode: Int, lang: String) {
        if (keyCode == KEY_F13 || keyCode == KEY_F14) {
            // 任意状态下按 F13/F14 都是"开始(重新)录音"
            if (state == PREVIEW || state == RECORDING) hideOverlay()
            startRecording(lang)
            state = RECORDING
            showOverlay("🎤 录音中...", "")
        }
    }
    fun onKeyUp(keyCode: Int) {
        if ((keyCode == KEY_F13 || keyCode == KEY_F14) && state == RECORDING) {
            val audio = stopRecording()
            state = ASR_PENDING
            showOverlay("识别中...", "")
            scope.launch {
                val text = doubao.recognize(audio)  // suspend
                currentText = text
                state = PREVIEW
                showOverlay("🎤 识别完成", text)
            }
        }
    }
    fun onEnter(): Boolean {  // return true = 拦截,false = 透传到 WebView
        if (state == PREVIEW) {
            val text = currentText ?: return false
            ssh.outputStream().write(text.toByteArray()); ssh.outputStream().flush()
            hideOverlay(); state = IDLE; currentText = null
            return true
        }
        return false
    }
    fun onEsc(): Boolean {
        if (state == PREVIEW || state == RECORDING || state == ASR_PENDING) {
            stopRecording(); hideOverlay(); state = IDLE; currentText = null
            return true
        }
        return false
    }

    private fun showOverlay(status: String, text: String) {
        val s = JSONObject.quote(status); val t = JSONObject.quote(text)
        runOnUiThread { webView.evaluateJavascript("showOverlay($s, $t)", null) }
    }
    private fun hideOverlay() {
        runOnUiThread { webView.evaluateJavascript("hideOverlay()", null) }
    }

    companion object {
        const val KEY_F13 = 326  // KEYCODE_F13 (API 36 公开常量,raw int 跨版本可用)
        const val KEY_F14 = 327
    }
}
```

### 3.5 按键事件路由(Activity 顶层)

`Activity.dispatchKeyEvent` 全局拦截 — F13/F14 路由到 Voice Daemon,Enter/Esc 按 overlay 状态决定:

```kotlin
override fun dispatchKeyEvent(event: KeyEvent): Boolean {
    when (event.keyCode) {
        VoiceDaemon.KEY_F13 -> {
            if (event.action == KeyEvent.ACTION_DOWN) voiceDaemon.onKeyDown(event.keyCode, "zh")
            else if (event.action == KeyEvent.ACTION_UP) voiceDaemon.onKeyUp(event.keyCode)
            return true  // 不传给 WebView
        }
        VoiceDaemon.KEY_F14 -> {
            if (event.action == KeyEvent.ACTION_DOWN) voiceDaemon.onKeyDown(event.keyCode, "en")
            else if (event.action == KeyEvent.ACTION_UP) voiceDaemon.onKeyUp(event.keyCode)
            return true
        }
        KeyEvent.KEYCODE_ENTER -> {
            if (event.action == KeyEvent.ACTION_DOWN && voiceDaemon.onEnter()) return true
            // overlay 不显示时透传到 WebView(xterm.js 处理)
        }
        KeyEvent.KEYCODE_ESCAPE -> {
            if (event.action == KeyEvent.ACTION_DOWN && voiceDaemon.onEsc()) return true
        }
    }
    return super.dispatchKeyEvent(event)
}
```

**新增物理键只有 F13/F14 两个**;Enter/Esc 复用并按 overlay 状态智能路由。

---

## 4. Stage A:3 个实验,1 周决定 80% 架构风险

每个实验失败,本节都给出 named fallback。Stage A 全过才进 Stage B(MVP)。

### A.1(1 天)8BitDo Micro F13/F14 keycode 实战

**做什么**:
1. 用 [8BitDo Ultimate Software](https://app.8bitdo.com/Ultimate-Software-V2/) 把 8BitDo Micro 两个按键配成 F13、F14(官方已声明支持 F13-F24)
2. Beam Pro 蓝牙配对 8BitDo Micro
3. 写一个空 Android Studio 项目,Activity 里:
   ```kotlin
   override fun dispatchKeyEvent(event: KeyEvent): Boolean {
       Log.d("KEY", "keyCode=${event.keyCode} action=${event.action} scanCode=${event.scanCode}")
       return super.dispatchKeyEvent(event)
   }
   ```
4. 按键,看日志:是否收到 `keyCode=326` / `keyCode=327`

**Pass**:Beam Pro 上 Android 14 收到 keyCode 326/327 → 主路径成立
**Fail**:收到其他 keyCode 或 0(UNKNOWN)→ Fallback:
- 把 8BitDo 配成 Ctrl+Alt+1 / Ctrl+Alt+2(组合键 8BitDo 必支持,Android 必收到)
- VoiceDaemon 改成检测 Ctrl+Alt+1/2 组合,语义不变

### A.2(2 天)sshj on Android 实战

**做什么**:
1. 空 Android Studio 项目 + 加 sshj 0.39+ 依赖
2. 准备好云端服务器,authorized_keys 加你的公钥
3. Activity 里 `SSHClient → connect → authPublickey → startSession → allocatePTY → startShell`
4. UI 上一个 EditText + Button,按钮触发 `shell.outputStream.write("ls -la\n")`
5. 读 `shell.inputStream` 输出到 TextView,看是否能跑通 `ls / vim / Ctrl+C`(Ctrl+C 用 `write(0x03)`)

**Pass**:基本 shell 命令、PTY、Ctrl+C 都通 → 主路径成立
**Fail**:BouncyCastle 异常 / SSH 握手失败 / PTY 不响应 → Fallback:
- 优先 [sshlib (ConnectBot)](https://github.com/connectbot/sshlib) — 专为 Android 维护,无 BouncyCastle 问题
- 其次 Apache MINA SSHD client

### A.3(2 天)全栈端到端

**做什么**:把 A.1 + A.2 合并,加 WebView + xterm.js + JSBridge:

1. WebView 加载 `assets/terminal.html`(§3.1 那个骨架)
2. JSBridge 双向(§3.3 那个骨架)
3. Activity 启动时 SSH 连接,把 inputStream 推 WebView,把 WebView onData 写 outputStream
4. 实测:在 WebView 里按键,远端 shell 收到字符,执行 `ls / vim / less large.log`,看输出是否流畅渲染、滚动是否正常、Ctrl+C 是否中断、resize 是否同步

**Pass**:**整套架构 proven**,剩下就是堆体验(Voice Daemon、主题、稳定性)
**Fail**:可能的故障点:
- JSBridge Base64 在大输出下卡顿 → 切到 localhost WebSocket(§3.3 备选)
- xterm.js WebGL renderer 在 Beam Pro GPU 上崩 → 退回 Canvas renderer
- WebView 输入法 / 中文 IME 异常 → Activity 拦截 IME 事件 + 手动转发到 onData

---

## 5. Stage B:MVP(1-2 周)

A 全过后:
- 集成 Voice Daemon(§3.4 状态机 + 豆包 ASR client)
- 集成 overlay HTML/CSS(§3.1 那块)
- 实测 F13/F14 录音 → ASR → overlay → Enter 写 SSH 端到端
- Foreground Service + Notification 保活
- 蓝牙 / SSH / WebView 三者断线重连

## 6. Stage C:体验优化(持续)

- xterm.js 主题、字体、行间距 polish
- 接 `docs/03-input.md` §[热词持续优化](03-input.md#热词持续优化)的动态热词表
- xterm.js `search` / `web-links` / `serialize` addon
- 长会话内存优化(scrollback buffer 上限)
- 多服务器配置切换 UI

---

## 7. 关键技术决策

### 7.1 单 app 闭环 vs 多 app 通信

| | 单 app | 多 app(SSH client + Voice Daemon 跨 App) |
|---|---|---|
| 跨 App 文本注入难题 | ❌ 不存在 | ⚠️ docs/06 §1.3.1.1 那一整个章节 |
| 权限(Accessibility / SYSTEM_ALERT_WINDOW)| 不需要 | 需要 |
| 工程量 | 2-3 周 | 1-2 周 + 永久不确定性 |
| 长期可控性 | 高 | 低(依赖外部 SSH client API)|

**选 单 app**。

### 7.2 WebView + xterm.js vs 原生 Compose terminal

| | WebView + xterm.js | 原生 Compose terminal |
|---|---|---|
| UI 现代化天花板 | 极高(web 技术栈)| 中(Compose 还在演进)|
| terminal 模拟成熟度 | 极高(xterm.js 多年沉淀)| 几乎要自造 |
| 工程量 | 低(xterm.js 即拿即用)| 高(月级)|
| 性能 | WebGL renderer 接近原生 | 原生 |

**选 WebView + xterm.js**。

### 7.3 sshj vs Apache MINA SSHD vs sshlib

| | sshj | Apache MINA SSHD | sshlib (ConnectBot) |
|---|---|---|---|
| API 现代度 | 高 | 中 | 低(老 API)|
| Android 兼容历史 | 有 BouncyCastle 问题(已修)| 类似 | **零依赖,稳**|
| 代码量 | 中 | 大 | 中 |
| 维护活跃度 | 高 | 高 | 中 |

**主推 sshj,fallback sshlib**(Stage A.2 决定)。

### 7.4 Base64 JSBridge vs localhost WebSocket

主推 Base64(更简单),Stage A.3 实测如果 60fps 大输出卡顿,30 行代码切到 localhost WebSocket。

### 7.5 Overlay = HTML 元素 vs SYSTEM_ALERT_WINDOW

**HTML 元素 — 关键简化**。docs/06 旧设计需要 `SYSTEM_ALERT_WINDOW` 权限是因为 overlay 要浮在"其他 App(Termius)"之上;本设计 overlay 就在自己的 WebView 里,只是个 `<div>`,**零权限,零跨 App,零问题**。

### 7.6 Voice → SSH 直写 vs Voice → xterm.js

**直写 SSH**。Voice Daemon 拿到 ASR 文本,直接 `ssh.outputStream.write(text)`,字符走 SSH 到远端 shell,shell echo 回送,xterm.js 渲染。**Voice 路径不需要知道 xterm.js 存在** — 少一个集成点。

---

## 8. 失败模式与降级

| 故障 | 检测 | 体感 | 恢复 |
|---|---|---|---|
| 豆包 ASR 错(高频)| overlay 文本明显不对 | 一眼看清 | **按 Esc 撤销** — SSH 永不收到错的字符 |
| 豆包 API 失败 | HTTP 非 200 | overlay 显示"ASR 失败" | F13 重试 / Esc 取消 |
| SSH 断 | sshj 抛 IOException | xterm.js 显示"Disconnected" | 自动重连 + 提示;tmux 远端 session 不丢 |
| 蓝牙断 | InputDevice 拔出事件 | 按键无响应 | 状态栏 notification + 自动重连 |
| WebView crash | uncaught exception | terminal 空白 | Activity 重建 WebView,SSH 不断 |
| App 整个挂 | — | 都不响应 | **任何 SSH client 都能直连服务器继续工作** — 这是本设计的核心降级承诺 |

**核心承诺**:本 App 是体验增强,不是必需品。挂了用 Termius / Termux 继续干,服务端零变化。

---

## 9. 与现有 docs 的关系

| 文档 | 关系 |
|---|---|
| `docs/03-input.md` | 本设计沿用 03 的硬件方案一(8BitDo Micro)+ 直接复用 03 §热词持续优化机制。03 讨论的"IME 注入 vs AccessibilityService 注入"在本设计下**整段不需要**(同 app 内无注入问题)|
| `docs/06-voice-interaction-execution.md` | 本文是 06 经过多轮迭代收敛后的最终方案。06 的多种变体(双端 Voice Gateway / 剪贴板桥接 / Termux Intent 等)在本设计下被取代;06 仍可读作"未选方案的历史 + Voice Gateway 备选" |
| `docs/04-workflow.md` | 工作流不变 — 还是 SSH + tmux + Claude Code,只是 SSH client 换成自己写的 App |
| `docs/05-roadmap.md` | 建议"近期"重排为本文 Stage A 三个实验 |
| `scripts/` 预制脚本 | 不冲突 — 那些是服务端预制的 HTML 生成器,跟客户端 App 正交 |

---

## 10. 未解决问题(Stage A 必须验)

1. **8BitDo Micro 在 Android 14(Beam Pro)上是否真的发出 keyCode 326/327** — Ultimate Software 官方支持 F13-F24,但 Android 端 InputDevice 是否正确映射需实测(A.1)
2. **sshj 0.39+ 在 Android 14 上的 BouncyCastle 加载** — sshj 官方近期 PR 显著改善,但 Beam Pro 的 NebulaOS 可能有自定义 security provider,需实测(A.2)
3. **WebView 内 xterm.js + WebGL renderer 在 Beam Pro GPU 上的性能** — 大输出(`top` / `htop`)60fps 是否撑得住(A.3)
4. **Beam Pro Foreground Service 长 idle 时是否被系统杀** — NebulaOS 后台策略未知,Stage B 实测,可能要加 partial wakelock / 用户手动加入电池白名单
5. **WebView 中文输入法 / 系统输入法跟 xterm.js 的兼容性** — 实测如果用 Beam Pro 系统输入法在 WebView 里打字,字符是否正确进 SSH(可能需要 onInput 钩 IME composing state)
6. **豆包 ASR 对你日常指令的真实准确率** — 不在 Stage A(已经在 docs/06 讨论过门槛 ≥ 70%),仍是 Stage B 的关键依赖

---

## 11. 依赖项清单

```kotlin
// build.gradle.kts (Module: app)
android {
    compileSdk = 35  // Android 15
    defaultConfig {
        minSdk = 34  // Android 14 (Beam Pro)
        targetSdk = 35
    }
}
dependencies {
    // SSH
    implementation("com.hierynomus:sshj:0.39.0")
    implementation("org.bouncycastle:bcprov-jdk18on:1.78.1")
    implementation("net.i2p.crypto:eddsa:0.3.0")

    // UI
    implementation("androidx.activity:activity-ktx:1.9.0")
    implementation("androidx.webkit:webkit:1.11.0")
    // (Compose 可选,如果 settings 页面用)
    implementation(platform("androidx.compose:compose-bom:2024.05.00"))

    // 豆包 ASR(火山引擎 SDK 或自己用 OkHttp 调 REST API)
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("org.json:json:20240303")

    // Audio
    // AudioRecord 是 Android SDK 自带,Opus 编码可用:
    implementation("io.github.lostromb.concentus:concentus:1.1.1")
}
```

**xterm.js 静态资源**(放 `app/src/main/assets/`):
- `xterm.js` + `xterm.css`(v5.5+)
- `addon-fit.js` v0.10+
- `addon-webgl.js` v0.18+
- `addon-unicode11.js` v0.8+(可选,CJK 宽字符)
- 自定义 `terminal.html` + 主题 CSS

合计 APK 增量大约 **8-12 MB**(xterm.js + addons ~500KB,sshj + BouncyCastle ~3MB,其他依赖 ~5MB)。

---

## 12. 总结一句话

**一个 Android App,WebView 跑 xterm.js 当漂亮 terminal UI,Kotlin 用 sshj 连云端 SSH,同 app 内一个 Voice Daemon 录音→豆包→直接写 SSH outputStream。服务端零增量,只跑你已有的 tmux + Claude Code。Stage A 三个独立实验(1+2+2 天)决定 80% 架构风险。**
