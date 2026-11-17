---
name: gpu-box-ops
description: "在共享多卡 GPU 机器上做 kernel / 系统级 infra 的工程实践：占卡 lease 礼仪与还卡（含 flock 孤儿进程）、跑完清理残留进程与 /dev/shm、远程迭代回路（引号转义、增量同步、后台跑 + 轮询日志）、以及 CRIU / NCCL / 容器挂载 / 国内网络镜像这些只有在真机上才会撞到的坑。适用于借用他人 GPU 机器、搭远程开发回路、或调试 checkpoint-restore 与多卡通信。"
---

# GPU Box Ops — 在别人的机器上干活

共享 GPU 机器上做 infra，一半时间不是写代码，是在跟环境打架。这里是真机上撞出来的东西。

## 一、占卡礼仪（最高优先级）

共享机器通常跑着「占卡脚本」（burn / controller）把空闲卡占满，防止被别人抢走。

<HARD-GATE>
把占卡脚本停了 → 跑自己的东西 → **跑完立刻把占卡脚本起回来**。
不要「我待会儿还要跑所以先占着」——利用率会被你拖垮。按批次借还：攒够一批测试再借一次。
</HARD-GATE>

### 还卡要杀进程组

`flock <lockfile> sleep infinity` 这种占锁方式会留**孤儿子进程**继续持锁。只 kill 父进程锁不会释放：

```sh
kill -- -$pid      # 杀整个进程组，不是 kill $pid
```

还完之后一定要**独立核对**，不要信脚本的退出码：

```sh
nvidia-smi --query-gpu=index,memory.used,utilization.gpu --format=csv,noheader
pgrep -af controller.py | wc -l      # 占卡脚本回来了没
```

### 跑完清理自己的残留

测试框架里起的守护进程、共享内存服务经常不随 pytest 退出而死。一次实测留下
**57 个残留进程和 48 GB 的 /dev/shm**。收尾时固定做一遍：

```sh
pkill -f '<your daemon>'; sleep 3; pkill -9 -f '<your daemon>'
rm -rf /tmp/<your prefix>-*
df -h /dev/shm | tail -1
nvidia-smi --query-compute-apps=gpu_uuid,pid,used_memory --format=csv,noheader
```

显存读数有时是陈旧的——以 `--query-compute-apps` 有没有进程为准，不以 `memory.used` 为准。

## 二、远程迭代回路

### 引号转义：直接 base64

从 PowerShell（或任何多层 shell）往 ssh 里塞带引号的命令，必然在某一层被吃掉。
不要逐层转义，**把整段脚本 base64 编码再在远端解**：

```powershell
$b64 = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($script))
ssh $box "echo $b64 | base64 -d | bash"
```

一次搞定，之后再不用想引号。

### 同步：tar 掉 git 认识的文件

比 rsync 省事，而且天然不会把本地临时文件带过去：

```sh
git ls-files -co --exclude-standard | tar -cf /tmp/sync.tar -T -
scp -O -q /tmp/sync.tar $box:/tmp/ && ssh $box "tar -xf /tmp/sync.tar -C $REMOTE"
```

注意：**运维脚本本身不在被测仓库里**，所以 `sync.ps1` 同步代码、运维脚本单独 `scp`。
换了机器或机器重置后，运维脚本会消失——先确认在不在，别直接调。

### 跑长任务：后台 + 轮询日志

不要前台等。会话一断就全没了，而且阻塞住你做别的事：

```sh
nohup bash run.sh > /root/logs/run.log 2>&1 & echo started
# 之后独立去 tail
```

批处理脚本用 `trap ... EXIT INT TERM` **保证还卡**，哪怕中间挂了：

```sh
trap 'bash return_gpus.sh' EXIT INT TERM
bash borrow_gpus.sh
<your tests>
```

## 三、CRIU（进程 checkpoint/restore）

- **版本门禁**：`--network-lock skip` 要 CRIU **3.19+**。老版本只会给一个看不懂的拒绝。
- **rseq**：内核 < 5.13 没有 `PTRACE_GET_RSEQ_CONFIGURATION`，CRIU 拒绝 dump 任何注册了 rseq 的进程——
  也就是**所有**进程。解法是让目标进程带着 `GLIBC_TUNABLES=glibc.pthread.rseq=0` 启动。
