---
tags:
  - embedded-linux
  - bsp
  - capstone
  - driver
  - i2c
  - interrupt
  - runtime-pm
  - sysfs
  - rockchip
category: embedded-linux
created: 2026-06-22
updated: 2026-06-22
status: active
soc: Rockchip RV1126B
kernel: Linux 6.1.141
---

# 阶段六：Capstone — 完整 BSP 驱动项目

> **JD对标**：驱动设计编码测试、技术文档、综合能力
>
> 这是整个学习路线的收尾项目。你将从零设计一个完整的 I2C 温度传感器驱动，集成前五个阶段的所有知识：设备模型 + 设备树 + 中断处理 + I2C 子系统 + 电源管理 + sysfs 接口。

---

## 一、项目概述

### 1.1 项目目标

编写一个完整的 I2C 温度传感器驱动，具备以下功能：

| 功能 | 技术点 | 来源阶段 |
|------|--------|---------|
| I2C 通信 | `i2c_smbus_read_byte_data` | 阶段四 |
| 设备树匹配 | `of_match_table` + DTS 节点 | 阶段二 |
| GPIO 中断 + workqueue | `request_threaded_irq` + `schedule_work` | 阶段三 |
| Runtime PM | `pm_runtime_get/put` + autosuspend | 阶段五 |
| sysfs 接口 | `device_attribute` + `DEVICE_ATTR` | 阶段二 |
| 字符设备 (可选) | `cdev` + `file_operations` | 阶段二 |
| 错误处理 + 调试 | Ftrace + Lockdep + dmesg | 附录A |

### 1.2 硬件假设

使用一个通用的 I2C 温度传感器模型（类似 LM75 / TMP102）：
- I2C 7-bit 地址：0x48
- 温度寄存器：0x00 (16-bit, 2 字节)
- 配置寄存器：0x01 (8-bit)
- 中断引脚：GPIO0 pin 5，低电平有效（温度超阈值时触发）

> 如果板上没有真实传感器，可以用一个 I2C EEPROM (如 24C02) 模拟，或用"虚拟设备"方式（不实际读写 I2C，返回模拟数据）。

---

## 二、设备树

```dts
/* 在 rv1126b-sportcam.dts 或板级 DTS 中添加 */

/ {
    /* 传感器温度阈值 (mC 毫摄氏度) */
    temp_threshold: temp-threshold {
        compatible = "my,temp-threshold";
        threshold-millicelsius = <50000>;  /* 50 度报警 */
        hyst-millicelsius = <2000>;        /* 2 度滞回 */
    };
};

&i2c2 {
    status = "okay";

    temp_sensor: temperature-sensor@48 {
        compatible = "my,temp-sensor";
        reg = <0x48>;                          /* I2C 地址 */

        /* 中断: 温度超阈值时 GPIO0 pin5 拉低 */
        interrupt-parent = <&gpio0>;
        interrupts = <5 IRQ_TYPE_LEVEL_LOW>;

        /* 供电 regulator */
        vdd-supply = <&vcc3v3>;

        /* 时钟 (如果传感器需要) */
        clocks = <&cru CLK_I2C2>;
        clock-names = "bus";

        /* 电源域 (如果适用) */
        /* power-domains = <&power RV1126B_PD_VDO>; */

        /* 默认采样间隔 (ms) */
        sample-interval-ms = <1000>;

        status = "okay";
    };
};
```

---

## 三、完整驱动源码 (temp_sensor.c)

