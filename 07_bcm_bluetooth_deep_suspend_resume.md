# 07. Linux BCM 蓝牙 Deep Suspend/Resume 故障排查：从 serdev、UART 到 HCI 首条命令超时

> 本文记录一次 RK 平台 Broadcom BCM 蓝牙控制器在 Linux `deep` 休眠约 50 秒后唤醒异常的完整排查过程。重点是建立可复现的边界、还原 serdev/UART/HCI 链路，并用二分法把范围缩小到最小可疑层次。
>
> 为适合公开发布，本文隐去了设备内网地址和本地绝对路径；源码行号来自当时使用的 Linux 6.12 目标树，换版本后应以函数名和调用关系为准。

## 摘要

最终得到的最小可疑范围是：

```text
真实 deep 低功耗入口/返回
        ↓
UART 控制器 / serdev 恢复、FIFO、时钟、RTS/CTS
        ↓
hci_bcm 唤醒时序与 HOST_WAKE
        ↓
BCM 控制器返回唤醒后的首条 HCI 命令响应
```

已经有较强证据降低或排除的范围：

- `bluetoothd/BlueZ` 用户空间业务逻辑；
- 蓝牙初始化、固件下载和正常运行时的基本 HCI 通信；
- 仅由 UART runtime PM 策略 `auto/on` 导致的问题；
- VBAT、32 kHz、BT_REG_ON 完全消失这类粗粒度掉电/复位问题；
- 仅仅把 `bcm_resume_device()` 中的 15 ms 固定等待改成 1 秒就能解决的问题；
- `pm_test=core` 所覆盖的基础 PM 回调流程本身。

尚未排除、且当前优先级最高的部分：

1. 真实 deep 期间 UART/IO 的时钟、复位、pinmux、FIFO 或 retention；
2. UART 恢复后 RTS/CTS、BT_WAKE、HOST_WAKE 的握手和时序；
3. BCM 控制器内部唤醒状态、固件状态以及唤醒后的首条 HCI 命令；
4. PM、UART、serdev、hci_bcm 之间的恢复顺序；
5. 物理 UART 信号本身。当前板上没有 UART 测试点，因此仅凭软件日志还不能证明 TX/RX/RTS/CTS 的电气波形正确。

## 1. 先把复现条件固定下来

### 1.1 失败现象

目标板上的蓝牙在正常启动后工作正常。执行真正的 `deep` 休眠，保持休眠约 50 秒，再唤醒，蓝牙有概率进入异常状态。典型现象是：

```text
系统本身能够恢复
    ↓
蓝牙内核设备仍能看到
    ↓
BCM 唤醒后的第一条 HCI 命令没有得到有效响应
    ↓
HCI 命令超时，例如 -110
    ↓
出现 hardware error 0x00，hci0 不能恢复为 UP RUNNING
```

这里的“蓝牙异常”不能只理解成 `bluetoothd` 没有恢复。真正需要观察的是：

- `hci0` 是否回到 `UP RUNNING`；
- `hci_bcm` 是否执行了 resume；
- 第一条 HCI 命令是否从主机发出；
- BCM 是否返回了事件；
- serdev/UART 是否收到了字节；
- HCI parser 是否识别到了完整事件。

### 1.2 最小测试环境

复现不需要蓝牙业务连接，也不需要扫描、配对或音频业务。最小环境只保留：

```text
内核 PM
  + UART/serdev
  + hci_uart/hci_bcm
  + BCM 控制器
  + 真实 deep 休眠/唤醒
  + 唤醒后 HCI 基本状态检查
```

`bluetoothd`、BlueZ 管理接口和业务应用不属于故障链路的必要条件。它们可以用于确认用户态表现，但不应该作为复现的前置条件。

### 1.3 有效的复现实验纪律

最终采用的有效基线是：

- 使用真正的 `deep`，不能只执行 `pm_test=core`；
- 每次休眠保持约 50 秒；
- 每一批至少测试 5 次；
- 每次失败后先把蓝牙恢复到已验证的正常状态，再开始下一次；
- 每次测试前确认 `hci0` 为 `UP RUNNING`；
- 如果一次失败后设备没有完整恢复，立即停止批次，不能把后续失败继续算作独立样本。

