# 实战案例：mmp_pdma 修复 sg 长度取错

| 项 | 值 |
|---|---|
| patch | `v2-0001-dmaengine-mmp_pdma-fix-wrong-sg-length-in-mmp_pdm.patch` |
| 分支 | `fix-mmp-pdma-sg-len`（基于 master `dc0d16773238`） |
| commit | v1 `4bb7a9656d5b` → v2 `fd94ccb4ccdd` |
| 文件 | `drivers/dma/mmp_pdma.c:716` |
| Fixes | `c8acd6aa6bed3`（"dmaengine: mmp-pdma support"，2012，驱动诞生即有） |

## 问题

`mmp_pdma_prep_slave_sg()` 里：

```c
for_each_sg(sgl, sg, sg_len, i) {
    addr = sg_dma_address(sg);     /* ✓ 当前 entry */
    avail = sg_dma_len(sgl);       /* ✗ 用了链表头 sgl，恒为第一个 sg 的长度 */
```

`avail` 恒等于第一个 sg 的长度。多 sg 时：当前段比第一段短 → 越界读；长 → 丢尾。
单 sg 或等长分片掩盖了 12 年。

## 修复

`sg_dma_len(sgl)` → `sg_dma_len(sg)`，一字之差。

## 收件人（get_maintainer）

- Vinod Koul <vkoul@kernel.org>
- Frank Li <Frank.Li@kernel.org>
- dmaengine@vger.kernel.org
- linux-kernel@vger.kernel.org

## 查重

NNTP `org.kernel.vger.dmaengine` 组（35581 篇）扫近 3 万篇，106 条涉 mmp_pdma，
无一条涉及 sg_dma_len / sg length / prep_slave_sg。从未有人报告。✓

（查重脚本：`/tmp/nntp_mmp_pdma.py`，用 `xhdr('subject','lo-hi')` 分批 5000 不崩。）

## 时间线

- 2026-09-09：v1 发出（commit `4bb7a9656d5b`）。
- 2026-09-09：Frank Li review："Need () for funciton mmp_pdma_prep_slave_sg()"
  ——标题里函数名要带 `()`（内核约定，方便 `grep "func("` 同时匹配代码和 patch）。
- 2026-09-10：v2 生成（commit `fd94ccb4ccdd`），Subject 加 `()`，加 v2 changelog。

## 发送命令（v2，回复 v1 线程）

```bash
ALL_PROXY=socks5h://127.0.0.1:31080 git send-email \
    v2-0001-dmaengine-mmp_pdma-*.patch \
    --to=vkoul@kernel.org \
    --cc=Frank.Li@kernel.org \
    --cc=dmaengine@vger.kernel.org \
    --cc=linux-kernel@vger.kernel.org \
    --in-reply-to="<v1 的 Message-ID>"
```

`--in-reply-to` 填 v1 邮件的 Message-ID（发件箱邮件头，或 NNTP 反查）。

## 状态

v2 待发（需填 v1 Message-ID）。

## 教训

- `for_each_sg(sgl, sg, ...)` 循环体内一律用 `sg`，`sgl` 只是链表头常量。
- 12 年老 bug 能藏住，是因为触发需要"多 sg 且长度不等长"——单 sg / 等长分片恰好绕过。
- maintainer 对 commit message 里函数名带 `()` 很在意（grep 约定），发前自查。
