# SSH Port Changer

一个面向 Linux 服务器的 SSH 端口安全修改脚本。重点不是单纯替换 `Port`，而是尽量降低远程修改 SSH 配置时把自己锁在服务器外面的风险。

当前版本：**V3.1**

## 功能

- 自动识别 `ssh.service` / `sshd.service`
- 兼容启用了 `ssh.socket` 的系统
- 修改前自动备份 SSH 配置
- 修改前优先放行新的防火墙端口
- 支持 UFW、firewalld、iptables，并安全识别自定义 nftables
- 支持 SELinux `ssh_port_t`
- 修改后执行 `sshd -t` 配置检查
- 检查 sshd 实际解析出的端口
- 重启后检查新端口是否真正监听
- 修改失败自动恢复 SSH 配置
- `--status` 查看当前状态
- `--cleanup` 确认新端口正常后清理旧防火墙端口
- `--rollback` 恢复上一次修改前的 SSH 配置
- 检测 `netfilter-persistent` / `/etc/iptables/rules.v4`，尽可能持久化 iptables 规则

## 一键安装

```bash
curl -fsSL -o /usr/local/sbin/ssh-port \
  https://raw.githubusercontent.com/treeslins/ssh-port-changer/main/ssh-port
chmod +x /usr/local/sbin/ssh-port
```

之后可以直接使用：

```bash
ssh-port 17247
```

也可以不安装，直接下载到当前目录：

```bash
curl -fsSL -O https://raw.githubusercontent.com/treeslins/ssh-port-changer/main/ssh-port
chmod +x ssh-port
./ssh-port 17247
```

## 命令

修改 SSH 端口：

```bash
ssh-port 17247
```

跳过修改前的确认：

```bash
ssh-port 17247 -y
```

查看状态：

```bash
ssh-port --status
```

确认新端口可以正常登录以后，清理旧端口防火墙规则：

```bash
ssh-port --cleanup
```

恢复上一次修改前的 SSH 配置：

```bash
ssh-port --rollback
```

查看帮助：

```bash
ssh-port --help
```

## 推荐操作流程

假设当前 SSH 端口是 `22`，准备改为 `17247`：

```bash
ssh-port 17247
```

脚本成功完成以后，**不要关闭当前 SSH 窗口**。

另外打开一个终端测试：

```bash
ssh -p 17247 root@服务器IP
```

确认新窗口能够正常登录以后，再执行：

```bash
ssh-port --cleanup
```

这样旧端口的防火墙放行规则才会被清理。

## 回滚

每次修改前，脚本会把 SSH 配置保存到：

```text
/root/ssh-port-changer-backups/
```

最近一次成功修改的信息保存在：

```text
/var/lib/ssh-port-changer/state
```

如果需要恢复上一次修改前的 SSH 配置：

```bash
ssh-port --rollback
```

回滚不会自动删除防火墙规则。这样做是为了尽量避免在远程环境中因为同时修改 SSH 和防火墙而造成失联。

## 防火墙说明

### UFW

脚本可以自动添加和清理端口规则。

### firewalld

脚本会处理活动 zone；如果没有活动 zone，则使用默认 zone。

### iptables

脚本会添加精确的 TCP ACCEPT 规则。如果检测到 `netfilter-persistent` 或 `/etc/iptables/rules.v4`，会尝试持久化规则；否则会提示重启后需要再次确认。

### nftables

独立 nftables 配置自由度很高，可能存在多个 table、base chain、priority 和跳转关系。因此脚本不会猜测应该修改哪个 chain。

如果检测到 nftables 正在过滤流量，而目标端口没有明确的 ACCEPT 规则，脚本会停止并要求管理员先手动放行新端口。这是有意的安全设计。

## SELinux

在启用了 SELinux 的系统上，脚本会尝试把新端口加入 `ssh_port_t`。

如果系统缺少 `semanage`，RHEL / Rocky Linux / AlmaLinux 通常可以安装：

```bash
dnf install policycoreutils-python-utils
```

## 云服务器特别注意

主机上的 SSH 和防火墙配置正确，并不代表公网一定能够访问新端口。

阿里云、腾讯云、AWS、Azure、Google Cloud 等平台通常还有外层安全组或云防火墙。执行脚本前后，请确认云平台已经允许新的 TCP SSH 端口。

脚本无法替你修改云厂商安全组。

## 安全建议

修改远程服务器 SSH 配置始终存在失联风险。建议保留当前已经登录的 SSH 会话，直到你从另一个终端确认新端口可以成功建立 SSH 连接。

如果服务器提供 VNC、串口控制台、云厂商 Web Console 等带外管理方式，重要服务器建议确保这些恢复手段可用后再修改 SSH。

## 支持范围

脚本主要面向使用 OpenSSH Server 的常见 Linux 发行版，包括 Debian、Ubuntu、Rocky Linux、AlmaLinux、RHEL 等。

不同发行版、云镜像、控制面板或自行定制的防火墙/SSH 配置可能存在差异。对于复杂的 nftables、策略路由、容器网络或第三方安全软件环境，请在执行前检查现有规则。

## License

MIT