早期曾以“连续 2 次 deep 通过”得出结论，后来用 50 秒休眠、5 次测试重新核对，发现该结论不能成立，已经撤回。两次通过只能说明当时两个样本通过，不能称为稳定。

另一次“20 秒休眠 10 次”的批次也不能作为有效成功率：脚本在失败后只重新执行了部分用户态启动动作，没有验证 `hci0` 是否重新回到 `UP RUNNING`，也没有为每次失败做完整的驱动解绑/绑定恢复，导致后续样本被前一次故障污染。因此该批次应判定为无效，而不是 0/10 或某个成功率。

## 2. 休眠流程到底有几个阶段

Linux 的“休眠”不是一个单独动作。为了排查问题，应把它拆成下面几段：

```text
用户态准备/冻结
    ↓
设备 suspend
    ↓
平台 noirq / late suspend
    ↓
平台固件或 SoC 深度低功耗入口
    ↓
CPU/时钟/电源域进入低功耗
    ↓
唤醒返回
    ↓
CPU/平台恢复
    ↓
设备 resume
    ↓
UART、serdev、hci_bcm、HCI 链路恢复
    ↓
用户态服务恢复
```

### 2.1 用户态准备与进程冻结

系统先停止或冻结用户进程，防止在设备挂起过程中继续产生普通 I/O。这个阶段通过 `pm_test=freezer` 可以单独验证。

如果 freezer 阶段就失败，问题还没有进入蓝牙设备驱动，应该优先看进程、任务冻结和唤醒源。

### 2.2 设备挂起

PM core 按设备依赖关系调用各设备的 suspend 回调。UART、serdev、蓝牙协议驱动都可能在这一阶段执行自己的准备动作。

对本案例来说，关键动作包括：

- 停止或限制 UART 数据流；
- 保存 UART 控制器状态；
- 处理 RTS/CTS；
- 让 BCM 知道主机即将进入睡眠；
- 配置 HOST_WAKE 为系统唤醒源；
- 关闭或切换 runtime PM 状态。

### 2.3 平台与真实低功耗入口

设备回调全部完成，并不代表 SoC 已经真正进入 deep。平台层还要执行实际的低功耗入口，可能涉及：

- 固件调用；
- 外设时钟门控；
- 电源域切换；
- reset domain 状态；
- pinctrl/IO retention；
- UART 寄存器和 FIFO 是否保留；
- 唤醒中断路由是否保留。

这是 `pm_test=core` 不能完全覆盖的边界，也是本案例最重要的分界点。

### 2.4 唤醒和设备恢复

唤醒后流程大体反向执行，但“反向”不等于“简单倒序”。父设备、UART 控制器、serdev 客户端、协议驱动和 HCI core 之间存在依赖关系，任何一个层级恢复得太早或太晚，都可能表现成第一条 HCI 命令超时。

## 3. 休眠命令分哪几类

### 3.1 选择低功耗模式

```sh
cat /sys/power/mem_sleep
echo s2idle > /sys/power/mem_sleep
echo deep > /sys/power/mem_sleep
```

- `s2idle`：软件空闲，通常不进入最深的 SoC/平台电源状态；
- `deep`：通常对应 suspend-to-RAM 或平台定义的深度低功耗。

如果 `s2idle` 稳定而 `deep` 失败，范围会明显偏向真实低功耗、时钟/复位、IO retention、UART 恢复和硬件握手，而不是 BlueZ 业务。

### 3.2 触发一次休眠

常见入口包括：

```sh
echo mem > /sys/power/state
systemctl suspend
rtcwake -m mem -s 50
```

它们最终都可能进入系统 suspend，但唤醒来源和等待方式不同。自动 RTC 唤醒适合固定休眠时长；外部按键、GPIO 或 HOST_WAKE 唤醒则适合验证唤醒源链路。

### 3.3 PM 分阶段调试

```sh
echo freezer   > /sys/power/pm_test
echo devices   > /sys/power/pm_test
echo platform  > /sys/power/pm_test
echo processors > /sys/power/pm_test
echo core      > /sys/power/pm_test
echo none      > /sys/power/pm_test
```

