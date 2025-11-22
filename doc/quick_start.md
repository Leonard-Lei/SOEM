# SOEM 快速构建指南

## 目录
- [项目简介](#项目简介)
- [系统要求](#系统要求)
- [快速开始](#快速开始)
- [构建选项](#构建选项)
- [编译步骤详解](#编译步骤详解)
- [运行示例程序](#运行示例程序)
- [常见问题](#常见问题)
- [RTOS 支持](#rtos-支持)
- [Linux PREEMPT_RT 使用指南](#linux-preempt_rt-使用指南)
- [进阶配置](#进阶配置)

## 项目简介

SOEM (Simple Open EtherCAT Master) 是一个用于开发 EtherCAT 主站设备的开源软件库。该库专为嵌入式系统的实时通信而设计，具有轻量级架构，资源消耗最小，适合资源受限的环境。SOEM 可以在 Linux 和 Windows 系统上使用。

**主要特性：**
- 轻量级设计，适合嵌入式系统
- 支持实时通信
- 跨平台支持（Linux/Windows/RTOS）
- 提供完整的 EtherCAT 主站功能
- 包含丰富的示例程序
- 支持多种实时操作系统（RTOS）

## 系统要求

### 必需组件

1. **CMake** 3.28 或更高版本
   ```bash
   cmake --version  # 检查版本
   ```

2. **C 编译器**
   - Linux: GCC 或 Clang
   - Windows: MSVC 或 MinGW

3. **构建工具**
   - Linux: make 或 ninja
   - Windows: Visual Studio 或 MinGW

4. **系统库**（Linux）
   - pthread (POSIX 线程库)
   - rt (实时扩展库)

### 可选组件

- **Python 3**（用于 eni_test 示例）
- **网络适配器**（用于实际 EtherCAT 通信）
- **PREEMPT_RT 内核**（Linux 实时性增强，推荐用于高精度应用）

## 快速开始

### 方法 1：标准构建（推荐）

```bash
# 1. 进入项目根目录
cd /path/to/SOEM

# 2. 创建构建目录
mkdir -p build
cd build

# 3. 配置项目（生成构建文件）
cmake ..

# 4. 编译项目
cmake --build .

# 5. （可选）安装到默认位置（build/install）
cmake --install .
```

### 方法 2：使用 CMake 预设

如果项目包含 `CMakePresets.json`：

```bash
# 查看可用的预设
cmake --list-presets

# 使用预设配置
cmake --preset <preset-name>

# 构建
cmake --build --preset <preset-name>
```

### 方法 3：一行命令构建

```bash
mkdir -p build && cd build && cmake .. && cmake --build .
```

## 构建选项

### 基本选项

在运行 `cmake ..` 时，可以通过 `-D` 参数设置以下选项：

#### 库类型选项

```bash
# 构建共享库（默认是静态库）
cmake -DBUILD_SHARED_LIBS=ON ..

# 构建静态库（默认）
cmake -DBUILD_SHARED_LIBS=OFF ..
```

#### 示例程序选项

```bash
# 构建示例程序（默认开启）
cmake -DSOEM_BUILD_SAMPLES=ON ..

# 不构建示例程序
cmake -DSOEM_BUILD_SAMPLES=OFF ..
```

#### 调试选项

```bash
# 启用调试输出
cmake -DEC_DEBUG=ON ..
```

#### 构建类型

```bash
# Debug 构建（包含调试信息）
cmake -DCMAKE_BUILD_TYPE=Debug ..

# Release 构建（优化）
cmake -DCMAKE_BUILD_TYPE=Release ..

# RelWithDebInfo 构建（优化+调试信息，默认）
cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo ..

# MinSizeRel 构建（最小体积）
cmake -DCMAKE_BUILD_TYPE=MinSizeRel ..
```

#### 安装路径

```bash
# 指定安装路径
cmake -DCMAKE_INSTALL_PREFIX=/usr/local ..
```

### 高级配置选项

SOEM 提供了许多可配置的参数，可以在 CMake 配置时修改：

#### 缓冲区大小配置

```bash
# 标准帧缓冲区大小（默认：EC_MAXECATFRAME，1518字节）
cmake -DEC_BUFSIZE=1518 ..

# 每个通道的帧缓冲区数量（默认：16）
cmake -DEC_MAXBUF=16 ..
```

#### 从站数量配置

```bash
# 最大从站数量（默认：200）
cmake -DEC_MAXSLAVE=200 ..

# 最大组数量（默认：2）
cmake -DEC_MAXGROUP=2 ..
```

#### 超时配置

```bash
# 帧返回超时（微秒，默认：2000）
cmake -DEC_TIMEOUTRET=2000 ..

# 状态检查超时（微秒，默认：2000000）
cmake -DEC_TIMEOUTSTATE=2000000 ..
```

#### MAC 地址配置

```bash
# 主 MAC 地址
cmake -DEC_PRIMARY_MAC="01:01:01:01:01:01" ..

# 次 MAC 地址
cmake -DEC_SECONDARY_MAC="04:04:04:04:04:04" ..
```

### 组合使用多个选项

```bash
cmake \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_SHARED_LIBS=ON \
  -DEC_DEBUG=ON \
  -DSOEM_BUILD_SAMPLES=ON \
  -DCMAKE_INSTALL_PREFIX=/usr/local \
  ..
```

## 编译步骤详解

### 步骤 1：准备构建目录

```bash
# 创建独立的构建目录（推荐，保持源码目录干净）
mkdir -p build
cd build
```

### 步骤 2：配置项目

```bash
cmake ..
```

**配置过程会：**
- 检测系统环境（编译器、库等）
- 根据平台选择相应的 OSAL 和 OSHW 实现
- 生成构建文件（Makefile 或 Visual Studio 项目文件）
- 配置头文件（如 `ec_options.h`）

**Linux 平台：**
- 自动选择 `osal/linux/` 和 `oshw/linux/`
- 链接 `pthread` 和 `rt` 库

**Windows 平台：**
- 自动选择 `osal/win32/` 和 `oshw/win32/`
- 使用 WinPcap 库

### 步骤 3：编译

```bash
# 使用 CMake 构建
cmake --build .

# 或使用原生构建工具
make          # Linux/Unix
# 或
ninja         # 如果使用 Ninja 生成器
```

**编译输出：**
- 库文件：`libsoem.a`（静态库）或 `libsoem.so`（共享库）
- 示例程序：在 `samples/` 子目录下

### 步骤 4：安装（可选）

```bash
cmake --install .
```

**默认安装位置：** `build/install/`

**安装内容：**
- 库文件：`install/lib/`
- 头文件：`install/include/soem/`
- 示例程序：`install/bin/`（如果构建了示例）
- CMake 配置文件：`install/cmake/`

## 运行示例程序

### 1. ec_sample - 基本 EtherCAT 示例

```bash
# 查找可用的网络适配器
./samples/ec_sample/ec_sample

# 运行示例（需要指定网络接口）
./samples/ec_sample/ec_sample eth0

# 指定周期时间（微秒）
./samples/ec_sample/ec_sample eth0 1000
```

**功能：**
- 扫描并配置 EtherCAT 从站
- 周期性交换过程数据
- 演示基本的主站功能

### 2. slaveinfo - 从站信息工具

```bash
./samples/slaveinfo/slaveinfo eth0
```

**功能：**
- 显示所有从站的详细信息
- 显示 EEPROM 内容
- 显示 PDO 映射

### 3. eepromtool - EEPROM 工具

```bash
./samples/eepromtool/eepromtool eth0 <slave_number>
```

**功能：**
- 读取/写入从站 EEPROM
- 显示 EEPROM 内容

### 4. simple_ng - 简单示例（无组）

```bash
./samples/simple_ng/simple_ng eth0
```

**功能：**
- 简化的示例，不使用组功能

### 5. firm_update - 固件更新工具

```bash
./samples/firm_update/firm_update eth0 <slave_number> <firmware_file>
```

**功能：**
- 通过 FoE（File over EtherCAT）更新从站固件

### 6. eoe_test - EoE 测试（仅 Linux）

```bash
./samples/eoe_test/eoe_test eth0
```

**功能：**
- 测试 Ethernet over EtherCAT 功能

### 7. eni_test - ENI 配置测试（需要 Python）

```bash
./samples/eni_test/eni_test eth0 <eni_file.xml>
```

**功能：**
- 使用 ENI（EtherCAT Network Information）文件配置网络

## 常见问题

### Q1: CMake 版本过低

**错误信息：**
```
CMake 3.28 or higher is required
```

**解决方法：**
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install cmake

# 或从源码编译最新版本
# 参考：https://cmake.org/download/
```

### Q2: 找不到 pthread 库

**错误信息：**
```
undefined reference to 'pthread_*'
```

**解决方法：**
- Linux 系统通常已包含，确保安装了 `build-essential`
```bash
sudo apt install build-essential
```

### Q3: 权限不足（Linux）

**问题：** 需要 root 权限访问网络接口

**解决方法：**
```bash
# 使用 sudo 运行
sudo ./samples/ec_sample/ec_sample eth0

# 或设置 CAP_NET_RAW 能力（推荐）
sudo setcap cap_net_raw+ep ./samples/ec_sample/ec_sample
```

### Q4: 找不到网络接口

**错误信息：**
```
No such device
```

**解决方法：**
```bash
# 列出可用的网络接口
ip link show
# 或
ifconfig

# 使用正确的接口名称
./samples/ec_sample/ec_sample <interface_name>
```

### Q5: 编译错误：找不到头文件

**解决方法：**
```bash
# 确保在构建目录中运行 cmake
cd build
cmake ..

# 检查包含路径是否正确
cmake --build . --verbose
```

### Q6: Windows 编译问题

**问题：** 缺少 WinPcap

**解决方法：**
1. 下载并安装 WinPcap 或 Npcap
2. 确保 CMake 能找到 WinPcap 库
3. 或使用 WSL2 在 Linux 环境下编译

## RTOS 支持

SOEM 支持多种实时操作系统（RTOS），通过平台抽象层（OSAL/OSHW）实现跨平台支持。

### 支持的 RTOS 平台

#### 1. RT-Kernel（RT-Kernel）

**位置：** `osal/rtk/`, `oshw/rtk/`

**配置：**
```cmake
# 使用 RT-Kernel 配置
# 需要设置 CMAKE_SYSTEM_NAME 为相应的 RT-Kernel 平台
# 参考 cmake/rt-kernel.cmake
```

**特点：**
- 专为实时应用设计
- 低延迟、确定性响应
- 需要 RT-Kernel BSP 支持

#### 2. RTEMS

**位置：** `contrib/osal/rtems/`, `contrib/oshw/rtems/`

**配置：**
```cmake
# 使用 RTEMS 工具链
# 参考 contrib/cmake/rtems.cmake
set(RTEMS_TOOLS_PATH /path/to/rtems/tools)
set(RTEMS_BSP your_bsp_name)
```

**特点：**
- 开源实时操作系统
- 支持多种处理器架构
- 适合嵌入式系统

#### 3. VxWorks

**位置：** `contrib/osal/vxworks/`, `contrib/oshw/vxworks/`

**特点：**
- 商业实时操作系统
- 高可靠性
- 广泛用于工业应用

#### 4. ERIKA Enterprise

**位置：** `contrib/osal/erika/`, `contrib/oshw/erika/`

**特点：**
- 开源 RTOS
- 符合 OSEK/VDX 标准
- 适合汽车电子应用

**参考：** http://www.erika-enterprise.com/wiki/index.php?title=EtherCAT_Master

#### 5. INtime

**位置：** `contrib/osal/intime/`, `contrib/oshw/intime/`

**特点：**
- Windows 实时扩展
- 双系统架构（Windows + RTOS）
- 适合需要 Windows 和实时性的应用

### RTOS 平台选择

选择 RTOS 平台时，CMake 会根据 `CMAKE_SYSTEM_NAME` 自动选择相应的实现：

```cmake
# Linux 平台（默认）
set(CMAKE_SYSTEM_NAME Linux)
# 使用 osal/linux/ 和 oshw/linux/

# RT-Kernel 平台
set(CMAKE_SYSTEM_NAME RT-Kernel)
# 使用 osal/rtk/ 和 oshw/rtk/
```

## Linux PREEMPT_RT 使用指南

### 什么是 PREEMPT_RT

PREEMPT_RT（Preemptible Real-Time）是 Linux 内核的实时补丁，将 Linux 内核转换为实时操作系统，提供：

- **可抢占内核**：降低中断延迟
- **优先级继承互斥锁**：避免优先级反转
- **高精度定时器**：微秒级定时精度
- **中断线程化**：减少中断处理延迟

### 安装 PREEMPT_RT 内核

#### Ubuntu/Debian

```bash
# 方法 1：使用预编译包（如果可用）
sudo apt update
sudo apt install linux-image-rt-amd64 linux-headers-rt-amd64

# 方法 2：从源码编译（推荐，可自定义配置）
# 1. 下载对应版本的 Linux 内核源码
# 2. 下载对应版本的 PREEMPT_RT 补丁
# 3. 应用补丁并编译
# 参考：https://wiki.linuxfoundation.org/realtime/documentation/howto/applications/preemptrt_setup
```

#### 验证安装

```bash
# 检查当前内核版本
uname -r

# 检查是否支持 PREEMPT_RT
cat /sys/kernel/realtime
# 输出 1 表示支持实时内核

# 检查内核配置
zcat /proc/config.gz | grep PREEMPT_RT
# 或
cat /boot/config-$(uname -r) | grep PREEMPT_RT
```

### PREEMPT_RT 使用注意事项

#### 1. 实时线程优先级设置

**SOEM 自动设置：**
```c
// osal_thread_create_rt() 会自动设置实时优先级
// 默认优先级：40（SCHED_FIFO）
OSAL_THREAD_FUNC_RT ecatthread(void *ptr)
{
    // 线程已自动设置为实时优先级
    // 无需手动设置
}
```

**手动调整优先级：**
```c
#include <pthread.h>
#include <sched.h>

void set_realtime_priority(int priority)
{
    struct sched_param param;
    param.sched_priority = priority;
    
    // SCHED_FIFO: 先进先出实时调度
    pthread_setschedparam(pthread_self(), SCHED_FIFO, &param);
}

// 使用示例
OSAL_THREAD_FUNC_RT ecatthread(void *ptr)
{
    // 设置更高的优先级（1-99，数字越小优先级越高）
    set_realtime_priority(80);
    
    // 周期任务代码
}
```

**优先级范围：**
- **SCHED_FIFO/SCHED_RR**：1-99（数字越大优先级越高）
- **SCHED_OTHER**：0（普通调度）
- **推荐值**：80-90（高优先级实时任务）

#### 2. 内存锁定

**问题：** 实时任务的内存可能被交换到磁盘，导致不可预测的延迟

**解决：**
```c
#include <sys/mman.h>

int main(void)
{
    // 锁定所有当前和未来的内存页面
    if (mlockall(MCL_CURRENT | MCL_FUTURE) != 0) {
        perror("mlockall failed");
        return 1;
    }
    
    // 初始化 SOEM
    ecx_init(&ctx, "eth0");
    
    // ... 应用代码 ...
    
    return 0;
}
```

**注意事项：**
- 需要 root 权限或 `CAP_IPC_LOCK` 能力
- 锁定内存会减少系统可用内存
- 确保有足够的物理内存

#### 3. CPU 隔离和亲和性

**隔离 CPU 核心：**
```bash
# 在 /etc/default/grub 中添加内核参数
GRUB_CMDLINE_LINUX="isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3"

# 更新 grub 配置
sudo update-grub
sudo reboot
```

**设置 CPU 亲和性：**
```c
#include <pthread.h>
#define _GNU_SOURCE
#include <sched.h>

void set_cpu_affinity(int cpu_id)
{
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu_id, &cpuset);
    
    pthread_setaffinity_np(pthread_self(), sizeof(cpu_set_t), &cpuset);
}

// 在实时线程中调用
OSAL_THREAD_FUNC_RT ecatthread(void *ptr)
{
    set_cpu_affinity(2);  // 绑定到 CPU 2
    // ... 周期任务 ...
}
```

#### 4. 使用单调时间

**SOEM 已自动使用：**
```c
// osal_get_monotonic_time() 使用 CLOCK_MONOTONIC
// 不受系统时钟调整（NTP）影响
void osal_get_monotonic_time(ec_timet *ts)
{
    clock_gettime(CLOCK_MONOTONIC, ts);
}
```

**关键点：**
- ✅ 使用 `CLOCK_MONOTONIC`（单调递增）
- ❌ 避免使用 `CLOCK_REALTIME`（受 NTP 影响）
- ✅ SOEM 已正确实现，无需修改

#### 5. 高精度睡眠

**SOEM 实现：**
```c
// osal_monotonic_sleep() 使用 clock_nanosleep
// 支持绝对时间和高精度
int osal_monotonic_sleep(ec_timet *ts)
{
    return clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, ts, NULL);
}
```

**优势：**
- 使用绝对时间，避免累积误差
- 高精度（纳秒级）
- PREEMPT_RT 下延迟更低

#### 6. 中断延迟优化

**禁用中断合并：**
```bash
# 降低网络接口中断延迟
sudo ethtool -C eth0 rx-usecs 0
sudo ethtool -C eth0 tx-usecs 0
sudo ethtool -C eth0 rx-frames 1
sudo ethtool -C eth0 tx-frames 1
```

**绑定中断到特定 CPU：**
```bash
# 查找网络接口的中断号
grep eth0 /proc/interrupts

# 绑定中断到隔离的 CPU（例如 CPU 2）
echo 4 | sudo tee /proc/irq/24/smp_affinity
# 24 是中断号，4 = CPU 2 (2^2 = 4)
```

#### 7. 系统配置优化

**CPU 频率管理：**
```bash
# 设置为性能模式（禁用频率调节）
sudo cpupower frequency-set -g performance

# 或使用 cpufreq
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

**禁用电源管理：**
```bash
# 禁用 CPU 休眠
sudo systemctl mask sleep.target suspend.target hibernate.target

# 禁用 USB 自动挂起
echo 'SUBSYSTEM=="usb", ATTR{power/autosuspend}="-1"' | \
     sudo tee /etc/udev/rules.d/50-usb_power_save.rules
```

**禁用看门狗：**
```bash
# 禁用软件看门狗（如果不需要）
sudo systemctl stop watchdog
sudo systemctl disable watchdog
```

### PREEMPT_RT 性能验证

#### 1. 检查实时性

```bash
# 安装实时测试工具
sudo apt install rt-tests

# 运行 cyclictest 测试中断延迟
sudo cyclictest -t 4 -p 80 -n -i 10000 -l 10000

# 输出示例：
# T: 0 ( 1234) P:80 I:10000 C:  10000 Min:      5 Act:    8 Avg:   10 Max:   45
# Min: 最小延迟（微秒）
# Max: 最大延迟（微秒）
# 目标：Max < 50μs（高精度应用）
```

#### 2. 测量周期抖动

```c
// 在应用中测量周期时间
void measure_cycle_jitter(void)
{
    ec_timet cycle_start, cycle_end;
    int64 min_cycle = INT64_MAX, max_cycle = 0;
    int64 sum_cycle = 0;
    
    for (int i = 0; i < 1000; i++) {
        osal_get_monotonic_time(&cycle_start);
        
        // 执行周期任务
        ecx_send_processdata(&ctx);
        wkc = ecx_receive_processdata(&ctx, EC_TIMEOUTRET);
        
        osal_get_monotonic_time(&cycle_end);
        
        int64 cycle_time = (cycle_end.tv_sec - cycle_start.tv_sec) * 1000000000LL +
                          (cycle_end.tv_nsec - cycle_start.tv_nsec);
        
        if (cycle_time < min_cycle) min_cycle = cycle_time;
        if (cycle_time > max_cycle) max_cycle = cycle_time;
        sum_cycle += cycle_time;
    }
    
    printf("Cycle time: min=%lld ns, max=%lld ns, avg=%lld ns, jitter=%lld ns\n",
           min_cycle, max_cycle, sum_cycle / 1000, max_cycle - min_cycle);
}
```

### 常见问题

#### Q1: PREEMPT_RT 内核编译失败

**解决方法：**
- 确保使用匹配的内核版本和补丁版本
- 检查内核配置选项
- 参考 PREEMPT_RT 官方文档

#### Q2: 实时线程优先级设置失败

**错误：** `Operation not permitted`

**解决：**
```bash
# 需要 root 权限或设置能力
sudo setcap cap_sys_nice+ep your_app

# 或使用 sudo 运行
sudo ./your_app
```

#### Q3: 内存锁定失败

**错误：** `Cannot allocate memory`

**解决：**
```bash
# 检查可用内存
free -h

# 增加内存限制
ulimit -l unlimited

# 或使用 root 权限
sudo ./your_app
```

#### Q4: 周期抖动仍然很大

**可能原因：**
1. CPU 未隔离
2. 其他进程占用 CPU
3. 中断未绑定

**检查：**
```bash
# 检查 CPU 使用情况
top -H -p $(pidof your_app)

# 检查中断分布
cat /proc/interrupts

# 检查 CPU 隔离
cat /proc/cmdline | grep isolcpus
```

### 最佳实践

1. **使用 PREEMPT_RT 内核**：高精度应用必须使用
2. **隔离 CPU 核心**：为实时任务预留专用 CPU
3. **锁定内存**：避免交换导致的延迟
4. **设置高优先级**：确保实时任务优先执行
5. **绑定中断**：将网络中断绑定到隔离的 CPU
6. **禁用电源管理**：避免 CPU 频率变化
7. **定期测试**：使用 cyclictest 验证实时性

### 性能目标

**一般应用（标准 Linux）：**
- 周期抖动：< 1ms
- 中断延迟：< 100μs

**高精度应用（PREEMPT_RT）：**
- 周期抖动：< 10μs
- 中断延迟：< 50μs
- 同步精度：< 1μs

## 进阶配置

### 交叉编译

```bash
# 设置工具链
cmake -DCMAKE_TOOLCHAIN_FILE=/path/to/toolchain.cmake ..

# 示例工具链文件内容：
# set(CMAKE_SYSTEM_NAME Linux)
# set(CMAKE_C_COMPILER arm-linux-gnueabihf-gcc)
```

### 使用 Ninja 构建系统

```bash
# 配置时指定 Ninja
cmake -G Ninja ..

# 构建
ninja
```

### 并行编译

```bash
# 使用所有 CPU 核心
cmake --build . -j$(nproc)  # Linux
cmake --build . -j%NUMBER_OF_PROCESSORS%  # Windows
```

### 清理构建

```bash
# 清理编译产物
cmake --build . --target clean

# 或删除整个构建目录
rm -rf build
```

### 生成编译数据库

```bash
# 生成 compile_commands.json（用于 IDE 支持）
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ..
```

### 详细构建输出

```bash
# 显示详细的编译命令
cmake --build . --verbose
```

## 验证安装

### 检查库文件

```bash
# Linux
ls -lh build/libsoem.a
# 或
ls -lh build/libsoem.so

# 查看库的符号
nm build/libsoem.a | head
```

### 检查头文件

```bash
# 检查生成的头文件
ls build/include/soem/
cat build/include/soem/ec_options.h
```

### 运行测试

```bash
# 运行示例程序（需要实际的 EtherCAT 硬件）
./samples/ec_sample/ec_sample eth0
```

## 集成到项目

### 使用 CMake

```cmake
# 在你的 CMakeLists.txt 中
find_package(soem REQUIRED)
target_link_libraries(your_target soem)
```

### 使用 pkg-config

```bash
# 如果安装了 pkg-config 文件
pkg-config --cflags --libs soem
```

### 直接链接

```cmake
# 直接链接库文件
target_link_libraries(your_target
    ${CMAKE_CURRENT_SOURCE_DIR}/path/to/libsoem.a
)
target_include_directories(your_target PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/path/to/include
)
```

## 下一步

构建成功后，你可以：

1. **阅读架构文档**：查看 `doc/framework.md` 了解代码架构
2. **运行示例程序**：学习如何使用 SOEM API
3. **阅读 API 文档**：查看头文件中的注释
4. **开发自己的应用**：基于示例程序开发

## 参考资源

- **官方文档**：https://docs.rt-labs.com/soem
- **EtherCAT 规范**：https://www.ethercat.org/
- **项目仓库**：查看项目根目录的 README.md

---

**提示：** 如果在构建过程中遇到问题，请检查：
1. CMake 版本是否满足要求
2. 所有依赖是否已安装
3. 网络接口是否可用（运行示例时）
4. 是否有足够的权限访问网络接口

