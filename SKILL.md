---
name: phone-agent
description: "通过 SSH 远程操控 Android 手机（Termux），支持命令执行、文件管理、定时任务和自动化操作。"
---

# Phone Agent

通过 SSH 连接 Android 手机上的 Termux，实现远程命令执行、文件管理和自动化任务。

## 连接方式

```bash
SSH_CMD="ssh -T -o BatchMode=yes -o ConnectTimeout=30 -o IPQoS=none -o StrictHostKeyChecking=accept-new ${PHONE_USER}@${PHONE_HOST} -p ${PHONE_PORT}"
```

### 连接参数

> **注意:** 具体 IP、用户名等敏感信息见本地 `TOOLS.md`，不提交到仓库。

- **主机:** Tailscale IP（见本地配置）
- **端口:** `8022`（Termux sshd 默认端口）
- **用户:** Termux 用户名（见本地配置）
- **网络:** Tailscale，直连约 17-140ms（偶尔走 DERP 中继约 370ms）
- `-T`: 禁用 PTY 分配（提高中继连接稳定性）
- `-o BatchMode=yes`: 禁用交互提示（使用密钥认证）
- `-o ConnectTimeout=30`: 中继延迟高，需要较大超时
- `-o IPQoS=none`: **必须加！** SSH 默认的 QoS 标志在 Tailscale 隧道上会导致 banner exchange 超时

### 注意事项

- **所有 ssh/scp 命令必须包含 `-o IPQoS=none`**，否则在 Tailscale 隧道上会间歇性超时
- Tailscale 可能走直连(~17ms)或 DERP 中继(~370ms)，延迟波动大
- Termux 可能被 Android 系统冻结，连接失败时提示用户打开 Termux 或重启 sshd
- **vivo/OriginOS 需额外设置**: 电池→允许 Termux 后台活动 + i管家权限放行 + 最近任务锁定 Termux
- `termux-wake-lock` + 电池无限制 = 后台可保持 sshd 响应（通知栏会显示 "wake lock held"）
- 认证方式: 公钥认证（服务器 `~/.ssh/id_ed25519` → 手机 `~/.ssh/authorized_keys`）
- 无 root 权限，部分系统级操作不可用（如 `ss`、`iptables`）
- 使用 `exec` 工具时 timeout 建议 45-60 秒
- Termux:API 需从 F-Droid 安装 Termux + Termux:API App（Google Play 版不支持）

## 1. 远程命令执行

```bash
# 单条命令
ssh -T -o BatchMode=yes -o ConnectTimeout=30 u0_a208@100.84.45.49 -p 8022 "command"

# 多条命令
ssh -T -o BatchMode=yes -o ConnectTimeout=30 u0_a208@100.84.45.49 -p 8022 "cmd1 && cmd2 && cmd3"

# 通过 stdin 执行脚本
ssh -T -o BatchMode=yes -o ConnectTimeout=30 u0_a208@100.84.45.49 -p 8022 << 'EOF'
echo "step 1"
echo "step 2"
EOF
```

## 2. 文件管理

### 关键路径

**Termux 内部:**
- `~/` (`/data/data/com.termux/files/home/`) — Termux 主目录
- `~/.ssh/` — SSH 密钥配置
- `~/.bashrc` — 启动脚本（sshd 自启动）
- `~/.termux/boot/` — Termux:Boot 启动脚本

**Android 存储 (需先运行 `termux-setup-storage`):**
- `~/storage/shared/` — 内部存储根目录 (`/sdcard/`)

**相机照片:**
- `~/storage/shared/DCIM/Camera/` — 手机相机拍摄的照片和视频
- `~/storage/dcim/Camera/` — 同上（软链接）

**截图:**
- `~/storage/shared/Pictures/Screenshots/` — 手机截图
- `~/storage/pictures/Screenshots/` — 同上（软链接）

**屏幕录制:**
- `~/storage/shared/Pictures/Screenshots/` — 屏幕录制视频也在截图目录