这些模式用于把完整 suspend/resume 拆开：

- `freezer)：只验证进程冻结；
- `devices)：验证设备挂起/恢复；
- `platform)：验证平台相关阶段；
- `processors)：验证处理器相关阶段；
- `core)：跑核心 suspend/resume 流程，但不等价于真正调用平台固件完成 deep 低功耗；
- `none)：恢复正常测试。

`pm_test=core` 通过的准确含义是：核心 PM 流程、设备回调和基本恢复链路没有直接卡死。它不能证明真实 deep 期间 UART 的寄存器、时钟、pinmux、FIFO、IO retention 或 BCM 电气握手都正确。

## 4. serdev、UART、HCI 的层次顺序

### 4.1 发送方向

```text
bluetoothd / hciconfig / bluetoothctl
              ↓
Linux Bluetooth HCI core
              ↓
hci_uart
              ↓
hci_serdev
              ↓
serdev core
              ↓
SoC UART controller driver
              ↓
UART TX + RTS/CTS
              ↓
BCM Bluetooth controller
```

### 4.2 接收方向

```text
BCM controller
      ↓
UART RX + CTS/RTS
      ↓
SoC UART controller
      ↓
serdev receive callback
      ↓
hci_serdev
      ↓
hci_uart H4 packet parser
      ↓
Bluetooth HCI core
      ↓
用户态
```

这里最容易混淆的是：

- `serdev` 是串行设备总线框架，不是 HCI 协议；
- `hci_serdev` 把 HCI UART 客户端接到 serdev；
- `hci_uart` 负责 H4 等 UART HCI 协议和传输层；
- `hci_bcm` 是 Broadcom 控制器的协议/电源管理适配；
- SoC UART 驱动负责真正的时钟、FIFO、DMA、pinmux、硬件流控和物理引脚。

因此，“蓝牙驱动恢复成功”不代表 UART 硬件已经能可靠收发数据；“UART 节点还在”也不代表 BCM 已经完成唤醒。

## 5. DTS 给出的事实

目标设备树的关键节点可以抽象为：

``
dts
&uart9 {
        compatible = "brcm,bcm4345c5";
        device-wakeup-gpios = <...>; /* BT_WAKE */
        host-wakeup-gpios   = <...>; /* HOST_WAKE */
        shutdown-gpios      = <...>; /* BT_REG_ON */
        max-speed = <1500000>;
};
```

三个 GPIO 的语义不同：

| 信号 | 方向 | 作用 |
|---|---|---|
| BT_WAKE / device-wakeup | 主机到 BCM | 主机请求 BCM 唤醒或保持活动 |
| HOST_WAKE / host-wakeup | BCM 到主机 | BCM 请求主机唤醒或表示有数据要发 |
| BT_REG_ON / shutdown | 主机到 BCM | 电源/复位使能路径，通常不是每次普通 resume 都翻转 |

DTS 只能说明连接关系和配置意图，不能单独证明 deep 期间电平保持、IO retention、pinmux 复原、UART 时钟存在或线上的波形正确。

## 6. 关键内核代码链路

以下函数名对应目标 Linux 6.12 树；公开源码可对照 Linux 的 [hci_bcm.c](https://github.com/torvalds/linux/blob/v6.12/drivers/bluetooth/hci_bcm.c)、[hci_ldisc.c](https://github.com/torvalds/linux/blob/v6.12/drivers/bluetooth/hci_ldisc.c)、[hci_serdev.c](https://github.com/torvalds/linux/blob/v6.12/drivers/bluetooth/hci_serdev.c)、[serdev core](https://github.com/torvalds/linux/blob/v6.12/drivers/tty/serdev/core.c) 和 [Bluetooth HCI 头文件](https://github.com/torvalds/linux/blob/v6.12/include/net/bluetooth/hci.h)核对。

### 6.1 驱动组成

`drivers/bluetooth/Makefile` 把以下部分组合到 hci_uart：

- `hci_uart.o`
- `hci_ldisc.o`
- `hci_serdev.o`
- `hci_h4.o`
- `hci_bcm.o`

设备树里的 Broadcom serdev 子设备由 hci_uart/serdev 体系发现，随后 hci_bcm 注册 Broadcom 协议和 PM 回调。

### 6.2 suspend 方向

核心路径可以概括为：

```text
PM core
  → bcm_suspend()
      → bcm_suspend_device()
          → hci_uart_set_flow_control(hu, true)
          → 设置 is_suspended
          → set_device_wakeup(false)
          → msleep(15)
  → UART/平台继续进入 deep
