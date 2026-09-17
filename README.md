# SSH Port Changer

一个面向 Linux 服务器的 SSH 端口安全修改工具。

它不只是修改 `Port`，而是把远程修改 SSH 端口时最容易导致失联的步骤做成一个带保护的流程：环境检测、端口选择、防火墙提示、配置备份、`sshd` 校验、服务重启、监听验证以及回滚。

当前稳定版本：**V5.1**

## V5.1 特点

- 无参数运行进入中文交互式安全向导
- 自动检测当前 SSH 端口、SSH 服务和 `ssh.socket`
- 自动生成 20000-60000 范围内未占用的随机端口
- 支持手动指定端口
- 修改前执行 `sshd -t`
- 修改后使用 `sshd -T` 确认实际生效端口
- 修改前自动备份 SSH 配置
- 重启后确认新端口确实由 SSH 监听
- 任一关键步骤失败时自动恢复 SSH 配置
- 支持 UFW、firewalld、iptables，并保守处理自定义 nftables
- 支持 SELinux `ssh_port_t`
- 对云安全组/上游防火墙进行人工确认保护
- 支持查看状态、回滚和旧端口清理建议
- 保留高级命令行模式，适合自动化使用

## 已实机验证

V5.1 已在 **Debian GNU/Linux 13 (trixie)** 上完成真实远程 SSH 闭环测试：

```text
旧端口 17247
    ↓
修改为 24976
    ↓
新终端通过 24976 实际 SSH 登录成功
    ↓
执行 --rollback
    ↓
恢复 17247
    ↓
17247 再次实际 SSH 登录成功
```

这项测试验证了 Debian 13 + `ssh.service` + `ssh.socket` 未启用环境下的完整主流程。其他发行版和不同 SSH/防火墙组合仍可能存在差异。

## 一键安装

使用 root 用户执行：

```bash
curl -fsSL \
  https://raw.githubusercontent.com/treeslins/ssh-port-changer/main/ssh-port \
  -o /usr/local/sbin/ssh-port

chmod +x /usr/local/sbin/ssh-port
```

检查版本：

```bash
ssh-port --version
```

然后直接运行：

```bash
ssh-port
```

## 推荐：交互式安全向导

不带参数运行：

```bash
ssh-port
```

主菜单：

```text
1) 修改 SSH 端口        [推荐]
2) 查看 SSH 状态
3) 恢复上一次配置
4) 查看旧端口清理建议
5) 退出
```

修改端口时会依次经过：

```text
步骤 1/5  选择新的 SSH 端口
步骤 2/5  防火墙与云端入站检查
步骤 3/5  修改前安全检查
步骤 4/5  最终确认
步骤 5/5  新端口登录验证
```

在云安全组确认步骤，默认选择是“尚未确认，暂停操作”。因此一路按 Enter 不会直接越过这一项安全检查。

## 高级命令行模式

直接修改为指定端口：

```bash
ssh-port 17247
```

跳过脚本自身的修改确认：

```bash
ssh-port 17247 -y
```

注意：`-y` 适合明确了解服务器网络环境的管理员。它不会替你检查或修改云厂商安全组。

查看当前状态：

```bash
ssh-port --status
```

恢复上一次修改前的 SSH 配置：

```bash
ssh-port --rollback
```

查看旧端口防火墙清理建议：

```bash
ssh-port --cleanup
```

查看版本：

```bash
ssh-port --version
```

查看帮助：

```bash
ssh-port --help
```

## 安全操作流程

假设当前 SSH 端口为 `22`，准备修改为 `17247`。

首先，如果这是云服务器，请在云厂商控制台确认 TCP `17247` 已允许入站。

然后运行：

```bash
ssh-port
```

或高级模式：

```bash
ssh-port 17247
```

脚本完成服务端修改后，**不要关闭当前已经登录的 SSH 窗口**。

新开一个终端实际测试：

```bash
ssh -p 17247 root@服务器IP
```

只有在新终端真正登录成功以后，才建议关闭旧会话或清理旧端口的外部放行规则。

## 脚本会做什么

核心执行顺序：