**其他常见目录:**
- `~/storage/shared/Download/` / `~/storage/downloads/` — 下载目录
- `~/storage/shared/Pictures/` / `~/storage/pictures/` — 图片目录
- `~/storage/shared/Music/` / `~/storage/music/` — 音乐目录
- `~/storage/shared/Documents/` — 文档目录
- `~/storage/shared/Movies/` — 电影/视频目录
- `~/storage/shared/Recordings/` — 录音目录
- `~/storage/shared/Android/data/` — 应用数据目录

**微信/QQ 文件 (常见):**
- `~/storage/shared/Tencent/QQfile_recv/` — QQ 接收的文件
- `~/storage/shared/Android/data/com.tencent.mm/MicroMsg/` — 微信数据

### 文件浏览

```bash
ssh ... "ls -la ~/storage/dcim/Camera/"
ssh ... "find ~/storage/shared/ -name '*.jpg' -mtime -7"
ssh ... "du -sh ~/storage/dcim/Camera/"
```

### 文件传输

**下载（手机 → 服务器）:**
```bash
scp -P 8022 -o ConnectTimeout=30 u0_a208@100.84.45.49:~/storage/dcim/Camera/IMG_001.jpg /tmp/
```

**上传（服务器 → 手机）:**
```bash
scp -P 8022 -o ConnectTimeout=30 /tmp/file.txt u0_a208@100.84.45.49:~/
```

**批量同步 (rsync):**
```bash
# Termux 需先安装: pkg install rsync
rsync -avz -e "ssh -p 8022 -o ConnectTimeout=30" \
  u0_a208@100.84.45.49:~/storage/dcim/Camera/ \
  /tmp/phone-photos/
```

## 3. Termux API 工具

需安装 Termux:API App + `pkg install termux-api`：

- `termux-battery-status` — 电池信息
- `termux-wifi-connectioninfo` — WiFi 信息
- `termux-notification --title "Hello" --content "From server"` — 发送通知
- `termux-vibrate -d 500` — 振动
- `termux-clipboard-get` / `termux-clipboard-set "text"` — 剪贴板
- `termux-location` — GPS 位置（需定位权限）
- `termux-toast "Message"` — Toast 提示
- `termux-open-url "https://example.com"` — 打开 URL
- `termux-open ~/storage/shared/document.pdf` — 打开文件

## 4. 定时任务与自动化

### 服务器端调度（推荐，更稳定）

用 OpenClaw 的 `cron` 工具配置定时任务，通过 SSH 远程执行。服务器 7×24 稳定运行，不受 Android 省电影响。

### Termux 端 cron（手机主动执行）

```bash
pkg install cronie
crond
crontab -e
```

### 保活机制

- `termux-wake-lock` — 防止系统休眠杀后台
- `~/.bashrc` 自动启动 sshd:

```bash
if ! pgrep -x sshd > /dev/null 2>&1; then
    termux-wake-lock
    sshd
fi
```

- **Termux:Boot** — 开机自启动（需安装 Termux:Boot App，与 Termux 同源安装）
- 关闭 Termux 的电池优化

## 5. 常用操作示例

```bash
# 列出最近 7 天的照片
ssh ... "find ~/storage/dcim/Camera/ -name '*.jpg' -mtime -7 -exec ls -lh {} \;"

# 存储空间检查
ssh ... "df -h ~/storage/shared/"

# 系统信息
ssh ... "uname -a && uptime && free -h"
```

## 故障排查

- **Connection timed out / Banner exchange timeout** — DERP 中继延迟高或 Termux 被杀。重试；提示用户打开 Termux 或执行 `pkill sshd && sshd`
- **Permission denied (publickey)** — 检查手机 `~/.ssh/authorized_keys` 是否包含服务器公钥
- **Cannot access ~/storage/** — 在手机上执行 `termux-setup-storage`
- **`ss`/`netstat` 权限不足** — Termux 无 root，改用 `pgrep` 或 `netstat`（不带 -p）

详细排查步骤见 `references/troubleshooting.md`。

## 设备信息

- **设备:** Android (Linux 6.6.89, aarch64)
- **网络:** Tailscale（具体 IP/设备信息见本地 `TOOLS.md`）