```

需要特别注意：

- `bcm_suspend_device()` 本身没有返回状态；
- `bcm_suspend()` 调用后通常返回成功；
- PM 返回成功并不能证明 BT_WAKE、RTS/CTS 和 UART 状态真的按预期完成；
- GPIO 操作、UART flow-control 和真实电气波形需要单独观测。

### 6.3 resume 方向

```text
平台/CPU 从 deep 返回
  → UART/serdev 父设备恢复
      → bcm_resume()
          → bcm_resume_device()
              → set_device_wakeup(true)
              → msleep(15)
              → hci_uart_set_flow_control(hu, false)
  → HCI core 发送唤醒后的首条命令
  → serdev write
  → UART TX
  → BCM 处理并通过 UART 返回 HCI event
  → hci_serdev 接收
  → hci_uart/H4 解析
```

目标树中 `bcm_resume_device()` 的固定等待从 15 ms 改成 1 秒后，使用相同的 50 秒 deep、5 次测试仍然没有恢复成功。日志仍出现类似：

```text
resume, delaying 1000 ms
hardware error 0x00
HCI command timeout, for example 0x0c01 -110
```

因此当前证据不支持“只是 BCM 需要更长固定等待”这一解释。

### 6.4 flow-control 的实际落点

在 serdev 路径中，`hci_uart_set_flow_control()` 会落到类似下面的动作：

```text
serdev_device_set_flow_control(...)
serdev_device_set_rts(...)
```

在传统 tty 路径中则可能操作 CRTSCTS 和 RTS。最终的逻辑值、电气极性和 pinctrl 状态仍由 UART 控制器、serdev 和板级连接共同决定，不能只看一个软件布尔值。

### 6.5 HCI 首条命令

唤醒后看到的超时命令可能是 `HCI_OP_SET_EVENT_MASK`（`0x0c01`）或其他初始化/恢复命令。HCI 层面看到 `-ETIMEDOUT`，只说明在规定时间内没有收到匹配的 command complete/event；它不能区分：

- 主机根本没有发出 TX；
- TX 发出了，但 UART FIFO/时钟不工作；
- BCM 没有被 BT_WAKE 唤醒；
- BCM 收到了命令但没有返回；
- BCM 返回了，但 RX/CTS/HOST_WAKE 丢失；
- 字节到了 UART，但 H4 parser 没有得到完整事件。

所以要把 HCI 超时继续向下拆成“TX 是否发生”和“RX event 是否到达”。

## 7. BT_WAKE、HOST_WAKE、RTS/CTS 的正常时序

以下是逻辑顺序，具体高低电平必须以 DTS 的 GPIO polarity、UART 控制器定义和原理图为准。

### 7.1 正常工作

```text
UART 时钟、pinmux、FIFO 正常
        ↓
主机和 BCM 都处于可通信状态
        ↓
BT_WAKE 用于主机请求 BCM 保持唤醒
HOST_WAKE 用于 BCM 请求主机处理数据/唤醒
RTS/CTS 用于阻止接收方来不及处理时继续发送
```

### 7.2 suspend 前后

```text
停止新的 HCI TX / 等待 UART 空闲
        ↓
设置 UART flow-control 和 RTS 到 suspend 安全状态
        ↓
BT_WAKE 去往非唤醒状态
        ↓
等待控制器完成过渡（代码中为 15 ms）
        ↓
进入真实 deep
```

HOST_WAKE 是 BCM 到主机的输入/中断方向，通常作为唤醒源配置；它不是由 `bcm_resume_device()` 像 BT_WAKE 那样直接拉高或拉低。

### 7.3 resume 前后

```text
先恢复 UART 电源/时钟/reset/pinmux/FIFO
        ↓
