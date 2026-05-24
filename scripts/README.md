# 预制 Skill 脚本

## 使用方式

所有脚本设计为**终端里敲一个命令，生成一个 HTML 文件到 Nginx 目录**。

## 前置条件

```bash
# Nginx 根目录（按你的配置调整）
NGINX_ROOT=/var/www/html

# 确保脚本目录在 PATH 中
export PATH=$PATH:~/term-on-demand/scripts
```

## 脚本列表

| 脚本 | 功能 | 生成文件 |
|------|------|---------|
| `sysinfo` | 系统状态仪表盘 | sysinfo.html |
| `ls-html` | 目录文件浏览器 | dir-*.html |
| `log-view` | 日志查看器 | log-*.html |
| `ps-html` | 进程列表 | procs.html |
| `docker-stats-html` | Docker 容器状态 | docker.html |
| `du-html` | 磁盘用量可视化 | du.html |

## 模板层

| 模板 | 用途 |
|------|------|
| `templates/table.html.sh` | 接受 CSV/stdin 数据，输出可排序+搜索的 HTML 表格 |
| `templates/chart.html.sh` | 接受 JSON 数据，输出 Chart.js 图表页面 |
| `templates/file-tree.html.sh` | 传入目录路径，输出可折叠文件树 |

---

> 注：这些脚本是种子文件。更复杂的场景用 AI 现场生成 HTML。
> 如果某个 AI 生成的页面特别好用，就收进来变成永久预制脚本。
