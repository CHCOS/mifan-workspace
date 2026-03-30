---
name: linux-ops
description: Linux服务器运维管理。当用户提到Linux运维、服务器管理、系统巡检、性能排查、磁盘管理、网络配置、进程管理、防火墙、SSH、系统安全加固、CentOS/Ubuntu/Debian操作、systemd服务管理、日志分析、负载均衡、Nginx配置、Docker部署时使用。
---

# Linux 服务器运维

## 核心巡检项

执行巡检时，按以下顺序收集信息并输出报告：

### 1. 系统基础信息
```bash
uname -a
cat /etc/os-release
hostname
uptime
```

### 2. 资源使用情况
```bash
# CPU
top -bn1 | head -20
mpstat 1 3 2>/dev/null
cat /proc/cpuinfo | grep "model name" | head -1
nproc

# 内存
free -h
cat /proc/meminfo | grep -E "MemTotal|MemFree|MemAvailable|SwapTotal|SwapFree"

# 磁盘
df -hT
lsblk
fdisk -l 2>/dev/null | head -30
# 大文件查找
find / -type f -size +500M 2>/dev/null | head -20
# inode使用
df -i

# 网络
ss -tulnp
ip addr
ip route
netstat -an | grep ESTABLISHED | wc -l
sar -n DEV 1 3 2>/dev/null
```

### 3. 系统负载
```bash
uptime
cat /proc/loadavg
w
# 查看僵尸进程
ps aux | awk '$8 ~ /Z/ {print}'
# 查看高CPU进程
ps aux --sort=-%cpu | head -10
# 查看高内存进程
ps aux --sort=-%mem | head -10
```

### 4. 安全检查
```bash
# 登录失败
lastb | head -20
last | head -20
# 当前登录
who

# 防火墙
iptables -L -n 2>/dev/null
firewall-cmd --list-all 2>/dev/null
ufw status 2>/dev/null

# SELinux
getenforce 2>/dev/null
sestatus 2>/dev/null

# SSH配置安全检查
grep -E "PermitRootLogin|PasswordAuthentication|Port |PubkeyAuthentication|MaxAuthTries" /etc/ssh/sshd_config

# 可疑定时任务
crontab -l 2>/dev/null
ls -la /etc/cron.d/ 2>/dev/null
```

### 5. 服务状态
```bash
systemctl list-units --type=service --state=running
systemctl --failed
```

### 6. 日志检查
```bash
# 系统日志错误
journalctl -p err --since "24 hours ago" --no-pager | tail -50
grep -iE "error|fatal|fail|critical" /var/log/messages 2>/dev/null | tail -20
grep -iE "error|fatal|fail|critical" /var/log/syslog 2>/dev/null | tail -20
dmesg | grep -iE "error|fail|warn" | tail -20
```

## 常见运维操作

### 性能排查流程
1. `top` / `htop` 确认瓶颈（CPU/内存/IO）
2. CPU高：`top -H -p <pid>` 看线程，`strace -p <pid>` 跟踪
3. 内存高：`pmap -x <pid>`，检查 OOM：`dmesg | grep -i oom`
4. IO高：`iostat -x 1 5`，`iotop`
5. 网络问题：`tcpdump`，`netstat -s`，`ping`，`traceroute`

### 磁盘管理
```bash
# LVM扩容
lvextend -L +10G /dev/mapper/vg-lv_root
xfs_growfs /              # XFS
resize2fs /dev/mapper/vg-lv_root  # ext4

# 挂载
mount /dev/sdb1 /data
echo "/dev/sdb1 /data xfs defaults 0 0" >> /etc/fstab

# 磁盘IO测试
dd if=/dev/zero of=/data/test bs=1M count=1024 oflag=direct
```

### 网络配置
```bash
# 永久IP（CentOS）
nmcli connection modify eth0 ipv4.addresses 192.168.1.100/24
nmcli connection modify eth0 ipv4.gateway 192.168.1.1
nmcli connection modify eth0 ipv4.dns "8.8.8.8"
nmcli connection up eth0

# 永久IP（Ubuntu）
cat > /etc/netplan/01-eth0.yaml <<EOF
network:
  version: 2
  ethernets:
    eth0:
      addresses: [192.168.1.100/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8]
EOF
netplan apply
```

### systemd 服务管理
```bash
systemctl start/stop/restart/status <service>
systemctl enable/disable <service>
journalctl -u <service> -f          # 实时日志
journalctl -u <service> --since "1 hour ago"
```

## 巡检报告模板

输出报告时使用以下格式：

```markdown
# Linux 服务器巡检报告

**主机名:** xxx | **IP:** x.x.x.x | **系统:** CentOS 7.9 | **巡检时间:** 2026-03-30 16:00

## 1. 资源概览
| 指标 | 当前值 | 阈值 | 状态 |
|------|--------|------|------|
| CPU负载 | | < 核数*0.7 | ✅/⚠️ |
| 内存使用率 | | < 85% | ✅/⚠️ |
| 磁盘使用率 | | < 85% | ✅/⚠️ |
| inode使用率 | | < 85% | ✅/⚠️ |

## 2. 安全状态
## 3. 服务状态
## 4. 异常与建议
```

阈值标准：
- CPU负载 > 核数×0.7 或 > 80% → ⚠️
- 内存使用 > 85% → ⚠️
- 磁盘使用 > 85% → ⚠️，> 90% → 🔴
- 僵尸进程 > 0 → ⚠️
- 登录失败 > 10次/天 → ⚠️
