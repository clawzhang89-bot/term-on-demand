# 完整工作流

## 日常流（默认场景）

```
你戴着 XREAL 眼镜
  │
  ├── Beam Pro 在手里
  │   └── Termius SSH 连到云端 Ubuntu
  │
  ├── 眼镜里看到终端界面
  │
  └── 操作方式：
      ├── 迷你键盘：敲命令、Ctrl+C/V、Tab 补全、翻历史
      └── 语音键："查找所有超时的连接" → 自动键入终端
```

## 需要看图/看数据时

```
终端里敲：docker stats --format "{{json .}}"

你想"画个实时的容器资源饼图"
  │
  ├── 按英文语音键 → 说"write a python script to poll docker stats and
  │   generate a Chart.js HTML page with real-time container CPU/memory pie charts,
  │   save to /var/www/html/containers.html"
  │
  ├── AI agent 执行：写脚本 → 生成 HTML → 放到 Nginx 目录
  │
  └── 在 Beam Pro 浏览器打开 http://your-server/containers.html
      → 眼镜里看到完整的交互式饼图
      → 看完关掉 tab，回到终端
```

## 需要 AI agent 代操作时

```
按中文语音键 → 说"帮我看看生产环境最近有没有报错"

我（OpenClaw）→ SSH 到你的 Ubuntu
  → 执行 kubectl get events --sort-by='.lastTimestamp'
  → 返回结果给你

如果你想看可视化版本：
  → "把这些事件按时间画成一张图"
  → AI 写 HTML → Nginx serve → 眼镜浏览器打开
```

## 需要走开时

```
你站起来离开电脑
  │
  ├── 眼镜里依然能看到终端（Beam Pro 随身）
  ├── 迷你键盘 + 语音继续操作
  └── AI agent 的审批/通知推到眼镜上
      （参考 cc-g2 的模式处理 Claude Code permission hooks）
```

## 预制脚本的工作流

```bash
# 一键生成当前目录的 HTML 浏览器
ls-html /var/log
# → 生成 /var/www/html/dir.html
# → 眼镜浏览器打开 http://your-server/dir.html

# 系统状态仪表盘
sysinfo
# → 生成 CPU / 内存 / 磁盘 / 网络 一览页面

# 日志查看器
log-view /var/log/nginx/access.log
# → 分页、搜索、关键词高亮的日志页面
```
