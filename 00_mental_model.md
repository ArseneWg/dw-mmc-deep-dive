# 00. 先建立心智模型

读 `dw_mmc.c` 最大的坑，是一上来就把注意力放到 `dw_mci_tasklet_func()`。这个函数确实是核心状态机，但它不是整个驱动的入口，也不是事件的来源。更好的读法是先把文件分成四层：MMC core 接口层、请求生命周期层、事件/状态机层、硬件服务层。

![整体架构](assets/architecture.svg)

## 1. 这个驱动在 MMC 栈里的位置

Linux MMC core 管卡的枚举、块设备、SDIO function、调度和公共协议；host driver 负责把 core 发来的 `mmc_request` 转成控制器寄存器操作。`dw_mmc.c` 就是 Synopsys DesignWare MMC 控制器的 host driver 核心。

真正暴露给 MMC core 的入口集中在 `dw_mci_ops`：`request` 进入请求路径，`pre_req/post_req` 配合 DMA 映射，`set_ios` 配置电源/时钟/总线宽度，`start_signal_voltage_switch` 处理信号电压切换，`enable_sdio_irq/ack_sdio_irq` 处理 SDIO 中断。这张表在 `kernel/drivers/mmc/host/dw_mmc.c:1884` 开始。

这意味着：如果你想知道“MMC core 怎么调用 dw_mmc”，先看 `dw_mci_ops`；如果你想知道“设备怎么注册成一个 MMC host”，先看 `dw_mci_probe()` 和 `dw_mci_init_slot()`；如果你想知道“一次读写怎么跑完”，再看 `dw_mci_request()` 和 tasklet。

## 2. 三个对象：host、slot、request

`struct dw_mci` 是控制器级别状态。它保存寄存器地址、DMA 能力、当前请求、当前命令、当前数据、tasklet、事件位、状态机状态和等待队列。定义和注释在 `kernel/drivers/mmc/host/dw_mmc.h:58`，其中 `pending_events`、`completed_events`、`state` 的角色写得很清楚，字段本身在 `kernel/drivers/mmc/host/dw_mmc.h:197` 到 `kernel/drivers/mmc/host/dw_mmc.h:200`。

`struct dw_mci_slot` 是卡槽级别状态。这个驱动当前主要按一个 slot 使用，slot 内部挂着 `struct mmc_host`，`mmc_priv(mmc)` 取回的就是 slot。

`struct mmc_request` 是一次来自 core 的事务。它可能有 `sbc`、`cmd`、`data`、`stop`。读写块设备时通常是命令加数据，某些多块传输还会带 stop。`dw_mmc.c` 的请求路径就是围绕这四件东西做编排。

## 3. 文件里真正有几类“状态”

头文件里有一个显式状态机：`enum dw_mci_state`，定义在 `kernel/drivers/mmc/host/dw_mmc.h:21`。状态包括：

| 状态 | 人话解释 |
|---|---|
| `STATE_IDLE` | 控制器空闲，可以立刻启动新请求 |
| `STATE_SENDING_CMD` | 命令已经写入控制器，等待命令完成事件 |
| `STATE_SENDING_DATA` | 命令已完成，数据正在从 DMA/PIO/FIFO 侧搬运 |
| `STATE_DATA_BUSY` | 内存侧数据搬完，等待卡/控制器侧 data over |
| `STATE_SENDING_STOP` | stop/abort 命令已经发出，等待 stop 命令完成 |
| `STATE_DATA_ERROR` | 数据错误后进入过渡状态，等 transfer complete 后回到 data busy 收尾 |
| `STATE_SENDING_CMD11` | SD voltage switch 的 CMD11 特殊命令正在发送 |
| `STATE_WAITING_CMD11_DONE` | CMD11 命令完成，但电压切换/clock 更新还没完全收尾 |

还有一个隐式事件机：`EVENT_CMD_COMPLETE`、`EVENT_XFER_COMPLETE`、`EVENT_DATA_COMPLETE`、`EVENT_DATA_ERROR`，定义在 `kernel/drivers/mmc/host/dw_mmc.h:32`。事件不是状态，事件是 IRQ/timer/DMA/PIO 交给 tasklet 的“信号”。

所以这份代码至少要分两张表理解：状态表回答“现在卡在哪一步”，事件表回答“下一步靠什么信号触发”。把两者强行画成一张大图，读者会被交叉箭头淹没。

## 4. 两条主线：控制线和数据线

控制线是命令：`dw_mci_prepare_command()` 组 CMDR bits，`dw_mci_start_command()` 写 CMDARG/CMD 并启动响应 timeout，命令完成由 IRQ 或 timer 设置 `EVENT_CMD_COMPLETE`。对应代码在 `kernel/drivers/mmc/host/dw_mmc.c:283`、`kernel/drivers/mmc/host/dw_mmc.c:428`、`kernel/drivers/mmc/host/dw_mmc.c:2920`。

数据线是搬运：有 data 的命令会先写 BYTCNT/BLKSIZ，然后 `dw_mci_submit_data()` 决定 DMA 还是 PIO，最后由 DMA callback 或 PIO 读写函数设置 `EVENT_XFER_COMPLETE`。卡侧 data over 则由主 IRQ 设置 `EVENT_DATA_COMPLETE`。对应代码在 `kernel/drivers/mmc/host/dw_mmc.c:1384`、`kernel/drivers/mmc/host/dw_mmc.c:1187`、`kernel/drivers/mmc/host/dw_mmc.c:510`、`kernel/drivers/mmc/host/dw_mmc.c:2810`、`kernel/drivers/mmc/host/dw_mmc.c:3005`。

一句话总结：命令线决定“能不能开始”，数据线决定“内存侧搬没搬完”，data over 决定“卡侧事务是否真正结束”。

## 5. 推荐阅读顺序

第一遍只看大路径：`dw_mci_probe()` 注册 host，`dw_mci_ops` 暴露入口，`dw_mci_request()` 接收请求，`__dw_mci_start_request()` 启动命令/数据，`dw_mci_interrupt()` 产生事件，`dw_mci_tasklet_func()` 消费事件，`dw_mci_request_end()` 回调 MMC core。

第二遍再看分支：DMA/PIO、CMD11、SDIO IRQ、runtime PM、timeout timer、错误恢复。

第三遍才需要对照寄存器细节，例如 `CLKENA/CLKDIV/CTYPE/TMOUT/FIFOTH/INTMASK/RINTSTS/MINTSTS`。
