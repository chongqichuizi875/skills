# 复盘：三个 bug，端到端全绿，审阅者一个下午全找出来了

一次真实的 PR。给一个 CRIU + GPU 显存镜像的进程快照/重放项目加多卡支持和一套 kernel 迭代工具，
21 个 commit、71 个文件、+9256 行。自测结果：

- 主套件 337 passed / 4 skipped
- 自己新写的多卡 e2e 11 passed
- 上游 megakernel e2e 14 passed
- `ruff check` 和 `ruff format --check` 干净

上游作者合并前加了 6 个实质 commit，其中三个是**我代码里的真 bug**。下面是每个 bug 属于哪一类、
为什么我的测试结构性地碰不到、以及他用什么手段抓到的。

---

## Bug 1 — 缺一个环境变量，只在另一种互联拓扑上暴露

**症状**：多卡组搬到别的 GPU 之后，NCCL 的 host VMM IPC 导入仍按**物理** peer ordinal 去找，
而 CUDA 已经重映射了恢复进程的 device ordinal，于是 communicator warmup 阶段报
`invalid device ordinal`。

**修法**：加 `NCCL_CUMEM_HOST_ENABLE=0`，改走可移植的 POSIX SHM 路径。

**我为什么没碰到**：我的机器 GPU 之间全是 NVLink，NCCL 给 rank 间通信挑 P2P/CUMEM，
**host IPC 那条路根本不执行**。他的机器没有 NVLink（从他同时加了 NVLink 拓扑门禁反推），
退回 host SHM 传输，一跑就炸。

不是我漏了用例——是那段代码在我的机器上不跑。

**类别**：环境维度。

---

## Bug 2 — 错误路径把真正的错误盖掉了，外加一个死循环

**症状一**：保存/加载镜像段时，我从 `memoryview` 切出来的 chunk 一直活着，
出错时 `mmap` 关不掉，抛出来的是 `BufferError` 而不是原本的 `disk full` / `CRC mismatch` /
`ended early`。上层拿到一个毫无信息量的异常。

**症状二**：`while chunk: chunk = chunk[os.write(fd, chunk):]` —— `os.write` 返回 0 时原地转圈。

**修法**：每个切片用 `with` 管起来；`if not count: raise ImageError(...)`。

**我为什么没碰到**：我的测试全是成功路径。唯一沾边的「损坏镜像」测试用默认的 GPU 镜像，
走的是 device 分支（那条路有自己的 CRC 检查，先一步抛出来了）——**host 分支的错误处理一行测试都没有**。

更根本的是手法：我的测试清一色端到端（要 root + CRIU + 真 GPU），
在那种测试里你没法让 `os.write` 返回 0，也没法造磁盘满。

**他怎么抓到的**：写了 36 行单元测试，`monkeypatch` 把分块大小调成 4 逼出多块循环、
把 `os.write` 换成抛 `OSError`、喂一个短文件和一个 CRC 不对的文件，**直接调内部的
`_host_in` / `_host_out`**。不要 GPU、不要 root、不要 CRIU。

我当时默认「这个模块要 GPU 才测得了」。其实那两个函数只要一个文件描述符。
**这是我自己画的边界，不是事实。**

**类别**：失败路径。

---

## Bug 3 — 测试假定了一个硬件能力

**症状**：我写的「带 NVLink 的多卡组能 capture 并搬卡」这个测试，在没有 NVLink 的机器上直接失败。

**修法**：加 `_nvlinked()` 读 `nvidia-smi topo -m`，源和目标 GPU 之间没有 NV 链路就 `pytest.skip`。

**我为什么没碰到**：我在有 NVLink 的机器上验证了这个能力，然后把它写成测试，
**没有问「这个测试在没有 NVLink 的机器上会怎样」**。

**类别**：环境维度（和 Bug 1 同一个根因）。

---

## 还有一批类型错误：他是照着检查器输出逐条修的

剩下那批改动（`assert x is not None`、`list[T]` → `Sequence[T]`、
`Callable[[], None]` → `Callable[[], object]`、变量遮蔽、可能未绑定的局部变量）
**全部一一对应检查器的输出**。事后核对：在他动过的那些文件里，`pyrefly check` 报 20 个错，
他的 commit 正好修了这 20 个，一个不多一个不少；连 demo 目录里的 3 个也对得上。

这不是他眼睛毒，是他跑了一条我没跑的命令。

**而这条命令本来就写在仓库文档里**：`AGENTS.md` 说 "See docs/testing.md"，
`docs/testing.md` 写着 `Lint and types: ruff check, ruff format --check, pyrefly check`。
我跑了前两条，漏了第三条。

### 为什么会漏

事后量了一下：**上游 main 的 baseline 本来就有 73 个类型错误**，我那版 102 个。
也就是说这个检查常年是红的，不像一条「我弄绿变红」的门禁线，所以我下意识把
「ruff 干净」当成了通过标准。

但 **102 − 73 = 29 个是我们新增的**，这点不变。
**正确做法是量 delta，不是看绝对值是不是零。**

---

## 一个纯疏忽：CRLF

仓库没有 `.gitattributes`，git 不做换行规范化，我在 Windows 上写的一个测试文件
原样带着 CRLF 进了仓库（329 行）。本地跑测试完全正常，只有在 Linux 上 review diff 才扎眼。

一行 `.gitattributes`（`* text=auto eol=lf`）可以根治。

---

## 结论

不是"没考虑中间数值"——数值那一类我反而抓到了（A/B demo 一直在拿 NaN 比 NaN，
`equal_nan` 让所有检查假性通过，在源头修掉了）。

真正的问题是：**我把「测试」等同于「端到端跑通」。**

端到端只能证明顺利时是对的。它证明不了失败时的行为，证明不了换台机器的行为，
更碰不到那些根本没被执行的分支。补这三个缺口需要的手段，**都不需要 GPU，
成本比那些端到端测试低一个数量级**。
