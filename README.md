# kernel-patch-upstream

A step-by-step guide for submitting patches to the Linux kernel community.
From a code change to a sent `git send-email`.

## 文件说明

| 文件 | 内容 |
|---|---|
| [`kernel-patch.md`](kernel-patch.md) | 完整 10 步流程，从理解问题到发送邮件 |

## 快速索引

| 步骤 | 关键命令 |
|---|---|
| 编译验证 | `make ARCH=x86 W=1 drivers/xxx/yyy.o` |
| 找 Fixes hash | `git log --oneline drivers/xxx/yyy.c` |
| 找维护者 | `./scripts/get_maintainer.pl patch.patch` |
| checkpatch | `./scripts/checkpatch.pl --strict patch.patch` |
| 生成 patch | `git format-patch -1 -vN -o /tmp/patch-vN` |
| 发送（走代理） | `ALL_PROXY=socks5h://127.0.0.1:31080 git send-email ...` |

## 使用方式

提 patch 前对照 `kernel-patch.md` 逐步操作，或把修改记录和
发送命令告诉 Claude，让它按本 skill 帮你走完流程。

## 实战案例

- **dma-buf fd 泄漏修复（2026-07）**
  - 问题：`fd_install` 在 `copy_to_user` 之前，失败时 fd 泄漏且有竞态
  - 修法：`get_unused_fd_flags` → `copy_to_user` → `dma_buf_fd_install`
  - 经过：v1（被 Christian König 否定）→ v2（被 T.J. Mercier 指出 trace 丢失）→ v3（分两笔，修复完整）
  - 涉及文件：`drivers/dma-buf/dma-buf.c`、`dma-heap.c`、`drivers/misc/fastrpc.c`、`include/linux/dma-buf.h`

## Gmail SMTP 配置

```ini
# ~/.gitconfig
[sendemail]
    smtpserver = smtp.gmail.com
    smtpserverport = 587
    smtpencryption = tls
    smtpuser = yourname@gmail.com
    smtppass = xxxx xxxx xxxx xxxx   # Gmail 应用专用密码
```

网络不通时加 `ALL_PROXY=socks5h://127.0.0.1:<端口>` 前缀。