主机通过 BT_WAKE 请求 BCM 唤醒
        ↓
等待控制器稳定（代码中为 15 ms）
        ↓
恢复 RTS/CTS 和 UART flow-control
        ↓
发送首条 HCI 命令
        ↓
BCM 返回 command complete 或 hardware event
```

如果 UART 在“恢复 flow-control”之前尚未真正恢复，或者 BT_WAKE 已经有效但 UART 仍没有可用时钟，那么把 15 ms 增大到 1 秒仍然没有意义。

## 8. 已完成实验及其正确解读

### 8.1 `power/control=auto` 与 `on`

```text
power/control=auto
    = 允许 runtime PM 根据空闲状态自动挂起/恢复

power/control=on
    = 禁止该设备进入 runtime suspend，强制保持 runtime active
```

它们只反映 runtime PM 策略，不等于系统 suspend 时 UART 一定保持全功耗。执行 `echo mem > /sys/power/state` 时，system PM 仍然会执行自己的 suspend/resume 回调，平台也仍可能切换时钟、电源域和 IO retention。

在相同的真实 deep、约 50 秒休眠、5 次测试条件下：

- UART `power/control=auto)：5 次均失败；
- UART `power/control=on)：5 次均失败。

因此可以排除“单纯把 UART runtime PM 改为 auto 或 on 就能解决”的方向。早期“on 连续 2 次通过”的结论因样本太少且未按失败后恢复规则执行，已经撤回。

### 8.2 `pm_test=core`

`pm_test=core` 可以让 suspend/resume 核心链路走一遍且蓝牙仍正常，说明：

- 基础设备回调没有直接卡死；
- 基础 UART/serdev/HCI 恢复路径在浅层测试中可以工作；
- 不是所有 suspend/resume 路径都必然失败。

但它没有覆盖真实 deep 的平台固件和最深低功耗动作。因此它降低了“普通 PM 回调立即报错”的可能性，不能排除“驱动与真实硬件低功耗状态交互”的问题。驱动问题仍可能只在真实 deep 后暴露。

### 8.3 VBAT、32 kHz、BT_REG_ON

休眠期间观察到：

- VBAT 正常；
- 32 kHz 正常；
- BT_REG_ON 正常。

这可以降低以下可能性：

- BCM 主电源完全消失；
- 低频时钟完全停止；
- BT_REG_ON 被错误拉低导致控制器复位。

但不能排除：

- UART 电源域或时钟域单独被关闭；
- UART FIFO/寄存器没有 retention；
- pinmux/IO retention 不正确；
- RTS/CTS/TX/RX 有边沿或极性问题；
- HOST_WAKE 中断路由丢失；
- BCM 内部仍处于不可接受 HCI 命令的状态。

### 8.4 15 ms 改成 1 秒

把 resume 中的固定等待从 15 ms 改成 1 秒，重新构建并加载匹配的临时 hci_uart 模块，未刷写 boot，进行相同的 50 秒 deep、5 次测试，仍然全部失败。

这个实验排除了“只需要更长固定延时”的简单解释，但没有排除“时序顺序错误”。例如：

```text
UART 尚未恢复
    → BT_WAKE 已拉起
    → 等待 1 秒
    → flow-control 恢复
    → 首条 HCI 命令仍发不出去/收不回来