```c
#include <linux/module.h>
#include <linux/i2c.h>
#include <linux/of.h>
#include <linux/of_device.h>
#include <linux/of_irq.h>
#include <linux/interrupt.h>
#include <linux/workqueue.h>
#include <linux/gpio.h>
#include <linux/of_gpio.h>
#include <linux/sysfs.h>
#include <linux/device.h>
#include <linux/pm_runtime.h>
#include <linux/regulator/consumer.h>
#include <linux/clk.h>
#include <linux/mutex.h>
#include <linux/atomic.h>
#include <linux/string.h>

#define TEMP_REG         0x00    /* 温度寄存器 (2 bytes) */
#define CONFIG_REG       0x01    /* 配置寄存器 (1 byte) */
#define THYST_REG        0x02    /* 滞回阈值 */
#define TOS_REG          0x03    /* 过温阈值 */

#define TEMP_SENSOR_AUTOSUSPEND_DELAY  2000  /* 2 秒 */

struct temp_sensor_data {
    struct i2c_client *client;
    struct device *dev;

    /* I2C 通信保护 */
    struct mutex i2c_lock;

    /* 中断 + workqueue */
    int irq;
    struct work_struct alert_work;
    atomic_t alert_count;

    /* 电源管理 */
    struct regulator *vdd;
    struct clk *clk;

    /* 配置参数 (从 DTS) */
    int sample_interval_ms;
    int threshold_mc;
    int hyst_mc;

    /* 缓存温度值 */
    int cached_temp_mc;          /* 毫摄氏度 */
    bool alert_active;
};

/* ===== I2C 读写函数 ===== */

static int temp_read_reg16(struct temp_sensor_data *data, u8 reg, int *val)
{
    s32 msb, lsb;
    int ret = 0;

    mutex_lock(&data->i2c_lock);
    msb = i2c_smbus_read_byte_data(data->client, reg);
    if (msb < 0) { ret = msb; goto out; }
    lsb = i2c_smbus_read_byte_data(data->client, reg + 1);
    if (lsb < 0) { ret = lsb; goto out; }
    *val = (msb << 8) | lsb;
out:
    mutex_unlock(&data->i2c_lock);
    return ret;
}

static int temp_write_reg8(struct temp_sensor_data *data, u8 reg, u8 val)
{
    int ret;
    mutex_lock(&data->i2c_lock);
    ret = i2c_smbus_write_byte_data(data->client, reg, val);
    mutex_unlock(&data->i2c_lock);
    return ret;
}

/* 原始值转毫摄氏度 (LM75 格式: 0.5°C/bit, 16-bit signed) */
static int raw_to_millicelsius(int raw)
{
    /* raw 是 16-bit, 高 11 位有效, 分辨率 0.125°C */
    int temp = (raw >> 5) & 0x7FF;
    if (temp & 0x400)       /* 负数 (11-bit sign extend) */
        temp |= 0xFFFFF800;
    return temp * 125;      /* 0.125°C = 125 mC */
}

/* ===== 温度读取 ===== */

static int temp_read_temperature(struct temp_sensor_data *data, int *temp_mc)
{
    int raw, ret;

    /* Runtime PM: 确保设备活跃 */
    pm_runtime_get_sync(data->dev);

    ret = temp_read_reg16(data, TEMP_REG, &raw);
    if (ret)
        goto out;

    *temp_mc = raw_to_millicelsius(raw);
    data->cached_temp_mc = *temp_mc;

out:
    pm_runtime_put_autosuspend(data->dev);
    return ret;
}

/* ===== sysfs 接口 ===== */

static ssize_t temp_show(struct device *dev,
                         struct device_attribute *attr, char *buf)
{
    struct temp_sensor_data *data = dev_get_drvdata(dev);
    int temp_mc, ret;

    ret = temp_read_temperature(data, &temp_mc);
    if (ret)
        return sprintf(buf, "error: %d\n", ret);

    return sprintf(buf, "%d.%03d\n", temp_mc / 1000, abs(temp_mc % 1000));
}

static ssize_t threshold_show(struct device *dev,
                              struct device_attribute *attr, char *buf)
{
    struct temp_sensor_data *data = dev_get_drvdata(dev);
    return sprintf(buf, "%d\n", data->threshold_mc);
}

static ssize_t threshold_store(struct device *dev,
                               struct device_attribute *attr,
                               const char *buf, size_t count)
{
    struct temp_sensor_data *data = dev_get_drvdata(dev);
    int val, ret;

    ret = kstrtoint(buf, 10, &val);
    if (ret)
        return ret;

    data->threshold_mc = val;

    /* 写入传感器硬件 (LM75 TOS 寄存器, 9-bit) */
    pm_runtime_get_sync(dev);
    u8 raw = (val / 500) & 0xFF;  /* 简化: 0.5°C/bit */
    temp_write_reg8(data, TOS_REG, raw);
    pm_runtime_put_autosuspend(dev);

    return count;
}

static ssize_t alert_count_show(struct device *dev,
                                struct device_attribute *attr, char *buf)
{
    struct temp_sensor_data *data = dev_get_drvdata(dev);
    return sprintf(buf, "%d\n", atomic_read(&data->alert_count));
}

static ssize_t alert_status_show(struct device *dev,
                                 struct device_attribute *attr, char *buf)
{
    struct temp_sensor_data *data = dev_get_drvdata(dev);
    return sprintf(buf, "%s\n", data->alert_active ? "ALERT" : "OK");
}

static DEVICE_ATTR_RO(temp);
static DEVICE_ATTR_RW(threshold);
static DEVICE_ATTR_RO(alert_count);
static DEVICE_ATTR_RO(alert_status);

static struct attribute *temp_sensor_attrs[] = {
    &dev_attr_temp.attr,
    &dev_attr_threshold.attr,
    &dev_attr_alert_count.attr,
    &dev_attr_alert_status.attr,
    NULL,
};
ATTRIBUTE_GROUPS(temp_sensor);

/* ===== 中断 + Workqueue ===== */

static void temp_alert_work(struct work_struct *work)
{
    struct temp_sensor_data *data =
        container_of(work, struct temp_sensor_data, alert_work);
    int temp_mc, ret;

    atomic_inc(&data->alert_count);

    /* 在 workqueue 中读取温度 (进程上下文, 可睡眠, 可 I2C) */
    ret = temp_read_temperature(data, &temp_mc);
    if (ret) {
        dev_err(data->dev, "alert: failed to read temp: %d\n", ret);
        return;
    }

    data->alert_active = (temp_mc >= data->threshold_mc);

    dev_info(data->dev, "temperature alert! temp=%d mC, threshold=%d mC\n",
             temp_mc, data->threshold_mc);

    /* 这里可以: 发送 uevent / 通知用户态 / 触发降温策略 */
    if (data->alert_active) {
        kobject_uevent(&data->dev->kobj, KOBJ_CHANGE);
    }
}

static irqreturn_t temp_irq_handler(int irq, void *dev_id)
{
    struct temp_sensor_data *data = dev_id;

    /* top half: 只调度 work (不能在这里做 I2C 读取) */
    schedule_work(&data->alert_work);

    return IRQ_HANDLED;
}

/* ===== Runtime PM 回调 ===== */

static int temp_runtime_suspend(struct device *dev)
{
    struct temp_sensor_data *data = dev_get_drvdata(dev);

    /* 关闭时钟 (节省功耗) */
    if (data->clk)
        clk_disable_unprepare(data->clk);

    /* 注意: 不关闭 regulator, 否则 I2C 通信会失败 */
    /* 真正断电需要在 remove 或系统 suspend 中做 */

    dev_dbg(dev, "runtime suspend\n");
    return 0;
}

static int temp_runtime_resume(struct device *dev)
{
    struct temp_sensor_data *data = dev_get_drvdata(dev);
    int ret;

    /* 恢复时钟 */
    if (data->clk) {
        ret = clk_prepare_enable(data->clk);
        if (ret) {
            dev_err(dev, "failed to enable clock: %d\n", ret);
            return ret;
        }
    }

    dev_dbg(dev, "runtime resume\n");
    return 0;
}

static const struct dev_pm_ops temp_sensor_pm_ops = {
    SET_RUNTIME_PM_OPS(temp_runtime_suspend, temp_runtime_resume, NULL)
    SET_SYSTEM_SLEEP_PM_OPS(pm_runtime_force_suspend,
                            pm_runtime_force_resume)
};

/* ===== Probe / Remove ===== */

static int temp_sensor_probe(struct i2c_client *client,
                              const struct i2c_device_id *id)
{
    struct device *dev = &client->dev;
    struct temp_sensor_data *data;
    int ret, temp_mc;

    data = devm_kzalloc(dev, sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    data->client = client;
    data->dev = dev;
    mutex_init(&data->i2c_lock);
    atomic_set(&data->alert_count, 0);
    INIT_WORK(&data->alert_work, temp_alert_work);

    i2c_set_clientdata(client, data);

    /* 1. 读取 DTS 属性 */
    if (of_property_read_u32(dev->of_node, "sample-interval-ms",
                              &data->sample_interval_ms))
        data->sample_interval_ms = 1000;

    data->threshold_mc = 50000;  /* 默认 50°C */
    data->hyst_mc = 2000;        /* 默认 2°C 滞回 */

    /* 2. 获取 regulator */
    data->vdd = devm_regulator_get(dev, "vdd");
    if (IS_ERR(data->vdd)) {
        ret = PTR_ERR(data->vdd);
        if (ret != -EPROBE_DEFER)
            dev_err(dev, "failed to get vdd: %d\n", ret);
        return ret;
    }
    ret = regulator_enable(data->vdd);
    if (ret) {
        dev_err(dev, "failed to enable vdd: %d\n", ret);
        return ret;
    }

    /* 3. 获取时钟 */
    data->clk = devm_clk_get(dev, "bus");
    if (IS_ERR(data->clk)) {
        ret = PTR_ERR(data->clk);
        if (ret == -ENOENT)
            data->clk = NULL;  /* 传感器可能不需要时钟 */
        else
            goto err_regulator;
    }
    if (data->clk) {
        ret = clk_prepare_enable(data->clk);
        if (ret)
            goto err_regulator;
    }

    /* 4. 验证 I2C 通信 */
    ret = temp_read_temperature(data, &temp_mc);
    if (ret) {
        dev_err(dev, "I2C communication failed: %d\n", ret);
        goto err_clk;
    }
    dev_info(dev, "sensor initialized, temp=%d.%03d C\n",
             temp_mc / 1000, abs(temp_mc % 1000));

    /* 5. 注册中断 */
    data->irq = client->irq;  /* 从 DTS interrupts 自动解析 */
    if (data->irq > 0) {
        ret = devm_request_threaded_irq(dev, data->irq,
                                        temp_irq_handler,  /* top half */
                                        NULL,               /* 无 thread_fn */
                                        IRQF_TRIGGER_FALLING | IRQF_ONESHOT,
                                        "temp-sensor", data);
        if (ret) {
            dev_err(dev, "failed to request IRQ %d: %d\n",
                    data->irq, ret);
            goto err_clk;
        }
        dev_info(dev, "IRQ %d registered\n", data->irq);
    }

    /* 6. 创建 sysfs 接口 */
    ret = devm_device_add_groups(dev, temp_sensor_groups);
    if (ret) {
        dev_err(dev, "failed to create sysfs: %d\n", ret);
        goto err_clk;
    }

    /* 7. 启用 Runtime PM */
    pm_runtime_enable(dev);
    pm_runtime_set_autosuspend_delay(dev,
                                      TEMP_SENSOR_AUTOSUSPEND_DELAY);
    pm_runtime_use_autosuspend(dev);

    /* 初始状态: active, 然后释放给 autosuspend 管理 */
    pm_runtime_mark_last_busy(dev);
    pm_runtime_put_autosuspend(dev);

    dev_info(dev, "temperature sensor driver loaded\n");
    return 0;

err_clk:
    if (data->clk)
        clk_disable_unprepare(data->clk);
err_regulator:
    regulator_disable(data->vdd);
    return ret;
}

static int temp_sensor_remove(struct i2c_client *client)
{
    struct temp_sensor_data *data = i2c_get_clientdata(client);
    struct device *dev = &client->dev;

    /* 取消待处理的 work */
    cancel_work_sync(&data->alert_work);

    /* 恢复到 active 状态再清理 */
    pm_runtime_get_sync(dev);
    pm_runtime_put_noidle(dev);
    pm_runtime_disable(dev);

    /* 关闭时钟和电源 */
    if (data->clk)
        clk_disable_unprepare(data->clk);
    regulator_disable(data->vdd);

    dev_info(dev, "temperature sensor driver removed\n");
    return 0;
}

/* ===== 设备匹配表 ===== */

static const struct of_device_id temp_sensor_match[] = {
    { .compatible = "my,temp-sensor" },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, temp_sensor_match);

static const struct i2c_device_id temp_sensor_id[] = {
    { "temp-sensor", 0 },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(i2c, temp_sensor_id);

/* ===== 驱动结构 ===== */

static struct i2c_driver temp_sensor_driver = {
    .probe   = temp_sensor_probe,
    .remove  = temp_sensor_remove,
    .id_table = temp_sensor_id,
    .driver  = {
        .name = "temp-sensor",
        .of_match_table = of_match_ptr(temp_sensor_match),
        .pm = &temp_sensor_pm_ops,
    },
};

module_i2c_driver(temp_sensor_driver);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("BSP Student");
MODULE_DESCRIPTION("I2C Temperature Sensor Driver - Capstone Project");
```

