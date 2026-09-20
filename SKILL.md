---
name: roo-connection-recovery
description: 恢复 Canva Linux Devbox 到 Mac Roo daemon 的连接，诊断失效的 socket 软链接、SSH 转发及 Teleport 回调端口不匹配。用于“恢复 Roo 链接/连接”、cannot connect to roo daemon 等问题；仅在用户同时要求时恢复 Kubernetes 登录。
---

# 恢复 Roo 连接

根据已有恢复记录，先验证现有隧道，再修复入口；没有可用隧道时才重建。用户仅要求保存或整理本文时，只处理文档，不执行连接修复或登录。

## 环境与范围

当前环境的 Mac SSH 别名为 `grantzhang-pro-A.coder`，Coder CLI 为 `/usr/local/bin/coder`。其他环境应替换成实际配置。

连接路径：Devbox 的 `~/.roo/daemon.sock` → `/tmp/roo.daemon.*.sock` → Mac SSH 反向转发 → Mac `/var/roo/daemon.sock`。

- Roo daemon 在 Mac 上运行；`roo daemon restart` 应在 Mac 执行。
- Roo 可达、Okta 缓存有效、Teleport 登录有效和 Kubernetes/AWS 可访问需要分别验证。
- Devbox 工具不能启动 Mac 本机进程。需要用户在 Mac 操作时，先完成 Devbox 检查，再提供具体命令并说明执行位置。

## 1. Devbox：检查并复用现有连接

为 Roo 命令分配 TTY，避免把非交互环境中的 `/dev/tty` 错误误判为连接故障。

```bash
ls -l "$HOME/.roo/daemon.sock"
readlink "$HOME/.roo/daemon.sock"
ls -lt /tmp/roo.daemon.*.sock 2>/dev/null
timeout 15s roo daemon info
```

默认入口已经正常时，无需重启 Roo 或切换链接。失败时，优先探测用户刚建立的隧道，再逐个检查其他候选 socket；每个候选最多探测一次。文件存在、存在监听者或修改时间最新都不代表 Roo 可达。

针对一个候选路径执行以下命令；助手应将输入替换为实际查到的路径：

```bash
read -r -p '输入待验证的 Roo socket 完整路径: ' roo_socket
if [ -S "$roo_socket" ] && \
   ROO_DAEMON_SOCKET="$roo_socket" timeout 15s roo daemon info; then
  mkdir -p "$HOME/.roo"
  if [ -e "$HOME/.roo/daemon.sock" ] && [ ! -L "$HOME/.roo/daemon.sock" ]; then
    printf '默认入口不是软链接，保留现场并检查其用途。\n'
  else
    ln -sfn -- "$roo_socket" "$HOME/.roo/daemon.sock"
    timeout 15s roo daemon info
  fi
else
  printf '此路径未通过 Roo 验证，继续检查其他候选或重建隧道。\n'
fi
```

若修改前发现 `ROO_DAEMON_SOCKET` 环境变量已设置，检查它是否覆盖了默认入口；验证时明确区分显式路径与默认路径。

成功标准是命令正常退出，并返回 daemon 的 `Uptime`、`Version` 等状态。只对已验证成功的目标更新软链接；不要添加“每次启动 shell 都选择最新 socket”的自动配置。

## 2. 没有可用 socket：Mac 重建 SSH 隧道

先确定回调端口：需要恢复 Teleport 登录时，使用第 3 节新登录进程的端口；只修复 Roo 时，可使用现有配置中的实际回调端口，并检查 Mac 端是否已被占用。不要照搬历史端口 `43391` 或 `35069`。

请用户在 **Mac 本机**先运行 `roo daemon info`。仅本机 daemon 异常时执行 `roo daemon restart`；需要 Okta 会话且缓存为空时执行 `roo login`。

在 Mac 建立转发，输入已确定的端口：

```bash
printf '输入本次回调端口: '
read -r roo_login_port
roo_socket="/tmp/roo.daemon.$(date +%s).$$.${roo_login_port}.sock"
printf 'Devbox 应验证此 socket：%s\n' "$roo_socket"

ssh -N \
  -o 'ProxyCommand=/usr/local/bin/coder ssh --stdio --hostname-suffix coder %h' \
  -o ExitOnForwardFailure=yes \
  -o StreamLocalBindUnlink=yes \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3 \
  -L "127.0.0.1:${roo_login_port}:127.0.0.1:${roo_login_port}" \
  -R "${roo_socket}:/var/roo/daemon.sock" \
  grantzhang-pro-A.coder
```

