# SSH Port Changer

安全、简单的 Linux SSH 端口修改工具。

支持自动检测 SSH 服务和常见防火墙，修改前放行新端口，执行 sshd 配置校验，失败时恢复 SSH 配置，并提供状态检查和旧端口清理功能。

## 使用

```bash
chmod +x ssh-port
./ssh-port 17247
```

无人值守：

```bash
./ssh-port 17247 -y
```

查看状态：

```bash
./ssh-port --status
```

确认新端口能够正常 SSH 登录后清理旧端口：

```bash
./ssh-port --cleanup
```

## 安全提示

修改端口成功后不要立即关闭当前 SSH 会话。请新开终端测试新端口，确认可以登录后再清理旧端口。

云服务器还需要在云平台安全组中放行新的 TCP 端口。
