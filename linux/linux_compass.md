# Linux 核心操作与系统排查指南

Linux 是一套遵循 POSIX 规范的开源类 Unix 操作系统内核。“一切皆文件（Everything is a file）”是其最为核心的设计哲学。无论是普通文本、目录、硬件设备、套接字（Socket）还是进程间通信管道，在系统层面均被抽象为统一的文件描述符（File Descriptor, FD）进行读写交互。

本指南涵盖文件系统、权限机制、文本处理三剑客、进程调度、负载分析与 TCP 网络性能排查核心命令，并结合底层内核机制与性能调优展开剖析。

---

## 1. 目录结构与 Inode 机制

Linux 文件系统采用单根树状层次结构（FHS 规范）。物理磁盘分区格式化为 ext4/xfs 文件系统时，会被划分为两部分：**Inode 区域**（存放元信息）和 **Data Block 区域**（存放实际数据）。

```text
文件系统寻址流程:
文件名 (存在目录项中) ──> Inode 编号 ──> Inode 表项 (权限/大小/时间戳/块指针) ──> 物理磁盘块 (Data Blocks)

```

### 核心路径速查

| 目录路径 | 用途说明 |
| --- | --- |
| `/etc` | 系统的核心配置文件（如 `/etc/passwd`, `/etc/hosts`, `/etc/sysctl.conf`） |
| `/proc` | 虚拟文件系统，反映内核运行状态与进程信息（如 `/proc/cpuinfo`, `/proc/loadavg`） |
| `/dev` | 设备文件映射节点（如 `/dev/sda`, `/dev/null`, `/dev/urandom`） |
| `/var/log` | 系统与应用程序日志目录（如 `messages`, `syslog`, `secure`） |
| `/usr/bin` | 系统预装的绝大多数用户可执行命令 |

### 常用检索与空间排查命令

```bash
# 查看当前目录下文件的 inode 号与详细属性
ls -lai

# 以易读单位（GB/MB）查看各挂载磁盘分区的容量与使用率
df -h

# 查看各分区 Inode 节点消耗情况（排查因海量微小文件导致“磁盘明明有空余却报 No space left on device”的问题）
df -i

# 查看当前目录下各一级子目录所占磁盘空间，并按体积降序排列
du -h --max-depth=1 . | sort -hr

```

---

## 2. 权限体系与属性管控

Linux 对每个文件或目录设置了 3 组针对不同身份的读（r）、写（w）、执行（x）权限：

```text
文件模式标记 (10 位字符示例):
 -    r w x    r - x    r - -
[0]  [1 2 3]  [4 5 6]  [7 8 9]
 ↑      ↑        ↑        ↑
类型  拥有者   所属组    其他用户
     (User)   (Group)   (Others)

```

* 类型位：`-` 表示普通文件，`d` 表示目录，`l` 表示软链接，`c`/`b` 表示字符/块设备。
* 权限权重：`r = 4`, `w = 2`, `x = 1`。

```bash
# 修改权限：给拥有者全部权限，同组读与执行，其他只读 (754)
chmod 754 script.sh

# 递归赋予脚本执行权限
chmod +x ./build.sh

# 修改文件拥有者与所属用户组
chown www-data:www-data /var/www/html/index.html -R

```

---

## 3. 重定向、管道与三剑客实战

标准 I/O 抽象为三个默认文件描述符：

* **`0`**：标准输入（`stdin`）
* **`1`**：标准输出（`stdout`）
* **`2`**：标准错误输出（`stderr`）

```bash
# 输出重定向覆盖 ( > ) 与追加 ( >> )
echo "server_init=1" > config.conf
echo "port=8080" >> config.conf

# 标准输出与错误输出同时重定向到黑洞设备（静默执行）
./backup.sh > /dev/null 2>&1

# 管道（|）：将前一个命令的标准输出作为后一个命令的标准输入
cat /var/log/syslog | grep "ERROR" | wc -l

```

### 文本三剑客：grep, sed, awk

```bash
# 1. grep: 过滤与搜索行
# 递归搜索当前目录下所有包含 "timeout" 的文本文件，显示行号并忽略大小写
grep -rnI "timeout" ./

# 2. sed: 流式文本替换与编辑
# 将 config.txt 中所有 "http://" 替换为 "https://"，-i 参数表示原地写回文件
sed -i 's/http:\/\//https:\/\//g' config.txt

# 3. awk: 结构化文本列提取与统计
# 统计系统中以 /bin/bash 为默认 shell 的用户，并打印其用户名与 UID（以冒号分割）
awk -F: '$7 == "/bin/bash" { print "User: " $1, "UID: " $3 }' /etc/passwd

```

