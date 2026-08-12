# 07. Linux BCM 蓝牙 Deep Suspend/Resume：从稳定复现到 VCCIO3 IO Retention

> 本文记录一次 Linux 目标板上 Broadcom BCM 蓝牙在真正 deep 休眠约 50 秒后唤醒异常的定位过程。
>
> 目标不是把“某个回调返回 0”当成硬件已经恢复，也不是把“软件看见的寄存器正常”当成 UART 波形正常。全文按可重复实验记录证据，并明确区分：已经定位的可修复根因、被降低的候选，以及因硬件条件不足仍未细分的机制。

## 结论先行

在当前 RK3588 EVB7 V11、Linux 6.12 配置和 BCM 蓝牙组合下，已经找到可修复的软件根因边界：

> 真实 deep 休眠路径没有为 UART9 所属的 VCCIO3 IO domain 配置 IO retention。只增加 RKPM_VCCIO3_RET_EN 后，50 秒 deep 的故障从基线的 5/5 复现变为 5/5 通过；恢复原始 DTS 后再次 5/5 复现。

最小配置改动是：

~~~dts
&rockchip_suspend {
        rockchip,sleep-io-ret-config = <RKPM_VCCIO3_RET_EN>;
};
~~~

这已经足以把问题定位到“VCCIO3 IO-retention 配置缺失或不正确”这一可修复层级。

尚未定位到更底层的硬件机制：具体是 TX、RX、RTS、CTS 中哪一个 pad，或者 mux、输出使能、上下拉、隔离、瞬态中的哪一项发生了变化。UART 没有可用测试点，本轮没有示波器或逻辑分析仪波形，因此不能把这些细节写成已证实结论。

## 1. 问题与最小复现环境

### 1.1 现象

设备正常启动后，hci0 为 UP RUNNING，蓝牙可以正常工作。进入真正的 deep，休眠约 50 秒后唤醒，系统本身能够回来，但蓝牙控制器的 HCI 初始化通信失败，常见表现为：

~~~text
系统恢复
  ↓
唤醒后的首条 HCI 命令超时
  ↓
HCI hardware error 0x00 或命令 -110
  ↓
hci0 无法恢复为 UP RUNNING
~~~

### 1.2 最小测试环境

扫描、配对、音频连接和业务应用都不是复现所必需的。最小环境只有：

~~~text
Linux system PM
  + UART9 / serdev
  + hci_uart / hci_bcm
  + BCM 控制器
  + 真实 deep 休眠/唤醒
  + 唤醒后的 hci0/HCI 检查
~~~

典型测试准备和检查命令如下，实际恢复动作应使用板上已验证的恢复脚本：

~~~sh
echo deep > /sys/power/mem_sleep
hciconfig hci0
rtcwake -m mem -s 50
hciconfig hci0
hcitool cmd 0x04 0x0001
~~~

hcitool cmd 0x04 0x0001 只是一个最小 HCI 查询，用来确认唤醒后的控制器通信闭环，不代表驱动内部唯一的初始化命令。

最早曾观察到约 18 秒即可触发异常，但没有形成稳定基线；正式对照统一使用 50 秒。早期“连续 2 次通过”的判断已经撤回，两个样本不能证明稳定。

### 1.3 样本独立性

每次失败后的正确顺序是：

~~~text
记录失败
  → 完整恢复 UART/BCM/HCI
  → 确认 hci0 UP RUNNING
  → 才开始下一次 deep
~~~

如果失败后直接继续测试，下一次样本可能只是前一次异常状态的延续，不能与健康基线比较。

## 2. 先建立 PM 分层模型

### 2.1 system suspend 的阶段

~~~text
用户态准备与 freezer
        ↓
设备 prepare
        ↓
设备 suspend
        ↓
late suspend / noirq
        ↓
平台和 SoC 的低功耗入口
        ↓
唤醒返回
        ↓
平台 / CPU 恢复
        ↓
设备 resume
        ↓
UART / serdev / hci_bcm / HCI 恢复
        ↓
用户态服务恢复
~~~