- **失败时一定要把 criu 自己的 stderr 带出来。** 它经常只给一个「看日志」+ 一个并不存在的日志路径。
  把 stderr 接出来之后，后面一连串问题才变得可诊断——**这是投入产出比最高的一次改动**。
- **容器里的 `//deleted` 挂载**：Kubernetes configmap 更新会留下内核标记为 `//deleted` 的 bind mount
  （典型是 `/etc/ssh/sshd_config`），CRIU 恢复时直接失败。启动时把这类挂载 detach 掉。
- **CUPTI 是单向门**：进程只要订阅过 CUPTI（`torch.profiler` 带 CUDA activity、ncu、nsys），
  就再也不能被 checkpoint。profile 只能放在最后一个 checkpoint 之后。

## 四、NCCL

### 独立网络命名空间里的三个变量

```sh
NCCL_SOCKET_IFNAME=lo        # netns 里只有 loopback，bond1 之类根本不存在
NCCL_IB_DISABLE=1            # IB 设备映射不是普通文件，CRIU 拒绝 dump
NCCL_CUMEM_HOST_ENABLE=0     # 搬卡后 host VMM IPC 仍按物理 peer ordinal 导入，会 invalid device ordinal
```

第三个最隐蔽：**只在没有 NVLink（退回 host SHM 传输）且发生过 GPU 重映射时才暴露**。
有 NVLink 的机器上整条路不执行，测不出来。

### 基线必须和被测对象用同一套环境变量

对照实验里 plain baseline 如果没套上这些 NCCL 设置，摘要/数值就对不上。
**这时候要修的是基线，不是断言。** 把断言放宽来「让测试过」是在掩盖真问题。

## 五、环境假设会骗你

- **容器里的 torch 可能没装 numpy。** 任何 `.numpy()` 都会在那台机器上挂，而你本地有 numpy，
  所以从没走到 fallback。读张量字节用 `torch.frombuffer` + `.tolist()`，不要经过 numpy。
  顺带：numpy 到现在也没有 `bfloat16` / `float8_*` 的名字，而这正是这个领域在用的 dtype——
  经过 numpy 本来就走不通。
- **不要在挂了网络文件系统的机器上 `find /`。** 会把机器挂死。限定目录或用 `fd`/`rg`。
- **跨平台写文件注意换行。** 仓库没有 `.gitattributes` 时，Windows 上写的文件会把 CRLF 带进去，
  本地测试完全正常，只有 Linux 上 review diff 才扎眼。

## 六、国内网络

| 症状 | 解法 |
|---|---|
| `static.rust-lang.org` 15 KB/s | conda-forge 装 rust（走腾讯/清华镜像） |
| crates.io 23 KB/s | `~/.cargo/config.toml` 指 USTC 镜像；之后整编 ~1.5 min |
| GitHub 推送 `Proxy CONNECT aborted` | 公司代理间歇性挂。**先试直连**——清掉 `HTTPS_PROXY` / `HTTP_PROXY` 往往反而通 |
| pip/uv 慢 | `UV_HTTP_TIMEOUT=600` + 镜像源；`no_proxy` 里把镜像域名排除掉 |

代理那条特别值得记：遇到 `Proxy CONNECT aborted` 不要反复重试，
直接 `$env:HTTPS_PROXY=""` 再试一次，两秒就知道答案。

## 七、收尾检查单

```
- [ ] 占卡脚本已起回来，lease 已释放（kill 进程组，不是 kill 单进程）
- [ ] 自己的残留进程清零（daemon、共享内存服务、临时目录）
- [ ] /dev/shm 回落到借卡前水平
- [ ] nvidia-smi --query-compute-apps 里没有自己的进程
- [ ] 长任务的日志留在约定位置，不是只在终端里
```

## 配套

怎么判断「测够了没有」——静态检查 / 错误注入 / 环境矩阵：见 `infra-verification`。
