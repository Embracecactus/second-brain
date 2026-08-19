# 24. 外设驱动详解（二）：RTC / Timer / ADC

> 目标：继续通过 RTC、Timer、ADC 三个驱动，巩固"NuttX lower-half 封装 HAL"的读码方法。

---

## 24.1 驱动清单与位置

| 驱动 | 文件 | 设备节点 / 接口 | 核心 HAL |
|---|---|---|---|
| RTC | `chip/ap/bk7258_rtc.c` | `/dev/rtc0`、`up_rtc_initialize()` | `bk_rtc_*` |
| Timer | `chip/ap/bk7258_timer.c` | `/dev/timerN` | `bk_timer_*` |
| SARADC | `chip/ap/bk7258_saradc.c` | `/dev/adcN` | `bk_adc_*` |
| SDMADC | `chip/ap/bk7258_sdmadc.c` | `/dev/adcN` | `bk_adc_*` |

---

## 24.2 RTC 驱动详解

### 24.2.1 理论：RTC 是什么？

RTC（Real-Time Clock，实时时钟）是一颗即使主 CPU 休眠也能继续走时的时钟。它通常由独立电池或低功耗域供电，用来保存日期时间。

> 💡 **背景知识：RTC 和系统 tick 有什么区别？**
> 
> - 系统 tick：CPU 运行时的"心跳"，用于任务调度，休眠时可能停止。
> - RTC：独立硬件时钟，掉电/休眠也能继续计时，精度较低但持续运行。

---

### 24.2.2 源码解析

文件：`board/bk7258/chip/ap/bk7258_rtc.c`

ops 表：

```c
static const struct rtc_ops_s g_bk7258_rtc_ops =
{
  .rdtime   = bk7258_rtc_rdtime,
  .settime  = bk7258_rtc_settime,
  .hwrdsys  = bk7258_rtc_hwrdsys,
  .setalarm = bk7258_rtc_setalarm,
  ...
};
```

初始化：

```c
int up_rtc_initialize(void)
{
  // 注册 RTC lower-half
  rtc_initialize(0, &g_bk7258_rtc.dev);
}
```

方法映射：

| NuttX RTC 方法 | SDK 函数 |
|---|---|
| `rdtime` | `bk_rtc_get_time` |
| `settime` | `bk_rtc_set_time` |
| `setalarm` | `bk_rtc_register_alarm` |

---

### 24.2.3 实操：在 NSH 里读写 RTC

```bash
# 进入 NSH
ls /dev/rtc*      # 查看 RTC 设备

# 读取时间
date

# 设置时间(需权限)
date -s "2026-08-19 12:00:00"
```

---

## 24.3 Timer 驱动详解

### 24.3.1 理论：Timer 是什么？

Timer（硬件定时器）是 MCU 内部的外设，可以周期性地产生中断。和系统 tick 不同，通用 Timer 通常给应用层提供精确的定时服务（如 1ms、100us 一次回调）。

> 💡 **背景知识：Timer vs PWM 的区别**
> 
> - Timer：用来产生定时中断，不直接输出波形。
> - PWM：也是基于定时器，但额外把计数值输出到 GPIO，形成可调的方波。

---

### 24.3.2 源码解析

文件：`board/bk7258/chip/ap/bk7258_timer.c`

ops 表：

```c
static const struct timer_ops_s g_bk7258_timer_ops =
{
  .setup    = bk7258_timer_setup,
  .shutdown = bk7258_timer_shutdown,
  .start    = bk7258_timer_start,
  .stop     = bk7258_timer_stop,
  .ioctl    = bk7258_timer_ioctl,
};
```

关键问题：ISR 里重新装载定时器时，可能和任务上下文的 `stop()` 竞争。这个驱动用 `running` 标志做门闩：

```c
// 思想示意
stop() {
  // 先置 running=false,再停硬件
  running = false;
  bk_timer_stop(chan);
}

isr() {
  if (!running) return;       // 已 stop,不再重启
  // 处理回调
  if (reload) bk_timer_start(chan);  // re-arm 前再查 running
}
```

> ⚠️ 这是驱动并发编程的典型坑：ISR 里重新启动硬件，任务上下文同时 stop，必须用原子标志同步。

---

### 24.3.3 实操：用 Timer 做周期回调

```bash
# 进入 NSH
ls /dev/timer*

# 用 timer 工具测试(需 CONFIG_TIMER 支持)
timer -d /dev/timer0 -p 1000000   # 周期 1s,回调打印
```

---

## 24.4 SARADC 驱动详解

### 24.4.1 理论：ADC 是什么？