---

## 四、Makefile

```makefile
obj-m += temp_sensor.o

KERNEL_DIR := $(SDK_ROOT)/kernel-6.1
ARCH := arm64
CROSS_COMPILE := aarch64-none-linux-gnu-

all:
	make -C $(KERNEL_DIR) ARCH=$(ARCH) CROSS_COMPILE=$(CROSS_COMPILE) \
	     M=$(PWD) modules

clean:
	make -C $(KERNEL_DIR) ARCH=$(ARCH) M=$(PWD) clean
```

---

## 五、实验 1：编译 & 部署 & 验证

### 5.1 编译

```bash
# PC 端
export PATH=$PWD/prebuilts/gcc/linux-x86/aarch64/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin:$PATH
make
# 产物: temp_sensor.ko
```

### 5.2 部署

```bash
scp temp_sensor.ko rooter@192.168.1.109:/tmp/

# 板端
sudo insmod /tmp/temp_sensor.ko
# 预期 dmesg:
#   "sensor initialized, temp=XX.XXX C"
#   "IRQ XX registered"
#   "temperature sensor driver loaded"
```

### 5.3 验证清单

```bash
# 1. sysfs 接口
ls /sys/bus/i2c/devices/2-0048/
# 预期: temp, threshold, alert_count, alert_status

# 2. 读取温度
cat /sys/bus/i2c/devices/2-0048/temp
# 预期: "XX.XXX" (摄氏度)

# 3. 设置阈值
echo "60000" | sudo tee /sys/bus/i2c/devices/2-0048/threshold

# 4. 查看阈值
cat /sys/bus/i2c/devices/2-0048/threshold
# 预期: 60000

# 5. 查看中断计数 (初始)
cat /sys/bus/i2c/devices/2-0048/alert_count
# 预期: 0

# 6. 查看告警状态
cat /sys/bus/i2c/devices/2-0048/alert_status
# 预期: OK

# 7. 查看 Runtime PM 状态
cat /sys/bus/i2c/devices/2-0048/power/runtime_status
# 预期: suspended (2 秒后自动 suspend)

# 8. 触发温度读取 (自动 resume)
cat /sys/bus/i2c/devices/2-0048/temp
cat /sys/bus/i2c/devices/2-0048/power/runtime_status
# 预期: active (刚读完)

# 9. 等待 2 秒后检查
sleep 3
cat /sys/bus/i2c/devices/2-0048/power/runtime_status
# 预期: suspended

# 10. 查看设备绑定
ls -la /sys/bus/i2c/devices/2-0048/driver
# 预期: -> ../../../../bus/i2c/drivers/temp-sensor

# 11. 卸载
sudo rmmod temp_sensor
# 预期: "temperature sensor driver removed"
```

