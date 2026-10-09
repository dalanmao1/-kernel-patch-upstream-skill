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

- **mmp_pdma sg 长度取错（2026-09）** — 详见 [case-mmp_pdma-sg-len.md](case-mmp_pdma-sg-len.md)
  - 问题：`mmp_pdma_prep_slave_sg` 里 `sg_dma_len(sgl)` 应为 `sg`，`avail` 恒为第一个 sg 长度
  - 修法：`sgl`→`sg`，Fixes 指向 2012 驱动诞生 commit
  - 经过：v1 发出 → Frank Li 指出 Subject 函数名要带 `()` → v2

- **dw-axi-dmac apb_regs 误报降级（2026-09）** — 详见 [case-dw-axi-dmac-apb-log.md](case-dw-axi-dmac-apb-log.md)
  - 问题：无 apb_regs 的 SoC（snps/starfive/sophgo）每次 slave 传输打 `dev_err`
  - 修法：`dev_err`→`dev_dbg`，对齐同文件 `set_byte_halfword`

- **dw-axi-dmac tx_status 不报 DMA_PAUSED（2026-09）** — 详见 [case-dw-axi-dmac-tx-status-paused.md](case-dw-axi-dmac-tx-status-paused.md)
  - 问题：`dma_chan_pause` 设了 `is_paused`，但 `tx_status` 不读它，pause 后仍返回 IN_PROGRESS
  - 修法：补回 `if (is_paused && status==IN_PROGRESS) status=DMA_PAUSED`；git 考古定位为 `8e55444da65c` 重写遗漏

- **dw-axi-dmac 排队 desc 完成后挂死（2026-09）** — 详见 [case-dw-axi-dmac-restart-next.md](case-dw-axi-dmac-restart-next.md)
  - 问题：`333e11bf47fa` 删掉完成后的接力启动，一次 issue 多个 desc 时第一个完成后其余永久挂死
  - 修法：`axi_chan_wait_idle()` 等停稳 + 恢复完成路径启动 + issue_pending 忙时跳过（F/H/G 三合一）
  - 亮点：修正上游"删功能避时序"的药方；编译 W=1 + checkpatch + NNTP 查重全过

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
