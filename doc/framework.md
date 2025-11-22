# SOEM 代码架构与框架文档

## 目录
- [概述](#概述)
- [整体架构](#整体架构)
- [核心模块](#核心模块)
- [设计模式](#设计模式)
- [关键数据结构](#关键数据结构)
- [数据流与通信机制](#数据流与通信机制)
- [平台抽象层](#平台抽象层)
- [EtherCAT 协议实现](#ethercat-协议实现)
- [状态机管理](#状态机管理)
- [错误处理机制](#错误处理机制)
- [性能优化要点](#性能优化要点)
- [扩展点与钩子](#扩展点与钩子)
- [知识点总结](#知识点总结)

## 概述

SOEM (Simple Open EtherCAT Master) 是一个轻量级的 EtherCAT 主站实现库。其设计遵循以下原则：

- **模块化设计**：清晰的模块划分，便于维护和扩展
- **平台抽象**：通过 OSAL 和 OSHW 实现跨平台支持
- **上下文驱动**：使用上下文结构体管理状态，支持多实例
- **实时性优先**：针对嵌入式实时系统优化
- **轻量级**：最小化资源占用

## 整体架构

### 架构层次图

```
┌─────────────────────────────────────────────────────────┐
│                   应用程序层                              │
│         (samples: ec_sample, slaveinfo, etc.)            │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│                  SOEM API 层                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │ ec_main  │ │ ec_config│ │ ec_coe   │ │ ec_dc    │ │
│  │ ec_base  │ │ ec_foe   │ │ ec_soe   │ │ ec_eoe   │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│              平台抽象层 (OSAL/OSHW)                      │
│  ┌──────────────┐         ┌──────────────┐             │
│  │ OSAL         │         │ OSHW          │             │
│  │ - 线程       │         │ - 网络驱动    │             │
│  │ - 互斥锁     │         │ - 适配器查找  │             │
│  │ - 时间       │         │ - 原始套接字   │             │
│  └──────────────┘         └──────────────┘             │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────────────────────────────────────┐
│                   操作系统层                              │
│         Linux / Windows / RT-Kernel                      │
└─────────────────────────────────────────────────────────┘
```

### 核心组件关系

```
ecx_contextt (上下文)
    ├── ecx_portt (端口/网络接口)
    ├── ec_slavet[] (从站列表)
    ├── ec_groupt[] (组列表)
    ├── ec_mbxpoolt (邮箱池)
    └── ec_eringt (错误环)
```

## 核心模块

### 1. ec_main.c - 主模块

**职责：**
- EtherCAT 主站初始化和关闭
- 从站状态管理
- 邮箱通信
- EEPROM 访问
- 过程数据交换

**关键函数：**

```c
// 初始化主站
int ecx_init(ecx_contextt *context, const char *ifname);

// 初始化冗余端口
int ecx_init_redundant(ecx_contextt *context, ecx_redportt *redport, 
                       const char *ifname, char *if2name);

// 关闭主站
void ecx_close(ecx_contextt *context);

// 读取从站状态
int ecx_readstate(ecx_contextt *context);

// 写入从站状态
int ecx_writestate(ecx_contextt *context, uint16 slave);

// 状态检查
uint16 ecx_statecheck(ecx_contextt *context, uint16 slave, 
                      uint16 reqstate, int timeout);

// 发送过程数据
int ecx_send_processdata(ecx_contextt *context);

// 接收过程数据
int ecx_receive_processdata(ecx_contextt *context, int timeout);
```

**设计要点：**
- 使用上下文结构体 (`ecx_contextt`) 管理所有状态
- 支持多实例（通过不同的上下文）
- 邮箱池管理避免频繁内存分配

### 2. ec_base.c - 基础通信模块

**职责：**
- EtherCAT 数据报构建
- 底层读写操作
- 逻辑地址访问

**关键函数：**

```c
// 设置数据报
int ecx_setupdatagram(ecx_portt *port, void *frame, uint8 com, 
                      uint8 idx, uint16 ADP, uint16 ADO, 
                      uint16 length, void *data);

// 添加数据报
uint16 ecx_adddatagram(ecx_portt *port, void *frame, uint8 com, 
                       uint8 idx, boolean more, uint16 ADP, 
                       uint16 ADO, uint16 length, void *data);

// 广播读写
int ecx_BRD(ecx_portt *port, uint16 ADP, uint16 ADO, 
            uint16 length, void *data, int timeout);
int ecx_BWR(ecx_portt *port, uint16 ADP, uint16 ADO, 
            uint16 length, void *data, int timeout);

// 逻辑地址读写
int ecx_LRD(ecx_portt *port, uint32 LogAdr, uint16 length, 
            void *data, int timeout);
int ecx_LWR(ecx_portt *port, uint32 LogAdr, uint16 length, 
            void *data, int timeout);
int ecx_LRW(ecx_portt *port, uint32 LogAdr, uint16 length, 
            void *data, int timeout);
```

**设计要点：**
- 数据报封装遵循 EtherCAT 规范
- 支持多种寻址模式（广播、自动增量、配置地址、逻辑地址）
- 工作计数器（WKC）验证确保数据完整性

### 3. ec_config.c - 配置模块

**职责：**
- 从站自动发现和配置
- PDO 映射
- FMMU 配置
- 同步管理器（SM）配置

**关键函数：**

```c
// 初始化配置
int ecx_config_init(ecx_contextt *context);

// 映射组
int ecx_config_map_group(ecx_contextt *context, void *pIOmap, uint8 group);

// 恢复从站
int ecx_recover_slave(ecx_contextt *context, uint16 slave, int timeout);

// 重新配置从站
int ecx_reconfig_slave(ecx_contextt *context, uint16 slave, int timeout);
```

**配置流程：**

```
1. 检测从站 (ecx_detect_slaves)
   └── 广播读取 TYPE 寄存器，获取从站数量

2. 读取从站信息 (ecx_readstate)
   └── 读取每个从站的 EEPROM (SII)
   └── 解析从站配置信息

3. 配置同步管理器 (SM)
   └── 设置 SM0-SM3 的地址和长度
   └── 配置 SM 类型（邮箱/过程数据）

4. 配置 FMMU
   └── 设置逻辑地址到物理地址的映射
   └── 配置输入/输出方向

5. 映射过程数据
   └── 构建 IOmap 缓冲区
   └── 设置从站的输入/输出指针
```

### 4. ec_coe.c - CANopen over EtherCAT

**职责：**
- SDO（服务数据对象）通信
- PDO（过程数据对象）配置
- 对象字典访问
- 紧急消息处理

**关键函数：**

```c
// SDO 读写
int ecx_SDOread(ecx_contextt *context, uint16 slave, uint16 index, 
                uint8 subindex, boolean CA, int *psize, void *p, int timeout);
int ecx_SDOwrite(ecx_contextt *context, uint16 Slave, uint16 Index, 
                 uint8 SubIndex, boolean CA, int psize, const void *p, int Timeout);

// PDO 读写
int ecx_RxPDO(ecx_contextt *context, uint16 Slave, uint16 RxPDOnumber, 
              int psize, const void *p);
int ecx_TxPDO(ecx_contextt *context, uint16 slave, uint16 TxPDOnumber, 
              int *psize, void *p, int timeout);

// 对象字典
int ecx_readODlist(ecx_contextt *context, uint16 Slave, ec_ODlistt *pODlist);
int ecx_readODdescription(ecx_contextt *context, uint16 Item, 
                          ec_ODlistt *pODlist);
```

**CoE 协议栈：**

```
应用层
  │
CoE 层 (ec_coe.c)
  │
邮箱层 (ec_main.c - ecx_mbxsend/ecx_mbxreceive)
  │
EtherCAT 数据报层 (ec_base.c)
  │
网络层 (OSHW)
```

### 5. ec_dc.c - 分布式时钟

**职责：**
- 分布式时钟同步
- 时钟偏移计算
- 同步激活

**关键函数：**

```c
// 配置分布式时钟
boolean ecx_configdc(ecx_contextt *context);

// 同步激活
void ecx_dcsync0(ecx_contextt *context, uint16 slave, boolean act, 
                 uint32 CyclTime, int32 CyclShift);
void ecx_dcsync01(ecx_contextt *context, uint16 slave, boolean act, 
                  uint32 CyclTime0, uint32 CyclTime1, int32 CyclShift);
```

**DC 同步原理：**

1. **参考时钟选择**：第一个支持 DC 的从站作为参考
2. **传播延迟测量**：测量数据从主站到各从站的传播时间
3. **时钟偏移计算**：计算各从站时钟相对于参考时钟的偏移
4. **同步激活**：设置同步周期和偏移量

### 6. ec_foe.c - File over EtherCAT

**职责：**
- 文件传输
- 固件更新

**关键函数：**

```c
// FoE 读写
int ecx_FOEread(ecx_contextt *context, uint16 slave, char *filename, 
                uint32 password, int *psize, void *p, int timeout);
int ecx_FOEwrite(ecx_contextt *context, uint16 slave, char *filename, 
                 uint32 password, int psize, void *p, int timeout);
```

### 7. ec_soe.c - Servo over EtherCAT

**职责：**
- SoE 协议实现
- 伺服驱动器通信

### 8. ec_eoe.c - Ethernet over EtherCAT

**职责：**
- EoE 协议实现
- IP 地址配置
- 以太网帧隧道

## 设计模式

### 1. 上下文模式 (Context Pattern)

**实现：** `ecx_contextt` 结构体

**目的：** 封装所有状态，支持多实例

```c
struct ecx_context {
    // 公共接口
    ecx_portt port;              // 网络端口
    ec_slavet slavelist[];       // 从站列表
    ec_groupt grouplist[];       // 组列表
    
    // 私有状态
    uint8 esibuf[];              // EEPROM 缓存
    ec_eringt elist;            // 错误列表
    ec_mbxpoolt mbxpool;         // 邮箱池
    
    // 配置钩子
    ec_enit *ENI;                // ENI 配置
    int (*FOEhook)(...);         // FoE 钩子
    int (*EOEhook)(...);         // EoE 钩子
};
```

**优势：**
- 线程安全（每个线程使用独立上下文）
- 支持多网络接口
- 便于测试（可模拟上下文）

### 2. 策略模式 (Strategy Pattern)

**实现：** 平台特定的 OSAL/OSHW 实现

**目的：** 在不同平台上使用不同的实现策略

```
OSAL 接口
  ├── osal/linux/osal.c      (Linux 实现)
  ├── osal/win32/osal.c      (Windows 实现)
  └── osal/rtk/osal.c        (RT-Kernel 实现)

OSHW 接口
  ├── oshw/linux/oshw.c      (Linux 实现)
  ├── oshw/win32/oshw.c      (Windows 实现)
  └── oshw/rtk/oshw.c        (RT-Kernel 实现)
```

### 3. 对象池模式 (Object Pool Pattern)

**实现：** `ec_mbxpoolt` 邮箱池

**目的：** 减少内存分配，提高实时性能

```c
typedef struct {
    int listhead, listtail, listcount;
    int mbxemptylist[EC_MBXPOOLSIZE];
    osal_mutext *mbxmutex;
    ec_mbxbuft mbx[EC_MBXPOOLSIZE];
} ec_mbxpoolt;
```

**使用流程：**
1. 初始化时预分配邮箱缓冲区
2. 需要时从池中获取（`ecx_getmbx`）
3. 使用完毕后归还（`ecx_dropmbx`）

### 4. 状态机模式 (State Machine Pattern)

**实现：** EtherCAT 从站状态机

**状态转换：**

```
INIT → PRE_OP → SAFE_OP → OPERATIONAL
  ↑                              ↓
  └────────── ERROR ←────────────┘
```

**关键函数：**
- `ecx_writestate()` - 状态转换请求
- `ecx_readstate()` - 读取当前状态
- `ecx_statecheck()` - 等待状态转换完成

### 5. 观察者模式 (Observer Pattern)

**实现：** 错误列表和钩子函数

**目的：** 异步错误通知

```c
// 错误推送
void ecx_pusherror(ecx_contextt *context, const ec_errort *Ec);

// 错误弹出
boolean ecx_poperror(ecx_contextt *context, ec_errort *Ec);

// 钩子函数
int (*FOEhook)(uint16 slave, int packetnumber, int datasize);
int (*EOEhook)(ecx_contextt *context, uint16 slave, void *eoembx);
```

## 关键数据结构

### 1. ecx_contextt - 主上下文

```c
struct ecx_context {
    // 网络端口
    ecx_portt port;
    
    // 从站管理
    ec_slavet slavelist[EC_MAXSLAVE];
    int slavecount;
    
    // 组管理
    ec_groupt grouplist[EC_MAXGROUP];
    
    // 错误管理
    boolean ecaterror;
    ec_eringt elist;
    
    // 分布式时钟
    int64 DCtime;
    
    // 内部缓存
    uint8 esibuf[EC_MAXEEPBUF];      // EEPROM 缓存
    uint32 esimap[EC_MAXEEPBITMAP];  // EEPROM 位图
    
    // 邮箱池
    ec_mbxpoolt mbxpool;
    
    // 配置钩子
    ec_enit *ENI;
    int (*FOEhook)(...);
    int (*EOEhook)(...);
    
    // 用户数据
    void *userdata;
};
```

### 2. ec_slavet - 从站信息

```c
typedef struct ec_slave {
    uint16 state;              // 从站状态
    uint16 ALstatuscode;       // AL 状态码
    uint16 configadr;          // 配置地址
    uint16 aliasadr;           // 别名地址
    
    // EEPROM 信息
    uint32 eep_man;            // 制造商 ID
    uint32 eep_id;             // 产品代码
    uint32 eep_rev;            // 修订版本
    uint32 eep_ser;            // 序列号
    
    // 过程数据
    uint16 Obits, Ibits;       // 输出/输入位数
    uint32 Obytes, Ibytes;     // 输出/输入字节数
    uint8 *outputs, *inputs;   // 数据指针
    uint32 Ooffset, Ioffset;   // 偏移量
    
    // 同步管理器
    ec_smt SM[EC_MAXSM];
    uint8 SMtype[EC_MAXSM];
    
    // FMMU
    ec_fmmut FMMU[EC_MAXFMMU];
    
    // 邮箱
    uint16 mbx_l, mbx_rl;      // 邮箱长度
    uint16 mbx_wo, mbx_ro;     // 邮箱偏移
    uint16 mbx_proto;          // 支持的协议
    
    // 分布式时钟
    boolean hasdc;
    int32 DCcycle;             // DC 周期
    int32 DCshift;             // DC 偏移
    
    // 配置函数
    int (*PO2SOconfig)(ecx_contextt *context, uint16 slave);
    
    char name[EC_MAXNAME + 1]; // 可读名称
} ec_slavet;
```

### 3. ec_groupt - 组信息

```c
typedef struct ec_group {
    uint32 logstartaddr;       // 逻辑起始地址
    uint32 Obytes, Ibytes;     // 输出/输入字节数
    uint8 *outputs, *inputs;   // 数据指针
    boolean hasdc;              // 是否有 DC
    uint16 DCnext;             // 下一个 DC 从站
    uint16 nsegments;          // IO 段数量
    uint32 IOsegment[EC_MAXIOSEGMENTS]; // IO 段列表
    uint16 outputsWKC, inputsWKC; // 期望的工作计数器
    boolean docheckstate;       // 是否检查状态
    ec_mbxqueuet mbxtxqueue;    // 邮箱发送队列
} ec_groupt;
```

### 4. ec_errort - 错误信息

```c
typedef struct {
    ec_timet Time;             // 错误时间
    boolean Signal;            // 信号标志
    uint16 Slave;              // 从站号
    uint16 Index;              // 索引
    uint8 SubIdx;              // 子索引
    ec_err_type Etype;         // 错误类型
    union {
        int32 AbortCode;       // 中止代码
        struct {
            uint16 ErrorCode;
            uint8 ErrorReg;
            // ...
        };
    };
} ec_errort;
```

## 数据流与通信机制

### 过程数据交换流程

```
应用层
  │
  ├─ 准备输出数据 → IOmap[outputs]
  │
  ├─ ecx_send_processdata()
  │   ├─ 构建 EtherCAT 帧
  │   ├─ 填充输出数据
  │   └─ 发送到网络
  │
  ├─ 网络传输（EtherCAT 环）
  │   ├─ 从站 1 处理
  │   ├─ 从站 2 处理
  │   └─ ...
  │
  ├─ ecx_receive_processdata()
  │   ├─ 接收返回帧
  │   ├─ 验证工作计数器 (WKC)
  │   └─ 提取输入数据
  │
  └─ 读取输入数据 ← IOmap[inputs]
```

### 邮箱通信流程

```
发送端：
  ecx_mbxsend()
    ├─ 从邮箱池获取缓冲区
    ├─ 填充邮箱数据
    ├─ 发送到从站
    └─ 等待确认

接收端：
  ecx_mbxhandler() (周期性调用)
    ├─ 检查邮箱状态
    ├─ 接收邮箱数据
    ├─ 根据协议类型分发
    │   ├─ CoE → ec_coe.c
    │   ├─ FoE → ec_foe.c
    │   ├─ SoE → ec_soe.c
    │   └─ EoE → ec_eoe.c
    └─ 归还缓冲区到池
```

### EtherCAT 帧结构

```
以太网帧头 (14 字节)
  ├─ 目标 MAC (6 字节)
  ├─ 源 MAC (6 字节)
  └─ 类型 (0x88A4) (2 字节)

EtherCAT 数据报 (可变长度)
  ├─ 长度 (2 字节)
  ├─ 命令 (1 字节)
  ├─ 索引 (1 字节)
  ├─ ADP (2 字节)
  ├─ ADO (2 字节)
  ├─ 长度 (2 字节)
  ├─ 中断 (2 字节)
  └─ 数据 (可变)

工作计数器 (2 字节)
FCS (4 字节)
```

## 平台抽象层

### OSAL (Operating System Abstraction Layer)

**职责：** 抽象操作系统相关功能

**接口分类：**

1. **时间管理**
   - `osal_get_monotonic_time()` - 获取单调时间
   - `osal_current_time()` - 获取当前时间
   - `osal_timer_*()` - 定时器操作

2. **线程管理**
   - `osal_thread_create()` - 创建线程
   - `osal_thread_create_rt()` - 创建实时线程

3. **同步原语**
   - `osal_mutex_*()` - 互斥锁

4. **内存管理**
   - `osal_malloc()` / `osal_free()`

**平台实现：**
- `osal/linux/` - POSIX 实现
- `osal/win32/` - Windows API 实现
- `osal/rtk/` - RT-Kernel 实现

### OSHW (Operating System Hardware)

**职责：** 抽象硬件相关功能

**接口分类：**

1. **网络适配器**
   - `oshw_find_adapters()` - 查找适配器
   - `oshw_free_adapters()` - 释放适配器列表

2. **网络驱动**
   - `nicdrv.c` - 原始套接字操作
   - 发送/接收 EtherCAT 帧

**平台实现：**
- `oshw/linux/` - Linux 原始套接字
- `oshw/win32/` - WinPcap/Npcap
- `oshw/rtk/` - RT-Kernel 网络驱动

## EtherCAT 协议实现

### 寻址模式

1. **广播寻址 (Broadcast)**
   - `EC_CMD_BRD` - 广播读
   - `EC_CMD_BWR` - 广播写
   - 地址：ADP = 0x0000

2. **自动增量寻址 (Auto Increment)**
   - `EC_CMD_APRD` - 自动增量读
   - `EC_CMD_APWR` - 自动增量写
   - 地址：ADP = 从站位置地址

3. **配置地址寻址 (Configured Address)**
   - `EC_CMD_FPRD` - 配置地址读
   - `EC_CMD_FPWR` - 配置地址写
   - 地址：ADP = 配置地址

4. **逻辑寻址 (Logical)**
   - `EC_CMD_LRD` - 逻辑读
   - `EC_CMD_LWR` - 逻辑写
   - `EC_CMD_LRW` - 逻辑读写
   - 地址：32 位逻辑地址（通过 FMMU 映射）

### 工作计数器 (Working Counter)

**作用：** 验证数据报是否被正确处理

**规则：**
- 每个从站处理数据报后，WKC 加 1
- 主站发送时 WKC = 0
- 返回时 WKC = 处理的从站数量
- WKC 不匹配表示通信错误

**实现：**
```c
// 发送后检查 WKC
wkc = ecx_receive_processdata(context, timeout);
if (wkc != expectedWKC) {
    // 错误处理
}
```

## 状态机管理

### EtherCAT 从站状态

```
EC_STATE_NONE (0x00)        - 无状态
EC_STATE_INIT (0x01)        - 初始化
EC_STATE_PRE_OP (0x02)      - 预操作
EC_STATE_BOOT (0x03)        - 启动
EC_STATE_SAFE_OP (0x04)     - 安全操作
EC_STATE_OPERATIONAL (0x08) - 操作
EC_STATE_ERROR (0x10)      - 错误
EC_STATE_ACK (0x10)        - 确认
```

### 状态转换函数

```c
// 请求状态转换
ecx_writestate(context, slave);

// 等待状态转换完成
ecx_statecheck(context, slave, reqstate, timeout);

// 读取当前状态
ecx_readstate(context);
```

### 典型状态转换流程

```
1. INIT
   └─ ecx_init() 后所有从站处于 INIT

2. PRE_OP
   └─ 配置邮箱和 PDO
   └─ ecx_config_init()

3. SAFE_OP
   └─ 验证配置
   └─ 准备过程数据交换

4. OPERATIONAL
   └─ 开始周期性过程数据交换
   └─ ecx_send_processdata() / ecx_receive_processdata()
```

## 错误处理机制

### 错误类型

```c
typedef enum {
    EC_ERR_TYPE_SDO_ERROR,
    EC_ERR_TYPE_EMERGENCY,
    EC_ERR_TYPE_PACKET_ERROR,
    EC_ERR_TYPE_SDOINFO_ERROR,
    EC_ERR_TYPE_FOE_ERROR,
    EC_ERR_TYPE_SOE_ERROR,
    EC_ERR_TYPE_MBX_ERROR,
    // ...
} ec_err_type;
```

### 错误环缓冲区

```c
typedef struct ec_ering {
    int16 head;                    // 写指针
    int16 tail;                    // 读指针
    ec_errort Error[EC_MAXELIST + 1];
} ec_eringt;
```

### 错误处理流程

```
1. 错误发生
   └─ ecx_pusherror() 推入错误环

2. 应用检查
   └─ ecx_iserror() 检查是否有错误
   └─ ecx_poperror() 弹出错误

3. 错误恢复
   └─ ecx_recover_slave() 恢复从站
   └─ ecx_reconfig_slave() 重新配置
```

## 性能优化要点

### 1. 邮箱池

- 预分配邮箱缓冲区，避免运行时分配
- 使用对象池模式减少内存碎片

### 2. EEPROM 缓存

- 缓存从站 EEPROM 数据
- 使用位图跟踪已读取的数据

### 3. 过程数据对齐

- 字节对齐减少访问开销
- 支持打包模式（packedMode）减小帧大小

### 4. 工作计数器优化

- 预期 WKC 验证，快速检测错误
- 批量处理减少检查次数

### 5. 实时线程

- 使用 `osal_thread_create_rt()` 创建实时线程
- 周期性任务使用单调时间睡眠

## 扩展点与钩子

### 1. 配置钩子

```c
// PO2SO 配置函数
int (*PO2SOconfig)(ecx_contextt *context, uint16 slave);
```

**用途：** 从站特定的配置（PRE_OP → SAFE_OP）

### 2. FoE 钩子

```c
int (*FOEhook)(uint16 slave, int packetnumber, int datasize);
```

**用途：** FoE 传输进度回调

### 3. EoE 钩子

```c
int (*EOEhook)(ecx_contextt *context, uint16 slave, void *eoembx);
```

**用途：** EoE 帧处理回调

### 4. ENI 配置

```c
ec_enit *ENI;
```

**用途：** 使用 ENI 文件进行自动配置

## 知识点总结

### EtherCAT 核心概念

1. **主站 (Master)**：控制网络的主设备
2. **从站 (Slave)**：被控制的设备
3. **过程数据 (Process Data)**：周期性交换的 I/O 数据
4. **邮箱 (Mailbox)**：非周期性通信通道
5. **同步管理器 (SM)**：管理邮箱和过程数据缓冲区
6. **FMMU**：现场总线内存管理单元，实现逻辑地址映射
7. **分布式时钟 (DC)**：网络范围内的时钟同步

### SOEM 设计要点

1. **上下文驱动**：所有操作通过上下文进行
2. **平台抽象**：OSAL/OSHW 实现跨平台
3. **模块化**：清晰的功能模块划分
4. **实时性**：针对实时系统优化
5. **轻量级**：最小化资源占用

### 关键 API 使用模式

```c
// 1. 初始化
ecx_contextt ctx;
ecx_init(&ctx, "eth0");

// 2. 配置
ecx_config_init(&ctx);
ecx_config_map_group(&ctx, IOmap, 0);

// 3. 进入操作状态
for (int i = 1; i <= ctx.slavecount; i++) {
    ctx.slavelist[i].state = EC_STATE_OPERATIONAL;
    ecx_writestate(&ctx, i);
    ecx_statecheck(&ctx, i, EC_STATE_OPERATIONAL, timeout);
}

// 4. 周期性数据交换
while (running) {
    ecx_send_processdata(&ctx);
    wkc = ecx_receive_processdata(&ctx, timeout);
    // 处理数据
}

// 5. 关闭
ecx_close(&ctx);
```

### 线程模型建议

```
主线程
  └─ 初始化、配置、状态管理

实时线程
  └─ 周期性过程数据交换
  └─ 严格的时间要求

监控线程
  └─ 错误检查
  └─ 从站状态监控
  └─ 恢复处理
```

### 内存管理

- **静态分配**：从站列表、组列表等使用静态数组
- **对象池**：邮箱缓冲区使用对象池
- **缓存**：EEPROM 数据缓存
- **用户分配**：IOmap 由用户分配

### 时间管理

- **单调时间**：用于时间间隔测量
- **系统时间**：用于日志和 DC 初始化
- **定时器**：用于超时检测
- **实时睡眠**：周期性任务使用 `osal_monotonic_sleep()`

---

**总结：** SOEM 通过清晰的模块划分、平台抽象和上下文管理，实现了一个轻量级、可扩展的 EtherCAT 主站库。其设计充分考虑了实时性和资源受限环境的需求，是学习和实现 EtherCAT 主站的优秀参考。