```

### 8.5 BlueZ/用户态

停止或屏蔽 bluetoothd，并通过内核 HCI 层观察仍可复现同样的 `serial0-0`/HCI 失败，说明问题不依赖 BlueZ 的业务流程。

这不是说用户态永远不可能有问题，而是说当前故障已经在用户态之下出现，继续优先分析 bluetoothd 没有收益。

## 9. 当前排除范围和保留范围

| 层次 | 当前判断 | 依据 |
|---|---|---|
| 蓝牙业务应用 | 基本排除 | 不需要业务连接即可复现 |
| bluetoothd/BlueZ | 基本排除 | 用户态停止后仍能在内核 HCI 链路复现 |
| HCI 初始枚举/普通收发 | 部分排除 | 正常启动、正常运行阶段可用 |
| runtime PM `auto/on` 策略 | 基本排除 | 两种策略下 50 秒 deep ×5 均失败 |
| `pm_test=core` 覆盖的基础流程 | 降低可能性 | core 测试通过 |
| VBAT/32 kHz/BT_REG_ON 粗粒度掉电 | 降低可能性 | 休眠期间测量正常 |
| 15 ms 固定等待过短 | 降低可能性 | 改为 1 秒后仍失败 |
| 真实 deep 平台状态 | 未排除，优先级高 | 只有真实 deep 触发失败 |
| UART 恢复、FIFO、时钟、pinmux、retention | 未排除，优先级最高 | 首条 HCI 命令响应失败 |
| RTS/CTS、BT_WAKE、HOST_WAKE 时序 | 未排除，优先级最高 | 没有 UART 波形证据 |
| BCM 内部唤醒/固件状态 | 未排除，优先级高 | 电源正常但唤醒后不响应 |
| HOST_WAKE IRQ 路径 | 未排除 | 软件日志尚未证明实际 IRQ 来源和边沿 |
| 驱动恢复顺序和错误传播 | 未排除 | hci_bcm、serdev、UART 存在跨层依赖 |
| 物理 UART TX/RX/RTS/CTS | 未排除 | 当前没有测试点或逻辑分析波形 |

### 9.1 独立排序

不依赖“谁先提出”的顺序，按当前证据重新排序：

1. **UART/serdev 恢复与硬件流控链路**：包括 UART 时钟、FIFO、baud、RTS/CTS、DMA、父设备恢复顺序；
2. **真实 deep 导致的 UART/IO retention、pinmux、reset 或电源域状态变化**；
3. **BCM 唤醒握手和控制器内部状态**：BT_WAKE 已变化不等于 BCM 已能处理 HCI；
4. **HOST_WAKE 的 IRQ 路由、唤醒使能、极性和边沿配置**；
5. **hci_bcm/serdev/UART 的恢复顺序以及错误状态没有被及时反馈**；
6. **板级 UART 物理信号完整性**；
7. **BlueZ/用户态业务**：当前证据已经使其成为低优先级。

第 1～3 项彼此可能叠加，不能把“电源正常”误解成“控制器链路正常”。

## 10. 从链路两端向中间二分

### 10.1 第一刀：`s2idle` 对比 `deep`

```text
s2idle 通过、deep 失败
    → 真实低功耗/平台/retention/UART 恢复范围

s2idle 也失败
    → 继续看普通 system PM、serdev、hci_bcm、flow-control
```

这是当前最有价值的二分，因为它直接判断问题是否依赖真实 deep。

### 10.2 第二刀：PM test 分段

建议按同一份脚本重复测试：

```text
none
  → freezer
  → devices
  → platform
  → processors
  → core
  → 真实 deep
```

每一步都要记录：

- suspend 是否真正完成；
- resume 是否完成；
- `hci0` 是否为 `UP RUNNING`；
- 是否出现首条 HCI command timeout；
- 失败后是否做过完整恢复。

如果所有 `pm_test` 都通过、只有真实 deep 失败，软件普通回调的优先级下降，平台低功耗和硬件保持状态的优先级上升。

### 10.3 第三刀：拆 HCI 超时

用 `btmon`、HCI trace 或内核日志区分：

```text
首条 HCI 命令是否进入 hci_uart
        ↓
hci_serdev 是否调用 write
        ↓
UART TX FIFO/寄存器是否有变化
        ↓
BCM 是否通过 HOST_WAKE/CTS 表示可通信
        ↓
UART RX 是否收到字节
        ↓
hci_serdev 是否收到 buffer
        ↓
H4 parser 是否得到完整 event
        ↓