---

## 六、实验 2：Ftrace 验证驱动正确性

### 6.1 追踪 probe 流程

```bash
echo function_graph | sudo tee /sys/kernel/tracing/current_tracer
echo '*temp_sensor*|*i2c_smbus*|*pm_runtime*|*regulator*' | \
  sudo tee /sys/kernel/tracing/set_ftrace_filter
echo 128 | sudo tee /sys/kernel/tracing/max_graph_depth

echo | sudo tee /sys/kernel/tracing/trace
sudo insmod /tmp/temp_sensor.ko
sudo cat /sys/kernel/tracing/trace > /tmp/probe_trace.log
echo nop | sudo tee /sys/kernel/tracing/current_tracer
```

### 6.2 验证点

1. `temp_sensor_probe` → `i2c_smbus_read_byte_data` (I2C 通信正常)
2. `pm_runtime_enable` → `pm_runtime_put_autosuspend` (PM 初始化正确)
3. `regulator_enable` (电源使能)
4. `request_threaded_irq` (中断注册)

### 6.3 追踪温度读取的 PM 路径

```bash
echo function_graph | sudo tee /sys/kernel/tracing/current_tracer
echo '*pm_runtime*|*temp_read*|*i2c_smbus*' | \
  sudo tee /sys/kernel/tracing/set_ftrace_filter

echo | sudo tee /sys/kernel/tracing/trace
cat /sys/bus/i2c/devices/2-0048/temp
sudo cat /sys/kernel/tracing/trace > /tmp/read_trace.log
echo nop | sudo tee /sys/kernel/tracing/current_tracer
```

