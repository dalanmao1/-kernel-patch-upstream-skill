# 实战案例：dw-axi-dmac 降级 apb_regs 误报日志

| 项 | 值 |
|---|---|
| patch | `0001-dmaengine-dw-axi-dmac-demote-apb_regs-warning-to-deb.patch` |
| 分支 | `fix-dw-axi-dmac-apb-log-level`（基于 master `dc0d16773238`） |
| commit | `475b9f57ed35` |
| 文件 | `drivers/dma/dw-axi-dmac/dw-axi-dmac-platform.c:577` |
| 级别 | LOW（日志噪音 + 一致性），无 Fixes / Cc:stable |

## 问题

`dw_axi_dma_set_hw_channel()` 在 `!chip->apb_regs` 时用 `dev_err` 报
"apb_regs not initialized"。但 of_device_id 表 5 个 compatible 只有
`intel,kmb-axi-dma` 有 apb_regs：

| compatible | apb_regs |
|---|---|
| snps,axi-dma-1.01a | ✗ |
| intel,kmb-axi-dma | ✓（唯一） |
| sophgo,cv1800b-axi-dma | ✗ |
| starfive,jh7110-axi-dma | ✗ |
| starfive,jh8100-axi-dma | ✗ |

无 apb_regs 的 SoC 跑 slave/cyclic 会走 set_hw_channel（prep 末尾 + terminate），
每次 dev_err 误报刷屏。

同驱动 `dw_axi_dma_set_byte_halfword()`（:406）对同样情况已是 `dev_dbg`，
set_hw_channel（:577）用 `dev_err` 不一致。

## 修复

`dev_err` → `dev_dbg`，对齐 set_byte_halfword。一行。

## 收件人（get_maintainer）

- Eugeniy Paltsev <Eugeniy.Paltsev@synopsys.com>（原作者）
- Vinod Koul <vkoul@kernel.org>
- Frank Li <Frank.Li@kernel.org>
- dmaengine@vger.kernel.org
- linux-kernel@vger.kernel.org

## 查重

未做（日志降级小 cleanup，重复概率低）。发前可选扫 dmaengine 组 "demote.*apb"。

## 发送命令

```bash
ALL_PROXY=socks5h://127.0.0.1:31080 git send-email \
    0001-dmaengine-dw-axi-dmac-demote-*.patch \
    --to=Eugeniy.Paltsev@synopsys.com \
    --to=vkoul@kernel.org \
    --cc=Frank.Li@kernel.org \
    --cc=dmaengine@vger.kernel.org \
    --cc=linux-kernel@vger.kernel.org
```

## 状态

patch 已生成，待查重/发送。

## 教训

- 同一文件里"相同的检查"若两个函数用了不同日志级别，往往是笔误——对齐到更合理的那个。
- "无某特性"对多数 IP 是常态时，那不是 err 是 dbg。
- of_device_id 表是判断"某特性是否普遍"的快速参照——只有一个 compatible 带某 flag，那它就是特例。