---

## 4. 进程生命周期、Load Average 与性能负载判定

### 进程状态转移模型

* **R（Running / Runnable）**：正在运行或在 CPU 调度就绪队列中的进程。
* **S（Interruptible Sleep）**：可中断睡眠，因等待事件/信号而阻塞。
* **D（Uninterruptible Sleep）**：不可中断睡眠，通常在**等待硬件 I/O 响应**（如磁盘读写）。内核为防止数据不一致，处于此状态的进程**不响应任何外部信号（包括 `kill -9`）**。
* **Z（Zombie）**：僵尸状态，进程执行终止，但父进程尚未调用 `wait()` 回收其状态结构体。
* **T（Stopped）**：暂停状态（如通过 `SIGSTOP` 或 `Ctrl + Z` 挂起）。

### Load Average 核心原理与瓶颈判定

执行 `uptime` 或 `top` 时，会显示 1 分钟、5 分钟、15 分钟的系统平均负载（Load Average）。

在 Linux 内核中，**Load Average 统计的是活跃进程（Active Tasks）的平均数量**：

$$\text{Active Tasks} = \text{Runnable Tasks (R 状态)} + \text{Uninterruptible Tasks (D 状态)}$$

> **与传统 Unix 的本质差异**：Unix 仅统计 R 状态进程，而 Linux 将 **D 状态（等待磁盘 I/O）** 也纳入统计。因此，**Load 高并不等同于 CPU 满载**。

```text
               Load Average 偏高 ( > CPU 逻辑核心数)
                                 │
           ┌─────────────────────┴─────────────────────┐
           ▼                                           ▼
   【CPU 密集型瓶颈】                           【I/O 密集型瓶颈】
* 特征: %usr / %sys 极高, %wa 接近 0          * 特征: %wa (I/O Wait) 极高, %usr 偏低
* vmstat: r (运行队列) >> CPU 核心数          * vmstat: b (阻塞队列) 显著升高
* 状态分布: 大量进程处于 R 状态               * 状态分布: 频繁出现处于 D 状态的进程
* 定位手段: top / pidstat -u 1               * 定位手段: iostat -xz 1 / pidstat -d 1

```

```bash
# 1. 查看 CPU 与 I/O 综合阻塞情况（每秒输出一次，输出 3 次）
# r 列表示就绪进程数，b 列表示等待 I/O 阻塞的进程数，wa 列表示等待 I/O 的 CPU 占比
vmstat 1 3

# 2. 精准定位是哪个进程在疯狂进行磁盘读写（定位 D 状态元凶）
pidstat -d 1 5

# 3. 实时查看磁盘设备的利用率 (%util) 与响应耗时 (await)
# 当 %util 持续接近 100% 且 await 显著飙升时，说明磁盘已达物理瓶颈
iostat -xz 1

```

---

## 5. 网络连接诊断与 TCP 状态调优（TIME_WAIT 治理）

### TCP 核心状态分布与诊断

```bash
# 统计当前系统所有 TCP 连接的状态分布数量
ss -ant | awk 'NR>1 {print $1}' | sort | uniq -c | sort -nr

# 查看所有正在监听的 TCP 端口及对应的进程名称与 PID
ss -tulpn

# 查看本机 8080 端口具体的连接实例与通信对端
lsof -i :8080

```

### TIME_WAIT 过多深度剖析与参数调优

在 TCP 四次挥手过程中，主动发起关闭端（Active Closer）在发送完最后的 ACK 后，必须进入 `TIME_WAIT` 状态并停留 $2 \times \text{MSL}$（Maximum Segment Lifetime，报文最大生存时间，Linux 下默认约为 60 秒）：

```text
主动关闭端                                 被动关闭端
    │                                          │
    ├─────────── FIN (主动挥手) ──────────────>│
    │<────────── ACK (收到确认) ──────────────┤ (CLOSE_WAIT)
    │                                          │
    │<────────── FIN (处理完业务关闭) ────────┤
    ├─────────── ACK (发送确认) ──────────────>│ (LAST_ACK)
    ▼                                          ▼
[TIME_WAIT]                                  [CLOSED]
(持续 2MSL)
    │
    ▼
 [CLOSED]

```

#### TIME_WAIT 存在的两大不可替代意义