预期调用链：
```
sysfs read → temp_show()
  → pm_runtime_get_sync()     (触发 runtime_resume)
    → clk_prepare_enable()
  → temp_read_reg16()
    → i2c_smbus_read_byte_data()
  → pm_runtime_put_autosuspend()  (2 秒后触发 runtime_suspend)
    → clk_disable_unprepare()
```

---

## 七、实验 3：中断触发测试

### 7.1 模拟中断

如果没有真实传感器，可以模拟 GPIO 中断：

```bash
# 方法 1: 如果有按键连接到 GPIO0 pin 5
# 按键即可触发中断

# 方法 2: 用 sysfs gpio 手动触发 (需要 GPIO 支持)
echo 5 > /sys/class/gpio/export
echo falling > /sys/class/gpio/gpio5/edge
# 短接 pin5 到 GND

# 方法 3: 修改驱动, 用 timer 模拟周期性告警
```

### 7.2 验证中断处理

```bash
# 触发中断后检查
cat /sys/bus/i2c/devices/2-0048/alert_count
# 预期: 计数增加

cat /sys/bus/i2c/devices/2-0048/alert_status
# 预期: ALERT (如果温度超过阈值)

dmesg | tail -5
# 预期: "temperature alert! temp=XX mC, threshold=XX mC"
```