真正的 deep 不只是执行 Linux 设备回调，还会进入平台固件和 SoC 的深度低功耗状态。因此“Linux 回调都返回 0”和“deep 后硬件状态正确”是两个命题。

### 2.2 休眠类型

~~~sh
cat /sys/power/mem_sleep
echo s2idle > /sys/power/mem_sleep
echo deep > /sys/power/mem_sleep
echo mem > /sys/power/state
~~~

- s2idle：suspend-to-idle，通常不进入最深的平台电源状态；
- deep：平台定义的深度内存睡眠，本案例的故障只在该路径中稳定出现；
- disk：hibernation，走保存/恢复镜像的另一条路径，不是本案例的 mem/deep。

### 2.3 pm_test 能说明什么

本次 pm_test=core 在相同设备上通过，而真实 deep 可以复现。这建立了重要边界：问题需要真实 deep 入口或该入口改变的硬件状态。

但这不能推出“关联驱动已经排除”。准确说法是：

~~~text
pm_test=core 通过
  = 普通 PM 回调路径本身没有直接报错或卡死

真实 deep 失败
  = 真实平台/SoC 低功耗动作改变了驱动依赖的硬件状态，仍可能在驱动与硬件交界处出错
~~~

bcm_suspend_device 没有返回值，也只意味着 PM core 没有从该函数得到失败码；它不验证 BCM 是否真的进入正确状态，也不验证 UART pad、RTS/CTS、FIFO 或 IO domain 在 deep 期间是否保持正确。

## 3. serdev、UART、HCI 的完整链路

### 3.1 发送方向

~~~text
用户态 / HCI core
        ↓
hci_uart
        ↓
hci_serdev
        ↓
serdev core
        ↓
SoC UART9 控制器
        ↓
TX + RTS/CTS
        ↓
BCM 控制器
~~~

### 3.2 接收方向

~~~text
BCM 控制器
        ↓
RX + RTS/CTS
        ↓
SoC UART9 控制器
        ↓
serdev receive callback
        ↓
hci_serdev
        ↓
hci_uart H4 parser
        ↓
Bluetooth HCI core
        ↓
用户态
~~~

各层职责不同：

| 层 | 主要职责 |
|---|---|
| HCI core | 管理控制器、命令和事件 |
| hci_uart | UART H4 传输和字节解析 |
| hci_serdev | 将 HCI UART 客户端接入 serdev |
| serdev | 串行设备和 UART client 连接 |
| hci_bcm | BCM 唤醒、流控和控制器 PM 适配 |
| UART9/8250-DW | 时钟、FIFO、IRQ、DMA、pinmux、RTS/CTS 和物理引脚 |

这是静态架构图，不等于一次 resume 的实际事件顺序。实际 PM 回调顺序由设备层级和 PM core 决定，必须结合时间戳验证。

### 3.3 休眠/唤醒中的控制信号

逻辑上，进入低功耗通常要停止新的 HCI TX、处理 UART 流控、释放 BCM 唤醒请求，再进入平台低功耗；唤醒时则要先让 UART/IO 可用，申请 BCM 唤醒，恢复流控，然后发送 HCI 命令。

~~~text
进入 suspend:
  停止 HCI TX → 处理 RTS/CTS → BT_WAKE 进入非唤醒状态 → deep

返回 resume:
  UART/IO 可用 → BT_WAKE 请求 BCM 唤醒 → 恢复 RTS/CTS → 首条 HCI 命令
~~~

HOST_WAKE 是 BCM 到主机的输入/唤醒方向，不是由主机像 BT_WAKE 一样直接拉动。正常逻辑并不能证明实际电平和波形在 deep 窗口内没有瞬态错误。

## 4. 分层实验与排除边界

下面列出的是正式或有效的对照。早期不满足样本独立性、或者只有 1～2 次的结果，不作为稳定结论。