1. **可靠实现 TCP 全双工连接的终止**：若主动端发送的最终 ACK 在网络中丢包，被动端会重发 FIN。$2\text{MSL}$ 的等待能保证主动端收到重传的 FIN 并重新补发 ACK，避免被动端因接收不到 ACK 导致异常重置（RST）。
2. **净化网络中迟到的延迟报文**：确保当前连接生命周期内产生的所有旧报文均在网络中彻底消亡，避免与后续使用“相同四元组（源IP、源端口、目的IP、目的端口）”的新连接发生数据混淆。

#### TIME_WAIT 堆积的危害与生产环境调优

当高并发短连接服务（如 Nginx 反向代理、高频爬虫）作为客户端主动断开连接时，大量连接会停留在 `TIME_WAIT` 状态，**直接耗尽本地可用临时端口（Ephemeral Ports）**，导致应用抛出 `bind: Address already in use` 或 `Cannot assign requested address` 错误。

修改 `/etc/sysctl.conf` 进行内核参数治理（执行 `sysctl -p` 立即生效）：

```ini
# 1. 允许将处于 TIME_WAIT 状态的套接字重新用于新的出向连接（极高并发网关必备）
# 依赖 TCP Timestamps 协议头保证序号安全（必须确保 net.ipv4.tcp_timestamps = 1）
net.ipv4.tcp_tw_reuse = 1

# 2. 扩大本地可用端口范围，默认通常只有不到 3 万，建议扩满至 1024~65535
net.ipv4.ip_local_port_range = 1024 65535

# 3. 系统同时保持 TIME_WAIT 套接字的最大上限，超出则内核直接丢弃该状态并输出告警
net.ipv4.tcp_max_tw_buckets = 50000

# 4. 缩短等待 FIN-WAIT-2 的超时时长（默认 60s，可适当缩减降低僵死半连接）
net.ipv4.tcp_fin_timeout = 15

```

> **踩坑警告（关于 `tcp_tw_recycle`）**：
> 在过去的部分老旧博客中常推荐开启 `net.ipv4.tcp_tw_recycle`。在开启 NAT（网络地址转换）的多用户网络中，开启此选项会因不同客户端的时间戳不同步，导致大量连接被内核直接拒绝丢弃。因此**自 Linux 4.12 内核起，`tcp_tw_recycle` 已被官方彻底移除弃用**，切勿在现代 Linux 发行版中使用该参数。

---

## 6. 系统自动化巡检脚本示例

编写一个可直接运行的 Bash 巡检脚本 `check_sys.sh`，整合系统负载判定、网络 TCP 状态统计及核心资源扫描：

```bash
#!/usr/bin/env bash
# 生产环境系统性能与网络健康度快速排查脚本

set -euo pipefail

echo "==================== 系统与网络综合巡检 ===================="

# 1. 负载与瓶颈特征检查
echo "[1] 系统运行时间与平均负载 (Load Average):"
uptime
cores=$(nproc)
echo "CPU 物理/逻辑核心数: ${cores}"
echo ""

# 2. 内存使用排查
echo "[2] 内存与交换分区 (Free / Available):"
free -h
echo ""

# 3. 磁盘空间与 Inode 节点健康度
echo "[3] 根分区容量与 Inode 占用率:"
df -h / | awk 'NR==1 || NR==2 {print $0}'
df -i / | awk 'NR==1 || NR==2 {print $0}'
echo ""

# 4. 识别当前是否存在高 I/O 阻塞 (D 状态) 进程
echo "[4] 处于 D 状态 (Uninterruptible I/O) 的进程扫描:"
d_count=$(ps -eo stat | grep -c 'D' || true)
if [ "${d_count}" -gt 0 ]; then
    echo "警告: 检测到 ${d_count} 个进程处于 D 状态!"
    ps -eo pid,user,stat,cpu,%mem,comm | awk '$3 ~ /D/'
else
    echo "状态正常: 无不可中断 I/O 阻塞进程。"
fi
echo ""

# 5. TCP 连接状态分布统计
echo "[5] TCP 连接状态统计与 TIME_WAIT 监控:"
ss -ant | awk 'NR>1 {print $1}' | sort | uniq -c | sort -nr
echo ""

# 6. 当前正处于监听状态的核心端口
echo "[6] 处于 LISTEN 状态的核心端口:"
ss -tlpn | awk 'NR==1 || /LISTEN/ {printf "%-8s %-25s %-20s\n", $1, $4, $7}'

echo "=========================================================="

```