确认输入为 `1024–65535` 的整数，且 Mac 本地端口可用。若 SSH 报端口占用，先判断是否已有可复用隧道；不要终止无关进程。

`ssh -N` 成功后持续运行且通常没有输出，应保持窗口开启。用户需要后台运行时可用 `-fNT` 替换 `-N`；两种方式都依赖 Mac 唤醒、联网。SSH 返回成功仍需回到 Devbox，按第 1 节验证打印出的精确 socket 路径并切换入口。

保留六段文件名 `roo.daemon.<时间戳>.<进程号>.<端口>.sock`。本地 infra 实现会从倒数第二段识别 Teleport 回调端口。缺少端口时，Roo 可能正常，但后续 Kubernetes 登录失败。端口数字必须对应真实的 `-L` 转发；重命名 socket 或设置环境变量都不会创建网络转发。

## 3. 仅在需要时恢复 Teleport / Kubernetes

用户同时要求恢复集群访问时，先检查 `tsh status` 和现有 context；有效凭证可直接复用。不要为了检查 Roo 主动退出或刷新共享 Teleport 会话。

需要重新登录且没有可复用回调转发时，在 Devbox 启动：

```bash
tsh login --proxy=live.teleport.p.canva-cloud.com --browser=none
```

保持进程运行，记录本次输出的完整登录链接和端口。让 Mac 使用此端口执行第 2 节的 SSH 命令，再在 Mac 浏览器打开该链接完成认证。助手应保留登录进程 session ID，等待认证完成后读取退出结果。

已有正常 SSH 隧道时，可令新登录进程监听该隧道实际转发的端口：

```bash
read -r -p '输入已确认转发的回调端口: ' roo_login_port
tsh login --proxy=live.teleport.p.canva-cloud.com \
  --bind-addr="127.0.0.1:${roo_login_port}" --browser=none
```

使用新输出的完整链接，不复用旧链接。历史恢复中手动组合 `--callback` 与 `--bind-addr` 曾被拒绝；此流程不需要覆盖 `--callback`。浏览器提示无法访问 localhost 时，核对登录进程仍在运行，以及浏览器端口、Mac `-L` 端口和 Devbox 监听端口一致。

如果用户要求 USW2，认证完成后执行：

```bash
tsh kube login general-aws-usw2-prod-0 \
  --set-context-name=general-aws-usw2-prod-0 \
  --kube-namespace=b-core-cn-prod
kubectl --context general-aws-usw2-prod-0 --request-timeout=20s \
  auth can-i list pods -n b-core-cn-prod
kubectl --context general-aws-usw2-prod-0 --request-timeout=20s \
  get pods -n b-core-cn-prod -l jobset.canva.k8s/submitter=grantzhang
tsh status
```

其他集群使用用户指定的 context、namespace 和账号。权限检查应返回 `yes`，实际查询应正常退出；没有匹配的 Pod 不等于登录失败。

历史上 `infra kube login` 曾先退出共享 Teleport 会话，再因找不到回调端口重登失败，导致其他任务的登录状态一起丢失。有有效会话时优先使用 `tsh kube login` 恢复已授权的 context，并协调并发登录操作。

## 完成与停止条件

- Roo 正常后，报告已验证的 socket 和 daemon 状态；仅修复 Roo 的任务到此结束。
- 若还恢复了 Kubernetes，报告实际验证的集群和当前凭证有效期，不引用历史有效期。
- 全部现有候选都失败且需要 Mac 操作时，给出本机命令并等待用户完成；不要无限重试、删除其他 SSH 会话或重启 Devbox。
- 不将 Roo 修复解释为 AWS/S3 已恢复或 MFA 次数减少；这些需要对应任务的单独验证。

## 历史依据

整理自 2026-09-15 已验证的 Roo 恢复记录。原 Devbox 上的补充材料位于 `/home/coder/work/zhangguiwei/roo-recovery-20260915.md` 和 `/home/coder/work/zhangguiwei/roo-devbox-runbook.md`；本文包含执行所需步骤，不依赖这两个本地文件。原手册中其他人的主机名仅为示例。
