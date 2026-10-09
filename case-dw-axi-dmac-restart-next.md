# 实战案例：dw-axi-dmac 修复排队 desc 完成后挂死（Bug F/H/G 三合一）

| 项 | 值 |
|---|---|
| patch | `0001-dmaengine-dw-axi-dmac-restart-the-next-queued-transf.patch` |
| 分支 | `fix-dw-axi-dmac-restart-next`（基于 master `dc0d16773238`） |
| commit | `270c1dce3361` |
| 文件 | `drivers/dma/dw-axi-dmac/dw-axi-dmac-platform.c`（+30/-1，五处） |
| Fixes | `333e11bf47fa`（"Avoid hw_desc array overrun in dw-axi-dmac"，2024） |

## Bug F（严重）：非 cyclic 完成后排队 desc 永久挂死

`axi_chan_start_first_queued` 全文件只有 2 个调用点（issue_pending / handle_err），
**正常完成路径没有**。客户端一次 issue 多个 desc（DMA engine 合法用法）：

```
submit desc1 + desc2 + issue_pending
→ desc1 执行完成 → block_xfer_complete 非 cyclic 分支只 list_del + cookie_complete
→ desc2 成为 issued 链表头，但没有任何代码启动它
→ desc2 不在跑 → 无中断 → 无轮询 → callback 永不触发 → 客户端永久等待
```

多数客户端"提交-等回调-再提交"单 desc 节奏，所以藏得住。

## 根因：`333e11bf47fa` 吃错药

该 commit 删除了完成后的 `axi_chan_start_first_queued()`，理由（commit message 原话）：

> started descriptors can be interrupted and transfer ignored due to DMA channel
> not being enabled

观察对（DMA_TRF 中断到达时通道 EN 位可能未清，立刻启动会被 non-idle 检查丢弃），
药方错（删启动而非等停稳）——治好"启动被丢"，制造"永远不启动"。

## 修复（三动作闭环）

1. **`axi_chan_wait_idle()` helper**（新）：手写轮询循环复用
   `axi_chan_is_hw_enable()`（内部已正确处理 32/64 位 + id≥16 布局，天然绕开
   terminate_all 的 Bug B 雷区），锁内 udelay(2)×50 = 100us 上限。
2. **恢复完成路径启动**：非 cyclic 分支 `cookie_complete` 后调
   `axi_chan_start_first_queued()`；开头的 non-idle 分支 disable 后加 wait_idle，
   保证后续启动不再被 non-idle 检查丢弃（= 修 `333e11bf47fa` 遇到的时序本身，Bug H）。
3. **issue_pending 忙时跳过启动**：`vchan_issue_pending() && !axi_chan_is_hw_enable()`
   ——正常排队不再误报 "non-idle!" err（Bug G），接力交给动作 2。

## 验证

- `make W=1` 编译单文件：**0 error / 0 warning**（x86 + CONFIG_OF/DW_AXI_DMAC）。
- `checkpatch.pl --strict`：**0 errors, 0 warnings, 0 checks**。
  ⚠️ 踩坑：commit 引用标题必须**逐字匹配 git 原标题且不折行**——
  `333e11bf47fa` 实际标题无 `dmaengine: dw-axi-dmac:` 前缀（当年作者没按规范写），
  写成带前缀+折行会报 COMMIT_MESSAGE_STYLE 错误。
- 板端待验：①dmatest 回归；②submit 3 个 memcpy + issue 一次，修前只 1 个 callback、
  修后 3 个全完成；③忙通道 issue_pending 不再打 "non-idle!"。

## 查重

NNTP `org.kernel.vger.dmaengine` 近 2 万篇扫 dw-axi-dmac ×
(queue/restart/next/complet/hang/stuck/starv/pending/issue)，6 条命中全为
其它主题（轮询时长/缩进/PM），无同类修复。✓
（注：全量 3.6 万篇单次 280s 超时，近 2 万篇足够——老 patch 若存在且未合入恰证明可发。）

## 收件人（get_maintainer）

- Eugeniy Paltsev <Eugeniy.Paltsev@synopsys.com>
- Vinod Koul <vkoul@kernel.org>
- Frank Li <Frank.Li@kernel.org>
- dmaengine@vger.kernel.org
- linux-kernel@vger.kernel.org
- stable@vger.kernel.org（有 Fixes）

## 发送命令

```bash
cd /home/d-robotics/myworkspace/myrepo/kernel_patch_send_skill
ALL_PROXY=socks5h://127.0.0.1:31080 git send-email \
    ./0001-dmaengine-dw-axi-dmac-restart-the-next-queued-transf.patch \
    --to=Eugeniy.Paltsev@synopsys.com \
    --to=vkoul@kernel.org \
    --cc=Frank.Li@kernel.org \
    --cc=dmaengine@vger.kernel.org \
    --cc=linux-kernel@vger.kernel.org \
    --cc=stable@vger.kernel.org
```

## 状态

patch 已生成、编译/checkpatch/查重全过，**待板端验证后发送**。

## 教训

- 上游"删功能避时序问题"的 commit 要审视删除的代价——本例删掉了排队接力，
  单 desc 客户端掩盖了 2 年。
- 修时序问题的正确姿势是"等硬件到稳态再操作"（wait idle），不是绕开操作。
- checkpatch 对 commit 引用的标题查得极严：逐字、单行、以 git log 实际标题为准。
