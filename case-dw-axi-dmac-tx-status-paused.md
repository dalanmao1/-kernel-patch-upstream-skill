# 实战案例：dw-axi-dmac 修复 tx_status 不报 DMA_PAUSED

| 项 | 值 |
|---|---|
| patch | `0001-dmaengine-dw-axi-dmac-report-DMA_PAUSED-in-tx_status.patch` |
| 分支 | `fix-dw-axi-dmac-tx-status-paused`（基于 master `dc0d16773238`） |
| commit | `65cb25c9084e` |
| 文件 | `drivers/dma/dw-axi-dmac/dw-axi-dmac-platform.c:375` |
| Fixes | `8e55444da65c`（"Support burst residue granularity"，2021-01-25） |

## 问题

`dma_chan_pause` 设了 `chan->is_paused = true`，但 `dma_chan_tx_status` 不读它——
`dma_cookie_status()` 只给 COMPLETE/IN_PROGRESS，不知 pause，所以 pause 后客户端永远
拿到 `DMA_IN_PROGRESS` 而非 `DMA_PAUSED`。

## 根因（git 考古）

驱动诞生时（`1fe20f1b8454`）tx_status 本来有：

```c
if (chan->is_paused && ret == DMA_IN_PROGRESS)
    ret = DMA_PAUSED;
```

`8e55444da65c`（"Support burst residue granularity"，Sia Jee Heng, 2021-01-25）
为加 burst residue 报告**重写 tx_status**，重写时把这两行丢了——commit message
的意图是加 residue，不是去 pause，属于**重写遗漏**。

NNTP 查 lore 证据：
- 该系列迭代 v1→v12 共 12 版，v2 的 review（`9566`/`9635`）正文无 `is_paused`/`DMA_PAUSED`——review 没抓出来。
- 最终 v12（`10399`）diff 确认删除 `-	if (chan->is_paused && ret == DMA_IN_PROGRESS) / -		ret = DMA_PAUSED;`。
- 后续 "try best to get residue"/"move tx_status" 重构系列（`18141~19098`）正文也无 is_paused——没人补回。
- 全组标题搜无任何"补 DMA_PAUSED"patch——从未被发现/修复。

## 修复

在 tx_status 的 `spin_lock_irqsave` 之后、residue 计算之前补回：

```c
if (chan->is_paused && status == DMA_IN_PROGRESS)
    status = DMA_PAUSED;
```

放在锁内（is_paused 按 .h 注释受 vc.lock 保护），比原版（锁外读）更严谨。
residue 不受影响：pause 时硬件 suspend，`completed_blocks` 不前进，residue 仍准。

## 收件人（get_maintainer）

- Eugeniy Paltsev <Eugeniy.Paltsev@synopsys.com>（原作者）
- Vinod Koul <vkoul@kernel.org>
- Frank Li <Frank.Li@kernel.org>
- dmaengine@vger.kernel.org
- linux-kernel@vger.kernel.org

## 查重

NNTP（`org.kernel.vger.dmaengine`）扫 dw-axi-dmac pause/residue/granularity 相关
23 条，无补 DMA_PAUSED 的 patch；review 正文确认没人提过。✓

## 发送命令

```bash
ALL_PROXY=socks5h://127.0.0.1:31080 git send-email \
    0001-dmaengine-dw-axi-dmac-report-DMA_PAUSED-in-tx_status.patch \
    --to=Eugeniy.Paltsev@synopsys.com \
    --to=vkoul@kernel.org \
    --cc=Frank.Li@kernel.org \
    --cc=dmaengine@vger.kernel.org \
    --cc=linux-kernel@vger.kernel.org
```

## 状态

patch 已生成，查重通过，待发送。

## 教训

- "重写遗漏"是常见 bug 模式：新逻辑（residue）+ 旧逻辑（pause 映射）本该共存，重写时只搬新的，旧的被无声丢弃。`git log -S <被删的符号>` pickaxe 能直接定位删除 commit。
- review 看大 patch 容易只盯新功能，忽略被替换掉的旧行为——Reviewed-by 多的 patch 也靠不住。
- 一个 commit 迭代 12 版 + 4 个 Reviewed-by/Acked-by 仍可能藏着遗漏，git 考古 + lore 正文是最后防线。
