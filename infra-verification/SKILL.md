---
name: infra-verification
description: "写系统/基础设施代码（GPU runtime、checkpoint/restore、分布式通信、存储 I/O）时判断「测够了没有」的方法论。核心论点：端到端跑通只证明顺利路径在这台机器上是对的，证明不了没被执行的分支、只在失败时走的分支、以及换个环境才走到的分支——这三类各需要一种不同手段（静态类型检查 / 错误注入单元测试 / 环境能力探测）。另含 A/B 性能测量的口径纪律与「结论必须带边界」。适用于 kernel 优化、infra PR 自审、以及 review 别人的系统代码。"
---

# Infra Verification — 端到端跑通 ≠ 测过了

systems 代码最常见的自欺是：**一长串端到端测试全绿，于是认为验证做完了。**

端到端只能证明一件事：**在这台机器上，一切顺利时，结果是对的。**

它证明不了三件事，而这三件事各需要一种**完全不同**的手段。只用端到端，这三类 bug 会原样留在代码里，等别人换台机器或者磁盘写满时替你发现。

| 测不到的东西 | 端到端为什么碰不到 | 对应手段 | 成本 |
|---|---|---|---|
| **没被执行到的分支** | 测试数据构造得太「正常」，`None` 那条路永远不走 | 静态类型检查 | 秒级，不要 GPU |
| **执行到了，但只在失败时** | 磁盘满 / 短读 / CRC 错 / `write` 返回 0 —— 端到端里造不出来 | 错误注入单元测试 | 几十行，不要 GPU |
| **换个环境才走到的代码** | 有 NVLink 就不走 host SHM；装了 numpy 就不走 fallback | 环境能力探测 + 显式 skip | 一个探测函数 |

<HARD-GATE>
提交 infra 改动前，三类手段都要有对应动作。「端到端全绿」不构成通过。
尤其注意：**越是你确信不会出错的清理代码、越是你没见过失败的那条 `except`，越是没被测过。**
</HARD-GATE>

## 一、静态检查是用来覆盖「跑不到的代码」的

不是用来挑格式毛病的。它的价值恰恰在于**不需要那条路径被执行**。

**典型形状**：某个字段类型是 `Placement | None`，但你的测试数据全都构造得「放得进去」，于是 `None` 分支一次都没走过。再多端到端测试也发现不了，类型检查一眼看穿。

### 纪律

- **仓库文档里列了什么检查就全跑。** lint 干净 ≠ 通过。检查项经常不在 `AGENTS.md` 里，而在它指向的 `docs/testing.md` 之类的下一层——点进去看。
- **baseline 红的，不等于这个工具没用。** 大仓库的类型检查常年几十上百个错，没人看。这时候要量 **delta**，不量绝对值：

  ```sh
  # 改动前的基线
  git stash && <typechecker> check 2>&1 | tail -1   # 比如 73 errors
  git stash pop && <typechecker> check 2>&1 | tail -1  # 比如 102 errors
  ```

  差值就是你的。**delta 必须为 0**，哪怕仓库本来是红的。
- **跑的时候确认解释器选对了。** 工具挑错 Python 会报一堆 `missing-import`，真信号被淹没（pyrefly 用 `--python-interpreter-path .venv/bin/python`）。
- 修法就是老实加 narrowing：`assert x is not None`、把 `list[T]` 形参放宽成 `Sequence[T]`（list 是不变的）、回调签名用 `Callable[[], object]` 而不是 `Callable[[], None]`（否则有返回值的函数传不进来）。

## 二、错误路径要单元测试 + 注入，不要指望端到端

### 识别信号

**凡是有 `try/except/finally`、重试循环、资源释放（memoryview / mmap / 文件描述符 / 锁）的地方，都有一条测试从没走过的路。** 把它们列出来，一条一条问「这条怎么触发」。

### 两种真实的 bug 形状

1. **清理代码自己抛异常，把原始错误盖掉。** 切出去的 `memoryview` 没释放 → `mmap.close()` 抛 `BufferError` → 真正的 `disk full` / `CRC mismatch` 永远传不出来，上层只看到一个莫名其妙的 BufferError。
2. **边界返回值导致死循环。** `while chunk: chunk = chunk[os.write(fd, chunk):]` —— `os.write` 返回 0 时原地转圈。真实文件系统上几乎不发生，只能注入。

### 做法

不要走全链路。**直接调内部函数，把参数和环境压到触发错误的形状**：

```python
def test_corrupt_segment_reports_the_image_error(tmp_path, monkeypatch):
    monkeypatch.setattr(module, "_CHUNK", 4)        # 逼出多块循环
    path = tmp_path / "segment"; path.write_bytes(b"short")
    with pytest.raises(module.ImageError, match="ended early"):
        module._host_in(fd, 9, expected_crc, path)   # 直接调，不起服务

def test_write_failure_is_preserved(tmp_path, monkeypatch):
    monkeypatch.setattr(os, "write", lambda *a: (_ for _ in ()).throw(OSError("disk full")))
    with pytest.raises(OSError, match="disk full"):
        module._host_out(fd, 7, tmp_path / "segment")
```

