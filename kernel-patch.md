# Skill: 向 Linux 内核提交 Patch

本 skill 描述从有修改想法到发出 `git send-email` 的完整流程。
用户只需提供：
1. **修改记录**（改了什么、为什么改）
2. **git send-email 发送命令**（收件人列表）

---

## 触发条件

用户说"帮我提一个内核 patch"、"帮我准备 upstream patch"、
"帮我改好发给社区"等，即启动本流程。

---

## 第一步：理解问题

从用户提供的修改记录中提取：

- **Bug 描述**：问题现象是什么
- **根本原因**：为什么会出现这个问题
- **影响范围**：哪些文件/驱动/子系统受影响
- **修复思路**：正确的修法是什么

如有疑问先问清楚，不要猜测。

---

## 第二步：检查现有代码

```bash
# 阅读相关源文件，理解上下文
# 确认修复方案是否正确
# 检查是否有类似问题在其他地方
grep -rn "<关键函数/符号>" drivers/ include/ | head -20
```

重点检查：
- 修复是否引入新问题（UAF、竞态、泄漏）
- 是否有其他驱动有同样问题（可一并修复）
- API 变更是否需要同步更新头文件

---

## 第三步：编写代码

按照 Linux 内核编码风格：
- 使用 `ERR_PTR` / `IS_ERR` / `PTR_ERR` 处理指针错误
- 资源获取顺序和释放顺序相反（RAII 原则）
- `fd_install()` 必须是最后一步（point of no return）
- 注释使用 `/* */`，不用 `//`（内核偏好）
- 函数 < 80 列，缩进用 Tab

编译验证：
```bash
# 只编译修改的文件（快速验证）
make ARCH=x86 W=1 drivers/xxx/yyy.o

# 确认 0 error / 0 warning
```

---

## 第四步：确定 Fixes tag

```bash
# 找到引入 bug 的 commit
git log --oneline drivers/xxx/yyy.c | head -20

# 确认具体 commit
git show <hash> --stat
```

`Fixes:` tag 格式：
```
Fixes: <12位hash> ("<完整 commit subject>")
```

---

## 第五步：写 commit message

格式：
```
<子系统>: <驱动>: <一句话描述，不超过 72 字符>

<问题描述段>：说明 bug 的现象和触发条件。

<为什么原有修法不对>（如果适用）：解释为什么简单修法有缺陷。

<修复方案描述>：说明新方案的步骤和原理。

<可复现性说明>（如果适用）：如何触发这个 bug。

[1] https://lore.kernel.org/...（引用相关讨论）

Fixes: <hash> ("<subject>")
Cc: stable@vger.kernel.org
Reviewed-by: <name> <email>（如果已有）
Signed-off-by: <你的名字> <你的邮箱>
```

注意：
- `Signed-off-by` 必须有，表示你遵守 DCO
- `Cc: stable@vger.kernel.org` 表示需要 backport 到稳定版
- `Reviewed-by` 等 tag 要放在 `Signed-off-by` **之前**

---

## 第六步：创建干净的 patch branch

```bash
# 基于上游最新非 merge commit 创建 branch
git log --oneline --no-merges -5
git checkout -b <fix-branch-name> <base-hash>

# 提交（只 add 相关文件，排除无关改动）
git add drivers/xxx/yyy.c include/linux/zzz.h
git commit -s   # -s 自动加 Signed-off-by
```

---

## 第七步：生成 patch 文件

单笔 patch：
```bash
mkdir -p /tmp/patch-vN
git format-patch -1 -vN -o /tmp/patch-vN
```

多笔 patch（series）：
```bash
mkdir -p /tmp/patch-vN
git format-patch -<笔数> -vN -o /tmp/patch-vN --cover-letter
# 然后编辑 v*-0000-cover-letter.patch 填写 Subject 和正文
```

cover letter 内容：
```
Subject: [PATCH vN 0/M] <系列总标题>

<1-2段说明这个系列修了什么、为什么分成多笔>

vK: <上一版链接>

Changes in vN:
 - <相对于上一版的变化>
```

---

## 第八步：checkpatch 检查

```bash
./scripts/checkpatch.pl --strict /tmp/patch-vN/vN-0001-*.patch
```

- `0 errors` 是必须的
- `WARNING` 中 lore.kernel.org URL 超长是已知 false positive，忽略
- 其他 WARNING 和 CHECK 尽量修复

---

## 第九步：确定收件人

```bash
# 用内核脚本自动找维护者
./scripts/get_maintainer.pl /tmp/patch-vN/vN-0001-*.patch
```

输出格式：
```
姓名 <邮箱> (maintainer)      → 加到 --to
邮件列表 <列表地址> (open list) → 加到 --cc
```

常用固定 Cc：
- `stable@vger.kernel.org`（有 Fixes tag 时）
- `linux-kernel@vger.kernel.org`（所有 patch）

---

## 第十步：发送

如果是回复已有讨论线程，使用 `--in-reply-to`：

```bash
# 从邮件头找 Message-ID（格式：<...@...>，去掉尖括号传给参数）
# 例：Message-ID: <abc123@mail.gmail.com>

ALL_PROXY=socks5h://127.0.0.1:31080 git send-email \
    --to="维护者 <email>" \
    --cc="邮件列表 <list>" \
    --cc="stable@vger.kernel.org" \
    --in-reply-to="<上一版或评审邮件的Message-ID>" \
    /tmp/patch-vN/vN-0000-cover-letter.patch \   # 有 series 时加这行
    /tmp/patch-vN/vN-0001-*.patch \
    /tmp/patch-vN/vN-0002-*.patch                # 有第二笔时加这行
```

首次发送（无前序讨论）不加 `--in-reply-to`。

---

## 版本迭代规则

收到 review 意见后：

| 操作 | 命令 |
|---|---|
| 修改代码 | 正常编辑 |
| 更新 commit | `git commit --amend` |
| 添加 Reviewed-by | `git commit --amend`，在 Signed-off-by 前加 |
| 重新生成 patch | `git format-patch -1 -v<N+1> -o /tmp/patch-v<N+1>` |
| cover letter 加 Changes | 编辑 `v*-0000-cover-letter.patch` |
| 回复上一版 | `--in-reply-to` 指向 reviewer 回复你的 Message-ID |

---

## 常见错误速查

| 错误 | 原因 | 修法 |
|---|---|---|
| `Unable to initialize SMTP` | 缺应用专用密码或网络不通 | 配置 `sendemail.smtppass`，加 `ALL_PROXY` |
| `[PATCH v2 v2]` 标题重复 | 同时用了 `--subject-prefix` 和 `-v2` | 只用 `-v2` |
| patch 包含无关文件 | `git add` 了无关改动 | `git reset`，只 add 相关文件 |
| `Fixes:` hash 不对 | 用了近似 hash | `git log --oneline` 找准确 hash |
| 编译报 `implicit declaration` | 用了其他 .c 内部的宏/函数 | 封装成公开函数放头文件 |

---

## 本次实战参考（dma-buf fd 泄漏修复）

- **v1**：提交初版，用 `close_fd` 修复 → Christian König 指出是竞态
- **v2**：改用 `get_unused_fd_flags` + `fd_install` 分离 → T.J. Mercier 指出 trace 丢失
- **v3**：新增 `dma_buf_fd_install()` 保留 trace，拆成两笔（dma-heap + fastrpc）

关键教训：
- `fd_install` 之后不能再有可失败的操作
- `DMA_BUF_TRACE` 是 dma-buf.c 私有宏，外部用封装函数代替
- fastrpc 有相同问题（注释里都承认"没法修"），一并修掉