| 实验 | 结果 | 能说明什么 |
|---|---|---|
| 停止 bluetoothd/BlueZ，保留内核 HCI/serdev | 仍可复现 | 用户态业务不是必要条件 |
| s2idle，50 秒，3/3 | 全部通过 | 普通 idle 路径不触发该现象 |
| 原始配置 deep，50 秒，5/5 | 全部失败 | 建立稳定基线 |
| pm_test=core | 通过 | 基础 PM 流程本身不足以复现 |
| UART/serdev runtime PM 为 auto，50 秒，5/5 | 全部失败 | 不是单纯 runtime autosuspend 策略 |
| UART/serdev runtime PM 为 on，50 秒，5/5 | 全部失败 | 保持 runtime active 仍不能覆盖真实 deep 的影响 |
| bcm_resume_device 等待从 15 ms 改为 1 秒 | 仍失败 | 不是简单的固定等待时间不足 |
| 失败后 UART 驱动解绑/重绑 | 可恢复当前设备 | 这是恢复手段，不是根因证明 |
| 失败后 UART 寄存器、FIFO、时钟和 pinmux 快照 | 返回后软件状态接近健康状态 | 降低“返回后永久卡在明显错误状态”的概率，但不能排除 deep 期间瞬态 |
| VBAT、32 kHz、BT_REG_ON | 静态状态正常 | 降低粗粒度掉电/复位解释，不能排除 VCCIO3 pad retention |

这里尤其要注意两个容易混淆的结论：

1. power/control=on 只禁止 runtime suspend，不等于系统 deep 期间 UART 全程保持寄存器、时钟和 IO pad 状态。
2. pm_test=core 通过只说明基础 PM 回调路径没有直接复现，不能排除“真实 deep 改变硬件状态后，驱动恢复链出现问题”。

UART 没有可用测试点，因此 TX/RX/RTS/CTS 的实际电平和时序从未被硬件测量。这一项必须在证据矩阵中保留为“未验证”，不能用软件快照代替。

## 5. 从 UART 软件状态继续缩小

在 5 个正式 deep 失败样本中，失败后读取到的 UART 软件状态大致保持一致：

- LCR、IIR、USR 和 FIFO 水位没有显示出稳定的永久异常；
- clk_uart9、sclk_uart9、pclk_uart9 返回后仍为启用状态；
- UART9 四个 pad 仍显示为 uart9m0-xfer、uart9m0-rtsn、uart9m0-ctsn；
- clock_* tracepoint 在本平台没有产生可用事件，不能据此证明 deep 期间没有瞬时关钟。

因此可以降低：

~~~text
唤醒返回后 UART 寄存器/时钟/FIFO 永久停在错误状态
~~~

但不能排除：

~~~text
deep 期间瞬时关钟、IO 隔离、pad 状态变化、RTS/CTS 波形异常
~~~

这正是下一步查 IO retention 的理由：它可能在软件读取寄存器之前已经恢复，看起来正常，但在 deep 入口/返回窗口中仍影响了 BCM 的 UART 状态。

## 6. DTS、代码与 VCCIO3 的证据

### 6.1 当前 DTS 的缺口

当前有效的 UART9 配置只有默认 pinctrl，未显式配置 UART9 sleep pinctrl，也没有 rockchip,sleep-io-ret-config：

~~~dts
&uart9 {
        pinctrl-names = "default";
        pinctrl-0 = <&uart9m0_xfer &uart9m0_ctsn &uart9m0_rtsn>;
        uart-has-rtscts;
};
~~~

这不是“没有 sleep pinctrl 就一定坏”，但它说明当前配置没有在 Linux DTS 中显式声明 UART9 pad 的深睡保持策略。

### 6.2 Rockchip retention API

目标树中的头文件定义：

~~~c
/* io retention config */
#define RKPM_VCCIO3_RET_EN BIT(3)
~~~

BIT(3) 是软件位掩码，即 1 << 3，值为 0x8。它不是寄存器偏移。Linux 代码在 rockchip_pm_config.c 中读取 rockchip,sleep-io-ret-config，再通过 SUSPEND_IO_RET_CONFIG 的 SMC 接口交给 BL31/安全固件/PMU 处理。Linux 当前源码不能证明安全固件最终使用的具体物理寄存器。