---

## 八、实验 4：Lockdep 验证

### 8.1 确认无锁问题

```bash
# 确保 Lockdep 启用
zcat /proc/config.gz | grep LOCKDEP
# 预期: CONFIG_LOCKDEP=y

# 加载驱动并操作
sudo insmod /tmp/temp_sensor.ko
for i in $(seq 1 100); do
    cat /sys/bus/i2c/devices/2-0048/temp &
done
wait

# 检查 dmesg 是否有锁问题
dmesg | grep -i "lock\|deadlock\|inconsistent"
# 预期: 无输出 (无锁问题)

# 检查 Lockdep 统计
cat /proc/lockdep_stats
# 预期: 无 "soft-safe → unsafe" 等异常
```

---

## 九、实验 5：性能基准

### 9.1 测量温度读取延迟

```bash
# 测量 100 次读取的总耗时
time for i in $(seq 1 100); do
    cat /sys/bus/i2c/devices/2-0048/temp > /dev/null
done

# 预期: 100 次 ~500ms-1s (每次 ~5-10ms, 含 I2C 传输 + PM 开销)
```

### 9.2 对比 Runtime PM 开销

```bash
# 强制设备常开 (禁用 autosuspend)
echo on | sudo tee /sys/bus/i2c/devices/2-0048/power/control

time for i in $(seq 1 100); do
    cat /sys/bus/i2c/devices/2-0048/temp > /dev/null
done
# 预期: 更快 (无 PM resume/suspend 开销)

# 恢复 autosuspend
echo auto | sudo tee /sys/bus/i2c/devices/2-0048/power/control
```