ADC（Analog-to-Digital Converter，模数转换器）把模拟电压（如 0~3.3V）转换成数字值（如 0~4095）。

- **SARADC**：逐次逼近型 ADC，速度快、精度中等，适合温度、电压、按键等采样。
- **SDMADC**：Sigma-Delta 调制 ADC，精度高、速度较慢，适合音频等。

> 💡 **背景知识：分辨率是什么意思？**
> 
> 12 位 ADC 能把电压分成 2^12 = 4096 份。如果参考电压是 3.3V，每份约 0.8mV。读到的数字值越大，代表电压越高。

---

### 24.4.2 源码解析

文件：`board/bk7258/chip/ap/bk7258_saradc.c`

ops 表：

```c
static const struct adc_ops_s g_bk7258_saradc_ops =
{
  .bind     = bk7258_saradc_bind,
  .reset    = bk7258_saradc_reset,
  .setup    = bk7258_saradc_setup,
  .shutdown = bk7258_saradc_shutdown,
  .read     = bk7258_saradc_read,
  .ioctl    = bk7258_saradc_ioctl,
};
```

SARADC 在 AP 侧是客户端，真正的 ADC 采样服务在 CP 侧（`bk7258_saradc_server.c`）。AP 通过 mailbox/RPTUN 发请求，CP 完成采样后返回结果。

```c
int bk7258_saradc_initialize(void)
{
  // 初始化本地 ADC 结构
  // 注册 /dev/adcN
  adc_register(BK7258_SARADC_DEVPATH, &g_bk7258_saradc.dev);
}
```

---

## 24.5 SDMADC 驱动详解

文件：`board/bk7258/chip/ap/bk7258_sdmadc.c`

和 SARADC 类似，也是 `adc_ops_s` 接口，但对应 Sigma-Delta 调制 ADC。

配置：

```kconfig
config BK7258_SDMADC
    bool "SDMADC driver"
    select ANALOG
    select ADC

config BK7258_SDMADC_CHAN
    int "SDMADC channel"
    default 1
    range 1 13
```

---

## 24.6 CP 侧 WDT 和 SARADC Server

虽然本文件主要讲 AP 驱动，但有两个 CP 侧驱动值得对照：

| CP 驱动 | 文件 | 作用 |
|---|---|---|
| WDT | `chip/cp/bk7258_wdt.c` | 注册 `/dev/watchdog0`，系统异常时复位 |
| SARADC server | `chip/cp/bk7258_saradc_server.c` | 启动 `bk_adc_driver_init()`，响应 AP 的 ADC 请求 |

WDT 是系统可靠性的重要保障：

```c
static const struct watchdog_ops_s g_bk7258_wdt_ops =
{
  .start    = bk7258_wdt_start,
  .stop     = bk7258_wdt_stop,
  .keepalive= bk7258_wdt_keepalive,
  ...
};
```

---

## 24.7 实操：读取 ADC 值

```bash
# 进入 NSH
ls /dev/adc*

# 用 adc 工具读取(需 CONFIG_ADC 支持)
adc -d /dev/adc0 -n 10   # 读 10 次
```

---

## 24.8 本节小结

- **RTC**：`rtc_ops_s` 封装 `bk_rtc_*`，提供日期时间和闹钟功能。
- **Timer**：`timer_ops_s` 封装 `bk_timer_*`；ISR re-arm 时要和 stop 竞争，用 `running` 门闩。
- **SARADC/SDMADC**：`adc_ops_s` 封装 `bk_adc_*`；AP 侧是客户端，CP 侧有 SARADC server。
- **WDT**：CP 侧驱动，注册 `/dev/watchdog0`。

---

## 底部导航

←上一篇：[23 外设驱动详解（一）I2C-PWM-GPIOE](./23-外设驱动详解-一(I2C-PWM-GPIOE).md) · 下一篇→：[25 跨核通信 RPTUN](./25-跨核通信-RPTUN.md) · ↑返回导航：[00 开始这里](./00-开始这里-导航与学习路径.md)

---

📂 **本文涉及源码路径**

- `$CONTEST/board/bk7258/chip/ap/bk7258_rtc.c`
- `$CONTEST/board/bk7258/chip/ap/bk7258_timer.c`
- `$CONTEST/board/bk7258/chip/ap/bk7258_saradc.c`
- `$CONTEST/board/bk7258/chip/ap/bk7258_sdmadc.c`
- `$CONTEST/board/bk7258/chip/cp/bk7258_wdt.c`
- `$CONTEST/board/bk7258/chip/cp/bk7258_saradc_server.c`
