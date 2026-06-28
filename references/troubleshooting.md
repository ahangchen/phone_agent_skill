# 连接故障排查详细指南

## 1. 连接超时

### 症状
```
Connection timed out during banner exchange
```

### 排查步骤

1. **检查 Tailscale 网络层**
```bash
tailscale ping <PHONE_HOST>
# 如果显示 direct connection not established，说明走的是 DERP 中继
```

2. **检查 TCP 端口可达性**
```bash
nc -z -w 10 <PHONE_HOST> 8022
```

3. **检查 SSH 握手**
```bash
ssh -vvv -o ConnectTimeout=20 <PHONE_HOST> -p 8022 "echo ok" 2>&1 | head -30
```

### 常见原因

- **IPQoS 问题**: SSH 默认 QoS 标志在 Tailscale 隧道上会导致超时，必须加 `-o IPQoS=none`
- **DERP 中继延迟波动**: 重试即可
- **Termux 被系统杀/冻结**: 提示用户打开 Termux App 或重启 sshd
- **Tailscale 断开**: 提示用户检查 Tailscale App

## 2. 认证失败

```
Permission denied (publickey)
```

### 解决

确保服务器公钥在手机 `~/.ssh/authorized_keys` 中:
```bash
# 在手机 Termux 上执行
echo "<服务器公钥>" >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

## 3. 存储访问失败

```
Cannot access ~/storage/shared/
```

### 解决

在手机 Termux 上执行:
```bash
termux-setup-storage
# 会弹出权限请求，点击允许
```
