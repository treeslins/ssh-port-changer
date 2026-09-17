# SSH Port Changer

> 一个更安全地修改 Linux SSH 端口的交互式工具。
>
> Safe, guided SSH port changes for Linux servers.

**V5.1** · Bash · OpenSSH · MIT

[English README](README_EN.md) · 中文脚本 `ssh-port` · English script `ssh-port-en`

---

## 为什么用它

远程修改 SSH 端口，真正危险的不是改一行 `Port`，而是防火墙、配置错误、服务重启或云安全组导致服务器失联。

SSH Port Changer 把这些步骤组合成一个带保护的流程：

**检测环境 → 选择端口 → 检查防火墙 → 自动备份 → 校验配置 → 重启 SSH → 验证监听 → 新终端登录确认**

### V5.1 主要功能

- 5 步中文安全向导，随机推荐 `20000-60000` 未占用端口
- 自动检测 `ssh.service` / `sshd.service` / `ssh.socket`
- 修改前备份，使用 `sshd -t` 与 `sshd -T` 双重校验
- 支持 UFW、firewalld、iptables；保守处理自定义 nftables
- 支持 SELinux `ssh_port_t`
- 关键步骤失败自动恢复 SSH 配置
- 云安全组必须人工确认，默认不会直接越过
- 支持状态查看、回滚和旧端口清理建议

> V5.1 已在 **Debian GNU/Linux 13 (trixie)** 完成 `17247 → 24976 → 实际新端口登录 → rollback → 17247` 的完整远程闭环测试。

---

## 安装

### 中文版

```bash
curl -fsSL https://raw.githubusercontent.com/treeslins/ssh-port-changer/main/ssh-port \
  -o /usr/local/sbin/ssh-port
chmod +x /usr/local/sbin/ssh-port
ssh-port
```

### English Edition

```bash
curl -fsSL https://raw.githubusercontent.com/treeslins/ssh-port-changer/main/ssh-port-en \
  -o /usr/local/sbin/ssh-port
chmod +x /usr/local/sbin/ssh-port
ssh-port
```

建议使用 `root` 运行。

---

## 交互式向导

```text
1) 修改 SSH 端口        [推荐]
2) 查看 SSH 状态
3) 恢复上一次配置
4) 查看旧端口清理建议
5) 退出
```

修改端口会经过：

```text
1/5 选择新端口
2/5 防火墙与云端入站检查
3/5 修改前安全检查
4/5 最终确认
5/5 新端口实际登录验证
```

云端入站确认的默认选项是“暂停操作”，因此一路按 Enter 不会直接越过这项保护。

---

## 命令

| 操作 | 命令 |
| --- | --- |
| 启动安全向导 | `ssh-port` |
| 指定端口 | `ssh-port 17247` |
| 指定端口并跳过脚本确认 | `ssh-port 17247 -y` |
| 查看状态 | `ssh-port --status` |
| 回滚上一次配置 | `ssh-port --rollback` |
| 查看旧端口清理建议 | `ssh-port --cleanup` |
| 查看版本 | `ssh-port --version` |
| 查看帮助 | `ssh-port --help` |

---

## 安全使用

如果是云服务器，**先在安全组/云防火墙中允许新的 TCP 端口**。脚本无法可靠地从服务器内部修改所有云平台的外层入站规则。

修改完成后不要关闭当前 SSH 会话。新开一个终端：

```bash
ssh -p 17247 root@服务器IP
```

确认新端口真实登录成功后，再关闭旧会话或处理旧端口规则。

配置备份位于：

```text
/root/ssh-port-changer-backups/
```

最近一次修改状态位于：

```text
/var/lib/ssh-port-changer/state
```

`--rollback` 为降低失联风险，不会自动删除新增的防火墙或 SELinux 规则；`--cleanup` 目前只提供安全清理建议。

---

## 兼容性

目标支持 Debian、Ubuntu、Rocky Linux、AlmaLinux、RHEL 等常见 OpenSSH Server 环境。目前完整实机闭环验证为 **Debian 13**。

检测到带显式端口的 `ListenAddress` 或复杂自定义 nftables 时，脚本会保守停止，而不是猜测并重写管理员的网络拓扑。

CI 会自动运行：

```bash
bash -n ssh-port
shellcheck ssh-port
bash -n ssh-port-en
shellcheck ssh-port-en
```

---

## License

MIT License · Copyright © 2026 treeslins