### 6.3 UART9 与 VCCIO3

当前 DTS/pinctrl 映射的 UART9 M0 信号是：

| 信号 | SoC pad |
|---|---|
| TX | GPIO2_C2 |
| RX | GPIO2_C4 |
| RTS | GPIO4_C4 |
| CTS | GPIO4_C5 |

这些 pad 位于 VCCIO3 IO domain。工作区中的 EVB 原理图资料是 V10 版本，能用于核对网络拓扑；如果要做 V11 的最终硬件签核，应以 V11 原理图或实物为准。

### 6.4 IO retention 不等于电源保持

必须把三个概念分开：

| 名称 | 含义 | 不能直接推出 |
|---|---|---|
| VCCIO3_1V8 | VCCIO3 pad domain 的供电电压/电源网络 | pad mux、方向、输出使能和状态在 deep 中一定保持 |
| regulator-on-in-suspend | 请求外部 PMIC regulator 在 suspend 中保持开启 | SoC 内部 IO retention 已开启 |
| RKPM_VCCIO3_RET_EN | 请求 SoC 对 VCCIO3 IO domain 启用 retention | 外部电源一定保持 1.8 V |

IO retention 的含义是：低功耗期间由 SoC 的 retention/isolation 机制保持选定 IO pad 的状态或上下文。具体保持输出值、方向、上下拉、keeper、mux 或隔离行为中的哪些部分，要以芯片 TRM 和安全固件实现为准。

所以：

~~~text
VCCIO3 供电保持
        ≠
VCCIO3 IO pad 状态保持
~~~

把外部 VCCIO3_1V8 或 PLDO_REG2 保持开启的对照仍然不能替代 IO retention；该对照仍复现时，说明“电源轨保持”和“pad 状态保持”是两个独立控制面。

## 7. 决定性 A/B：只改变 IO retention

这是本次定位中信息量最高的实验。实验隔离条件如下：

- 使用同一内核、同一 resource image 和同一测试流程；
- 只修改 DTS 并重新生成 DTB；
- 不修改 UART 驱动、BCM 驱动、HCI 延时或用户态服务；
- 每次失败后先恢复 hci0 UP RUNNING 再开始下一次。

结果：

| 配置 | 50 秒 deep 结果 |
|---|---:|
| 原始 DTS，无 rockchip,sleep-io-ret-config | 5/5 失败 |
| 只增加 RKPM_VCCIO3_RET_EN | 5/5 通过 |
| 恢复原始 DTS | 5/5 失败 |
| 再增加 RKPM_VCCIO3_RET_EN 的反向复测 | 1/1 通过 |

这不是“通过一次就宣布稳定”。它的价值在于因果可逆：

~~~text
无 retention → 失败
加 retention → 通过
去 retention → 失败
再加 retention → 通过
~~~

在当前平台和测试条件下，这已经把故障从“UART/BCM/用户态的很多可能性”收敛到 VCCIO3 IO-retention 配置这一可复现、可修复的控制点。

## 8. 根因边界：什么已经找到，什么还没有

### 已经找到的根因

可以提交修复的表述是：

> deep suspend 配置遗漏了 UART9 所属 VCCIO3 IO domain 的 IO retention。真实 deep/LOGOFF 期间，UART9 pad 状态没有得到配置层面的保持保证，导致唤醒后的 BCM HCI 通信失败。

这是“软件配置/平台集成层”的根因，证据是 DTS 单变量、可逆 A/B，而不是单纯的相关性观察。

### 已经可以排除或大幅降低的部分

- bluetoothd、BlueZ 和具体蓝牙业务不是复现必要条件；
- 单纯 runtime PM auto 策略不是根因，on 也不能解决真实 deep；
- 单纯把 BCM resume 固定等待从 15 ms 延长到 1 秒不能解决；
- VBAT、32 kHz、BT_REG_ON 的静态异常不是主要解释；
- 唤醒返回后软件可见的 UART 寄存器、FIFO、时钟和 pinmux 没有呈现稳定永久损坏。

### 仍未定位到的更底层细节