```text
检测环境
  ↓
检查目标端口
  ↓
备份 SSH 配置
  ↓
处理主机防火墙 / SELinux
  ↓
修改 SSH 配置
  ↓
sshd -t
  ↓
sshd -T
  ↓
重启 SSH
  ↓
验证新端口监听
```

如果修改后的配置校验、有效端口、服务重启或监听验证失败，脚本会尝试恢复刚才创建的 SSH 配置备份。

## 备份与回滚

配置备份保存在：

```text
/root/ssh-port-changer-backups/
```

最近一次成功修改的信息保存在：

```text
/var/lib/ssh-port-changer/state
```

回滚：

```bash
ssh-port --rollback
```

为了降低远程失联风险，回滚 SSH 配置时不会自动删除之前新增的防火墙规则或 SELinux 端口映射。

## 防火墙

### UFW

检测到活动 UFW 时，脚本会在修改 SSH 前尝试放行新的 TCP 端口。

### firewalld

脚本会处理活动 zone；如果没有活动 zone，则尝试使用默认 zone。

### iptables

脚本会添加精确的新端口 TCP ACCEPT 规则。如果存在 `netfilter-persistent` 或 `/etc/iptables/rules.v4`，会尝试持久化。

### nftables

独立 nftables 配置可能包含多个 table、base chain、priority 和跳转关系，因此脚本不会猜测应该修改哪个 chain。

如果检测到自定义 nftables 过滤，而没有明确发现目标端口 ACCEPT 规则，脚本会停止并要求管理员先自行确认规则。

### 旧端口规则

`ssh-port --cleanup` 当前提供**清理建议**，不会自动删除旧端口规则。

这是有意的保守设计：脚本无法可靠证明某条已有规则一定由本工具创建，自动删除可能影响管理员原有防火墙策略。

## 云服务器特别注意

服务器本机显示新端口监听正常，并不等于公网一定能连接。

阿里云、腾讯云、AWS、Azure、Google Cloud 等平台可能在服务器外层还有安全组、云防火墙、ACL 或其他入站策略。

交互式向导会要求用户确认这一点，但脚本无法从服务器内部可靠判断所有云平台的外层规则，也不会擅自修改云安全组。

## SELinux

启用 SELinux 时，脚本会尝试把新端口加入 `ssh_port_t`。

如果系统缺少 `semanage`，RHEL / Rocky Linux / AlmaLinux 系列通常可安装：

```bash
dnf install policycoreutils-python-utils
```

## SSH Socket Activation

脚本会检测 `ssh.socket`。如果系统使用 systemd socket activation，会通过自己的 override 文件调整监听端口，并在修改配置后重新加载 systemd。

不同发行版对 OpenSSH socket activation 的默认配置可能不同，建议首次在新发行版使用时保留云控制台/VNC/串口等带外恢复方式。

## 特殊 SSH 配置

如果检测到 `ListenAddress` 显式携带端口，脚本会保守停止，而不是自动重写复杂监听拓扑。

对于大量自定义 `Include`、多端口 SSH、复杂 Match 规则、容器网络、策略路由或第三方安全软件环境，请先人工检查配置。

## 支持范围

目标环境是使用 OpenSSH Server 的常见 Linux 发行版，例如：

- Debian
- Ubuntu
- Rocky Linux
- AlmaLinux
- RHEL 及兼容发行版

目前明确完成完整实机闭环验证的是 Debian 13。列入目标支持范围不代表所有发行版、版本和定制镜像都已经逐一完成实机测试。

## 开发检查

项目使用 GitHub Actions 对脚本执行基础静态检查：

```bash
bash -n ssh-port
shellcheck ssh-port
```

本地也可以运行相同命令。

## 安全建议

远程修改 SSH 始终存在失联风险。

建议：

1. 保持当前 SSH 会话不关闭。
2. 提前确认云安全组/上游防火墙允许新端口。
3. 新开第二个终端实际登录新端口。
4. 确认新端口登录成功后，再关闭旧会话。
5. 重要服务器最好同时具备云控制台、VNC、串口控制台等带外恢复手段。

## License

MIT License。详见 `LICENSE`。
