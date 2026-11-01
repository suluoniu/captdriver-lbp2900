# captdriver — 让 Canon LBP2900 在 Linux + CUPS 下正常工作

这是 [agalakhov/captdriver](https://github.com/agalakhov/captdriver) 的 fork，
带上了让 **Canon LBP2900** 在 Linux 上真正打得出纸所需的修复。

> 实测环境：树莓派 4 / Ubuntu Server 24.04 (arm64) / CUPS 2.4.16，
> LBP2900 走 USB，通过 IPP 共享给 iOS、Android 和鸿蒙客户端。
> 现在每页都能打出来，作业也会正常结束，不会永远卡在"正在打印"。

---

## 问题是什么

LBP2900 是 GDI/CAPT 打印机，自身没有页面描述语言，必须由用户态过滤器
（`rastertocapt`）把每页渲染成位图，再用 Canon 的 CAPT 协议通过 USB 发送，
其中包含一次反向通道（back-channel）握手。

上游给 LBP2900 注册的是**条件式**扩展状态查询：

```c
.get_status = capt_get_xstatus,      /* 只有 XSTATUS_CHNG 置位时才读 0xA0A8 */
.wait_ready = capt_wait_xready,
```

`XSTATUS_CHNG` 是**短状态字**里的一个位。**LBP2900 从不置这一位。** 于是：

1. 扩展状态永远不被读取，`page_decoding` / `page_out` / `page_completed`
   停留在陈旧值；
2. 页尾循环等这些计数收敛 —— 永远等不到，驱动以每秒一次的频率对 `0xA0E0`
   （CHKSTATUS）**无限轮询**；
3. `CAPT_FIRE`（0xE0A7）只在那个等待之后才发，所以打印机收全了整页、也解码了，
   却永远不启动引擎。

外在表现就是：作业进入"打印中"，打印机没动作，队列排不下去，客户端一直显示
"正在打印"。

## 本 fork 的修复

### 1. 始终读取扩展状态（真正的根因）

LBP3010 走的那条路径不依赖 `XSTATUS_CHNG`，所以 LBP2900 也改用这一条：

```c
.get_status = capt_get_xstatus_only,
.wait_ready = capt_wait_xready_only,
```

**这一处改动是"能打出纸"的根本。**

### 2. 用"相对增量"判断页面完成

即使计数变活了，**这台机器仍要等作业结束才结算** `page_out` /
`page_completed` —— 作业运行期间它们不动（实测：发出 `CAPT_FIRE` 后轮询 12 秒，
`pages 3/2/2/2` 一个数都没变，而纸张确实在往外走）。所以上游的等值判断

```c
if (status->page_out == status->page_decoding)
```

在这台机器上永远不成立。本 fork 在每页开始时对三个计数拍快照，把
**"计数越过基线"**当作完成：

```c
lbp_base_out = status->page_out;
...
if (status->page_out > lbp_base_out ||
    status->page_out == status->page_decoding)
        return true;
```

原来的等值判断作为另一分支保留，所以等值判据本来就能用的机型不受影响。

### 3. 轮询间隔 1000 ms → 100 ms

`capt_wait_ready()` 等函数原来用 `sleep(1)`，每次握手白等最多一秒。
现在改为 `usleep(CAPT_POLL_US)`（100 ms），宏定义在 `src/std.h`。

### 4. 所有等待循环加上限

原来都是 `while (1)`，一旦某页状态不收敛就把整个队列挂死。现在每个循环都有
上限（单页 12 秒、作业收尾 4 秒），超时只打警告，不再无限空转。

### 5. 向 CUPS 上报打印机状态

出纸时和作业开始时上报 `STATE: -media-empty`，避免残留的"缺纸"理由挂在客户端
队列窗口里。

---

## 实测数据

单页文本作业，同一台树莓派 4、同一个文件：

| | 修复前 | 修复后 |
|---|---|---|
| 作业完成耗时 | 永不完成（只能取消） | **约 19.5 秒** |
| 仅打上游 + macOS 补丁、未做计数修复时 | 每页 65 秒以上（三个等待循环全部超时） | — |

19.5 秒的构成：渲染约 2 秒、USB 传输约 1 秒、等 LBP2900 机械首页约 12 秒、
收尾约 4 秒。那 12 秒是引擎固有时间，不是软件问题 —— 再压缩会有让整页卡在
打印机内存里的风险（只能断电才清得掉）。

---

## 编译

依赖（Debian/Ubuntu）：

```sh
sudo apt install build-essential libcups2-dev libcupsimage2-dev
```

编译：

```sh
cd src
gcc -O2 -Wall -o rastertocapt *.c $(cups-config --cflags) $(cups-config --image --libs)
sudo install -m 755 rastertocapt /usr/lib/cups/filter/rastertocapt
sudo systemctl restart cups
```

> 注意：参数是 `cups-config --image --libs`，**不是** `--image-libs`。
> 后者不是合法选项，会被当成文件名传给 gcc 导致编译失败。

上游的 autotools 构建（`./autogen.sh && ./configure && make`）同样可用。

## 添加打印队列

```sh
sudo lpadmin -p LBP2900 -E -v usb://Canon/LBP2900 \
     -P /path/to/Canon-LBP2900.ppd -o printer-is-shared=true
sudo cupsenable LBP2900 && sudo cupsaccept LBP2900
```

仓库根目录下的 `Canon-LBP2900.ppd` 就是实测所用的 PPD。

---

## 踩坑记录

这些都是花时间最多的坑，写下来备查：

**quirks 文件是按目录加载的，不是按文件名。** CUPS 的 USB 后端
`load_quirks()` 会读取 `/usr/share/cups/usb/` 下的**每一个**普通文件。如果你在那
目录里留了备份（比如 `org.cups.usb-quirks.bak`），而里面有一行
`0x04a9 0x2676 unidir`，它照样会被加载，反向通道被关掉，报
`CAPT: no reply from printer`。备份要放到这个目录外面。

**取消作业后反向通道会错位。** 打印机在连接时会吐一段 56 字节的状态包；如果上
一个作业在握手中途死掉，下一个作业的首次 6 字节读取会落在包中间，报
`bad reply from printer, expected A1 A1 xx xx xx xx`。那段数据被读完后会自动恢复。

**`usblp` 必须让出设备。** 用 `/etc/modprobe.d/` 屏蔽掉，让 CUPS 的 libusb 后端
独占访问。

**cupsd 会自己退出。** Ubuntu 的 CUPS 带 `IdleExitTimeout 60` 和 socket 激活，
空闲就退出，下一个客户端要付冷启动代价。做打印服务器时设 `IdleExitTimeout 0`。

---

## 致谢

- [agalakhov/captdriver](https://github.com/agalakhov/captdriver) —— 本 fork 的
  上游。
- [duy12i1i7/canon-LBP2900-for-macOS](https://github.com/duy12i1i7/canon-LBP2900-for-macOS)
  —— 其 `patches/lbp2900-macos.patch` 定位了"条件式状态查询"这一根因，并提供了
  `capt_get_xstatus_only` 策略、100 ms 轮询和 `PAGE:`/`STATE:` 上报，本 fork 在
  此基础上继续修了计数收敛问题。
- [mounaiban/captdriver](https://github.com/mounaiban/captdriver) —— 维护更活跃的
  上游。

不想 clone 整个仓库的话，`patches/captdriver-lbp2900-linux.patch` 是相对上游
master 的完整补丁。

## 许可证

GPL-2.0，见 `COPYING`。原始代码版权归 captdriver 作者所有（见 `AUTHORS`）。