- TX/RX/RTS/CTS 中具体哪根线受影响；
- 是输出值、方向、mux、pull/keeper、隔离还是瞬态毛刺；
- deep 期间是否有短暂的 UART 时钟或 pad 电平异常；
- BCM 内部 H4/HCI 状态被哪一个具体事件破坏；
- retention 由 BL31/PMU 最终落到哪个硬件寄存器和字段。

这些是“机制级细化”，不是当前软件修复成立的前置条件。因为没有 UART 测试点，本轮不能用波形把它们进一步拆开。

## 9. 最终定位图

~~~text
用户态 / BlueZ
        ↓ 已证明不是必要条件
HCI core / hci_uart / serdev 基础路径
        ↓ 普通运行和浅层 PM 正常
pm_test=core
        ↓ 通过
真实 deep / LOGOFF
        ↓
VCCIO3 IO-retention 配置缺失
        ↓
UART9 所属 IO domain 的 deep 期间 pad 保持不充分
        ↓
BCM 唤醒后的首条 HCI 通信失败
        ↓
hci0 异常
~~~

## 10. 修复与验证边界

建议保留的最小修复是：

~~~dts
&rockchip_suspend {
        rockchip,sleep-io-ret-config = <RKPM_VCCIO3_RET_EN>;
};
~~~

5/5 对照足以证明这项配置改变了故障结果，但不等于对无限次休眠作出统计保证。产品验收仍应在最终内核、BL31、DTB、模块和板卡版本完全固定后，按独立样本进行更长批次回归。

本次实验结束后，测试设备已恢复到原始运行状态；文章中的 DTS 片段是已验证的修复方案，不表示测试现场仍保留临时实验镜像。

## 11. 源码审计地图

以下路径来自目标 Linux 6.12 树：

| 目标 | 路径 |
|---|---|
| VCCIO retention bit 定义 | kernel-6.12/include/dt-bindings/suspend/rockchip-rk3588.h |
| DTS 属性读取和 SMC 下发 | kernel-6.12/drivers/soc/rockchip/rockchip_pm_config.c |
| 当前 UART9/BCM DTS | kernel-6.12/arch/arm64/boot/dts/rockchip/rk3588-evb7-v11-linux.dts |
| VCCIO3 UART9 pinctrl | kernel-6.12/arch/arm64/boot/dts/rockchip/rk3588-vccio3-pinctrl.dtsi |
| Rockchip pinctrl IOC 映射 | kernel-6.12/drivers/pinctrl/pinctrl-rockchip.c |
| Broadcom suspend/resume | kernel-6.12/drivers/bluetooth/hci_bcm.c |
| HCI serdev 连接 | kernel-6.12/drivers/bluetooth/hci_serdev.c |
| serdev core | kernel-6.12/drivers/tty/serdev/core.c |
| UART 控制器 PM | kernel-6.12/drivers/tty/serial/8250/8250_dw.c |

通用 Linux 机制可参阅：

- [Linux Basic PM Debugging](https://www.kernel.org/doc/html/latest/power/basic-pm-debugging.html)
- [Linux CPU and Device Power Management](https://www.kernel.org/doc/html/latest/driver-api/pm/index.html)
- [Linux suspend flows](https://cdn.kernel.org/doc/html/latest/admin-guide/pm/suspend-flows.html)
- [upstream hci_bcm.c](https://github.com/torvalds/linux/blob/v6.12/drivers/bluetooth/hci_bcm.c)
- [upstream hci_serdev.c](https://github.com/torvalds/linux/blob/v6.12/drivers/bluetooth/hci_serdev.c)

## 12. 一句话总结

这不是“蓝牙服务休眠后没重启”，也不是“把 UART runtime PM 设为 on 就能解决”的问题。可重复的因果证据已经把它定位到：

> RK3588 真实 deep suspend 中，UART9 所属 VCCIO3 IO domain 缺少 IO retention 配置；补上该配置后，BCM 蓝牙的唤醒 HCI 通信恢复正常。

更底层的具体 pad 和波形仍待硬件测量，但不影响当前最小软件修复的成立。