三十多行，不要 GPU、不要 root、不要特权，覆盖率比那条跑十分钟的端到端高。

### 反模式

把唯一的「损坏数据」测试写成：起守护进程 → 跑真脚本 → 存快照 → 翻一个字节 → 重启 → 再恢复。这条链路太重，**你没法在里面让 `os.write` 返回 0**，而且它往往只覆盖到其中一条实现路径（比如只走了 device 分支，host 分支一行没测）。

## 三、环境是一个测试维度，不是背景

对 infra 来说「在我机器上过了」信息量很低。先列出：**这段代码的行为依赖哪些环境事实？**

真实例子：GPU 之间有 NVLink 时 NCCL 挑 P2P 传输，没有时退回 host 共享内存。于是整条 host 传输路径在一台全 NVLink 的机器上**结构性地不执行**——不是漏了用例，是那段代码不跑。换到 PCIe 机器上立刻炸。

更讽刺的是：如果你专门验证了「带 NVLink 能工作」并写进文档，等于把测试钉死在最不容易暴露问题的配置上。

### 纪律

- **测试依赖某个环境能力时，必须显式探测并 skip，不要默认有。**

  ```python
  if not (_nvlinked(a, b) and _nvlinked(c, d)):
      pytest.skip("needs NVLink-connected source and destination GPUs")
  ```

  探测源：`nvidia-smi topo -m`（互联拓扑）、`nvidia-smi --query-gpu`（卡数/型号）、`uname -r`（内核特性）、`shutil.which`（外部工具）、`importlib.util.find_spec`（可选依赖）。
- **写下你的环境矩阵，并明说没覆盖哪些。** "8×B200 / driver 580 / NCCL 2.30 / kernel 5.4，单次测量" 比 "测过了" 有用得多。
- **可选依赖的 fallback 路径要专门测。** 容器里的 torch 可能根本没装 numpy——任何 `.numpy()` 都会挂，而你本地有 numpy 所以从没走到 fallback。

## 四、结论必须带边界，负面结果照写

性能/算法结论尤其容易被无意识地做成「对自己有利」。

**真实例子**：直觉上 bisect 找首个坏点比线性 scan 快。实测 60 条指令、故障在第 37 条：scan 跑 38 条指令 / 38 次检查 / 0.059 s，bisect 只要 7 次检查但**每次探测都要重跑前缀**，共 280 条指令 / 0.083 s。交叉点是「一次检查贵过 7.8 条指令」。

正确做法是**把交叉点算出来写进文档、默认改成 scan**，而不是把 demo 调成对 bisect 有利的规模。

### 纪律

- 每份报告有独立的**「边界」**一节，写清楚**没测什么**、哪些数字是单次测量、哪些是推断。
- 发现结论和预期相反时，这是最有价值的结果，不是失败。照写。
- 不要事后调参数/改形状让 demo 好看。

## 五、A/B 性能测量的口径

- **从同一状态出发。** 每轮都要把被测对象的输入复位，否则原地修改输入的实现会越跑越偏。能原地复位（而不是重建进程）时快两三个数量级，这是让「一个输入跑一百遍不同实现」可行的前提。
- **冷 L2、变体交错、用设备事件计时**，不要墙钟、不要顺序跑完 A 再跑 B。
- **对不计时的高精度 oracle 比，不要拿一个近似比另一个近似。** 跨 dtype 时在精度宽的那边做比较、容差取被测那边。
- **真实负载是形状分布，不是一个形状。** 同一个改动在三个 shape 上可以是 11.7× / 6.8× / 3.6×，加权后 8.0×。汇总要**点名表现最差的那个 case**，否则回退会藏在平均数里。
- **亚毫秒 kernel 要把 N 次调用合进一个计时区间再除**，否则测的是 launch 开销。
- **报置信区间**（配对比值中位数的 bootstrap），不报单点。
- **先断言有限性再比较。** 出现 NaN/Inf 时很多「相等」检查会假性通过（`equal_nan=True` 让 NaN == NaN）。真实踩过：准备阶段 20 步就溢出到 inf，之后所有正确性检查全部「通过」。

## 提交前检查单

```
- [ ] 仓库文档列的检查全跑了（不只 lint）；静态检查 delta = 0
- [ ] 新增的每处 try/except/finally、重试循环、资源释放，都有注入测试
- [ ] 测试依赖的环境能力都显式探测 + skip，没有「默认有」
- [ ] 可选依赖的 fallback 路径被走到过
- [ ] 报告有「边界」一节，写了没测什么
- [ ] 与预期相反的结果照写，没有调参让 demo 好看
- [ ] 跨平台写的文件：仓库没有 .gitattributes 时，Windows 会把 CRLF 带进去
- [ ] 基线和被测对象用同一套环境变量（对不上时修基线，不是修断言）
```

## 配套

- 具体的 GPU 机器操作、CRIU/NCCL/驱动层的坑：见 `gpu-box-ops`。
- 一份真实的复盘（三个 bug 分别属于上面哪一类、各自为什么漏掉）：见 [postmortem.md](postmortem.md)。