---

## 十、驱动设计文档模板

> 面试和实际工作中，驱动开发需要配套设计文档。以下是模板：

```markdown
# TEMP-SENSOR 驱动设计文档

## 1. 概述
- 功能: I2C 温度传感器驱动
- 硬件: 通用 LM75 兼容传感器 @ I2C 地址 0x48
- 接口: sysfs (/sys/bus/i2c/devices/X-0048/)

## 2. 设备树绑定
- compatible: "my,temp-sensor"
- reg: I2C 7-bit 地址
- interrupts: GPIO 中断 (温度告警)
- vdd-supply: 供电 regulator
- sample-interval-ms: 采样间隔

## 3. 软件架构
- 驱动类型: i2c_driver
- 并发保护: mutex (I2C 传输), atomic (中断计数)
- 中断处理: top half (schedule_work) + workqueue (I2C 读取)
- 电源管理: Runtime PM (autosuspend 2s)
- 用户接口: sysfs (temp, threshold, alert_count, alert_status)

## 4. 电源管理
- resume: clk_prepare_enable
- suspend: clk_disable_unprepare
- 总线时钟由 I2C 控制器驱动管理, 传感器时钟由本驱动管理

## 5. 测试验证
- I2C 通信: i2cdetect + i2cget 验证
- 中断: GPIO 触发 + dmesg 确认
- PM: /sys/.../power/runtime_status 观察
- 并发: 100 并发读取 + Lockdep 无报错
- 性能: 单次读取 ~5ms (含 PM 开销), 常开 ~2ms

## 6. 已知限制
- 不支持 DMA (I2C 传输量小, 无需)
- 不支持 hibernation (传感器无状态保存需求)
- 告警只发 uevent, 不实现主动降温策略
```

---

## 十一、Code Review 检查清单

在实际开发中，提交代码前对照以下清单：

| 检查项 | 标准 | 通过 |
|--------|------|------|
| 内存泄漏 | 所有 `devm_` 分配, probe 失败时自动释放 | |
| 锁一致性 | mutex 配对使用, 无中断上下文 mutex | |
| 锁排序 | 多锁场景, 获取顺序一致 | |
| 错误处理 | 所有可能失败的调用都检查返回值 | |
| Runtime PM | pm_runtime_enable + 正确 get/put 配对 | |
| 中断安全 | top half 无睡眠操作 | |
| 资源释放 | remove 中释放所有资源 (时钟/regulator/中断) | |
| sysfs 接口 | 属性读写有边界检查 | |
| DTS 兼容性 | of_match_table 与 DTS compatible 一致 | |
| 代码风格 | checkpatch.pl 无 WARNING | |
| 注释 | 复杂逻辑有注释, 无冗余注释 | |
| LICENSE | MODULE_LICENSE 正确声明 | |

### 验证命令