HCI core 是否完成 command
```

这一步能把“控制器没有响应”和“响应没有被主机收到”分开。

### 10.4 第四刀：检查真实唤醒 IRQ

不能看到某个 IRQ 计数变化就直接认定它是 HOST_WAKE。需要同时核对：

```sh
cat /proc/interrupts
cat /proc/irq/<irq>/wakeup
cat /sys/kernel/debug/wakeup_sources
cat /sys/power/pm_wakeup_irq
```

确认：

- 该 IRQ 是否对应 DTS 中的 HOST_WAKE；
- 休眠前是否启用了 IRQ wake；
- 唤醒后是否命中预期 IRQ；
- 极性和触发边沿是否与硬件一致；
- UART 或其他 IRQ 是否错误地成为实际唤醒源。

### 10.5 第五刀：软件和示波器同时看 UART

软件侧检查：

- UART 当前 baud、termios、CRTSCTS；
- UART clock/reset 是否已 enable；
- FIFO 是否有残留字节；
- RX/TX status、错误状态、DMA 状态；
- pinctrl 是否回到 UART9 的复用状态；
- RTS/CTS 当前逻辑值和方向；
- serdev open/close 和 runtime PM 引用计数。

如果软件侧无法证明物理线，必须在 UART 引脚上使用逻辑分析仪或示波器观察：

- BT_WAKE；
- HOST_WAKE；
- TX；
- RX；
- RTS；
- CTS；
- 休眠前、进入 deep、唤醒、首条 HCI 命令四个时间窗口。

当前没有 UART 测试点，因此硬件信号问题还不能被排除。

## 11. 建议加入的调试锚点

### 11.1 PM 和设备回调

启用 PM trace，重点看：

```text
power:suspend_resume
power:device_pm_callback_start
power:device_pm_callback_end
```

关注顺序：

```text
UART parent resume
  → serdev resume
      → hci_bcm resume
          → 首条 HCI TX
```

如果 hci_bcm resume 已结束但 UART 还没有恢复完成，或者首条 HCI TX 发生在 UART clock/pinmux 完成之前，就能直接证明顺序问题。

### 11.2 hci_bcm、hci_serdev、hci_ldisc

只对相关文件打开 dynamic debug，避免全内核日志淹没：

- `hci_bcm.c)：suspend/resume、BT_WAKE、HOST_WAKE、延时；
- `hci_serdev.c)：write、receive buffer、serdev open；
- `hci_ldisc.c)：flow-control、RTS、serdev/tty 分支；
- SoC UART 驱动：clock、reset、FIFO、DMA、termios 和 runtime/system PM。

### 11.3 首条命令前后加时间戳

至少记录：

```text
t0: UART parent resume end
t1: BT_WAKE active
t2: 15 ms/1 s delay end
t3: flow-control/RTS restored
t4: HCI command write called
t5: UART TX accepted
t6: HOST_WAKE asserted
t7: UART RX callback
t8: HCI event parsed
t9: command complete
```

只要缺少一个时间点，下一轮实验仍可能停留在“看起来像超时”的层面。

## 12. 结论

当前不能把问题概括成“蓝牙服务没有恢复”，也不能因为 `pm_test=core` 通过就排除驱动。更准确的结论是：

```text
Linux 休眠准备阶段基本正常
    ↓
基础 suspend/resume PM 流程可以运行
    ↓
只有真实 deep 低功耗返回后出现异常
    ↓
电源、32 kHz、BT_REG_ON 未见粗粒度异常
    ↓
BCM 唤醒后的首条 HCI 命令没有完成
    ↓
最小可疑范围：
真实 deep 状态变化
+ UART/serdev 恢复与硬件流控
+ BT_WAKE/HOST_WAKE/RTS/CTS 握手
+ BCM 内部唤醒状态
```

最快的下一步不是继续增加固定延时，而是先做 `s2idle` 与 `deep` 的对比，再把 PM、UART TX、HOST_WAKE、UART RX 和 HCI event 串成一条带时间戳的链路。这样可以用最少的实验判断问题是在“主机没有发出首条命令”，还是“BCM 没有响应”，还是“BCM 响应了但 UART/serdev 没有把它交给 HCI core”。

本文的实验规则也同样重要：每次失败后先恢复到已确认正常，再开始下一次；每一批明确休眠时长、样本数、恢复动作和成功判据。否则看似样本很多，实际上测量的是同一个未恢复的故障状态。