运行输出：

```text
==================== 系统与网络综合巡检 ====================
[1] 系统运行时间与平均负载 (Load Average):
 10:15:30 up 21 days,  4:18,  3 users,  load average: 3.12, 1.85, 0.94
CPU 物理/逻辑核心数: 4

[2] 内存与交换分区 (Free / Available):
               total        used        free      shared  buff/cache   available
Mem:            15Gi       3.8Gi       7.2Gi       210Mi       4.5Gi        11Gi
Swap:          2.0Gi          0B       2.0Gi

[3] 根分区容量与 Inode 占用率:
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        50G   15G   33G  32% /
Filesystem     Inodes IUsed IFree IUse% Mounted on
/dev/vda1        3.2M  310K  2.9M   10% /

[4] 处于 D 状态 (Uninterruptible I/O) 的进程扫描:
状态正常: 无不可中断 I/O 阻塞进程。

[5] TCP 连接状态统计与 TIME_WAIT 监控:
    1420 TIME-WAIT
     280 ESTAB
      18 LISTEN
       4 CLOSE-WAIT

[6] 处于 LISTEN 状态的核心端口:
State    Local Address:Port        Process             
LISTEN   0.0.0.0:80                users:(("nginx",pid=1024,fd=6))
LISTEN   0.0.0.0:22                users:(("sshd",pid=789,fd=3))
LISTEN   127.0.0.1:3306            users:(("mysqld",pid=982,fd=18))
==========================================================

```

---

### 面试常见问题与底层高频考点

#### 1. 产生大量 CLOSE_WAIT 状态的原因与排查方向？

* **成因**：被动关闭端收到了主动端发来的 FIN 报文，内核自动应答了 ACK，此时连接进入 `CLOSE_WAIT`。然而，**应用程序自身未调用 `socket.close()` 发送最后一个 FIN 报文**。
* **排查方向**：这是**业务应用代码 Bug**。常见原因包括代码逻辑阻塞在耗时操作上无法执行到关闭逻辑、连接池管理泄露、或者捕获异常后漏掉了在 `finally` 块中关闭连接。调优系统内核参数无法根治 `CLOSE_WAIT`，必须修复应用代码。

#### 2. 硬链接（Hard Link）与软链接（Symbolic Link）的本质区别？

* **硬链接**：本质是在目录项中新建一个映射条目，**直接指向现有的 Inode 节点号**。
* 多个硬链接共享同一份数据块与文件权限；
* 删除其中一个文件名，只是将 Inode 的引用计数（Link Count）减 1，直到计数清零才真正释放物理数据块；
* **硬链接不能跨文件系统（跨磁盘分区）创建，且为防止文件系统死循环成环，不能对目录创建**。


* **软链接（符号链接）**：是一个独立的文件，拥有自己**独立的 Inode 编号**。
* 其数据块内容存放的是目标文件的“路径字符串”；
* 原文件被删除后，软链接失效（变为死链/断链）；
* **可以跨文件系统创建，支持直接指向目录**。



#### 3. 孤儿进程（Orphan）与僵尸进程（Zombie）的成因与治理？

* **孤儿进程**：父进程在子进程退出前先终止。
* **治理**：无危害。操作系统会将孤儿进程过继给顶级进程（`systemd` / `init`，PID=1），由 1 号进程负责在其退出时调用 `wait()` 回收资源。


* **僵尸进程**：子进程已终止退出，但父进程未执行 `wait()` / `waitpid()` 读取其退出状态。
* **危害**：大量僵尸进程会持续**占满系统的进程号（PID）资源池**，导致系统无法创建任何新进程（报 `fork: retry: Resource temporarily unavailable`）。
* **排查与解决**：直接对僵尸进程发送 `kill -9` 无效（因为它已经终止）；必须找到其父进程 PID 并将其终止，使僵尸进程过继给 1 号进程后自动被清理。



#### 4. Linux 内存指标中的 Buffer 与 Cache 有何区别？

* **Buffer（缓冲区）**：用于缓释**块设备（原始磁盘块）写操作**的临时内存空间，合并分散的写请求，平滑 I/O 峰值。
* **Cache（页缓存，Page Cache）**：用于缓存**文件系统中的文件内容**。读取文件时预读并缓存，下次读取直接命中物理内存，避免再次触发磁盘物理寻道。
* 当系统物理可用内存不足时，内核的 `kswapd` 守护进程会自动回收无脏数据的 Buffer/Cache。