```bash
# checkpatch 检查代码风格
./kernel-6.1/scripts/checkpatch.pl --no-tree -f temp_sensor.c
# 预期: 无 ERROR, 尽量无 WARNING

# 模块加载验证
sudo insmod temp_sensor.ko
dmesg | grep temp_sensor
sudo rmmod temp_sensor
```

---

## 十二、思考题

1. 这个驱动中 `mutex_lock` 保护的是 I2C 传输。如果两个用户进程同时读 `temp` sysfs 属性，会发生什么？mutex 如何防止竞态？

2. `request_threaded_irq` 的 top half 中只调用了 `schedule_work`，没有使用 `thread_fn`。如果改为用 `thread_fn` (不调 `schedule_work`)，有什么区别？哪种更合适？

3. Runtime PM 的 `pm_runtime_put_autosuspend` 设置了 2 秒延迟。如果用户每 1 秒读一次温度，设备会 suspend 吗？这对功耗有什么影响？

4. remove 函数中为什么先 `pm_runtime_get_sync` 再 `pm_runtime_put_noidle` 再 `pm_runtime_disable`？直接 `pm_runtime_disable` 会怎样？

5. 如果这个传感器需要支持多实例（同型 I2C 总线上挂 2 个传感器，地址 0x48 和 0x49），驱动的哪些部分需要修改？DTS 怎么配置？

---

## 十三、学习路线总结

### 完成状态

| 阶段 | JD 覆盖 | 文档 | 状态 |
|------|---------|------|------|
| 一 | Bootloader 移植 + 启动优化 | `bsp-boot-flow.md` | ✅ |
| 二 | 设备模型 + 设备树 + 驱动基础 | `bsp-device-model-dtb.md` | ✅ |
| 三 | 中断处理 + 并发 + 稳定性 | `bsp-interrupt-concurrency.md` | ✅ |
| 四 | UART/I2C/SPI 外设驱动 | `bsp-peripheral-drivers.md` | ✅ |
| 五 | 电源管理 + 性能调优 | `bsp-power-management.md` | ✅ |
| 六 | 综合驱动项目 + 技术文档 | `bsp-capstone-driver.md` | ✅ |
| 附录A | 调试工具 (Ftrace/Lockdep) | `kernel-debug-env.md` | ✅ |
| 附录B | USB/V4L2 驱动 | `v4l2-isp-deep-dive.md` | ✅ |
| 附录C | 硬件编解码 (MPP) | `mpp-hardware-codec.md` | ✅ |

### JD 覆盖率

| JD 要求 | 覆盖程度 |
|---------|---------|
| 嵌入式 Linux 系统架构 | ✅ 完整 |
| Bootloader/Kernel 移植 | ✅ 阶段一 |
| C 语言 + 调试能力 | ✅ 全程 |
| UART/I2C/SPI 协议 | ✅ 阶段四 |
| USB 协议 | ✅ 附录B |
| Linux 设备模型 | ✅ 阶段二 |
| 中断处理 | ✅ 阶段三 |
| 电源管理 | ✅ 阶段五 |
| 启动优化 | ✅ 阶段一 |
| 稳定性 + 性能调优 | ✅ 阶段三 + 五 + 附录A |
| 技术文档 | ✅ 全程 |

---

## 相关笔记

- [[MOC-嵌入式Linux]] — 学习路线总览
- [[bsp-boot-flow]] — 阶段一：Bootloader + 启动流程
- [[bsp-device-model-dtb]] — 阶段二：设备模型 + 设备树
- [[bsp-interrupt-concurrency]] — 阶段三：中断处理 + 并发
- [[bsp-peripheral-drivers]] — 阶段四：外设驱动 I2C/SPI/UART
- [[bsp-power-management]] — 阶段五：电源管理
- [[kernel-debug-env]] — 附录A：内核调试环境
- [[v4l2-isp-deep-dive]] — 附录B：V4L2/UVC 驱动
- [[mpp-hardware-codec]] — 附录C：MPP 硬件编解码
