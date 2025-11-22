# eniconv.py 脚本使用指南

## 目录
- [概述](#概述)
- [功能说明](#功能说明)
- [使用方法](#使用方法)
- [ENI 文件格式](#eni-文件格式)
- [生成的代码结构](#生成的代码结构)
- [在应用中使用](#在应用中使用)
- [状态转换时机](#状态转换时机)
- [典型应用场景](#典型应用场景)
- [CMake 集成](#cmake-集成)
- [常见问题](#常见问题)
- [最佳实践](#最佳实践)

## 概述

`eniconv.py` 是 SOEM 提供的一个 Python 工具脚本，用于将 **ENI (EtherCAT Network Information)** XML 配置文件转换为可在 SOEM 应用中直接使用的 C 代码。

### 主要功能

- **格式转换**：将 XML 格式的 ENI 文件转换为 C 结构体代码
- **自动化配置**：生成的状态转换命令可在从站状态转换时自动执行
- **工具链集成**：与 TwinCAT、EtherCAT Studio 等配置工具无缝对接

### 工作原理

```
ENI XML 文件 (配置工具生成)
    ↓
eniconv.py (Python 脚本)
    ↓
C 代码文件 (ec_enit 结构体)
    ↓
编译到 SOEM 应用
    ↓
运行时自动执行配置命令
```

## 功能说明

### 输入：ENI XML 文件

ENI（EtherCAT Network Information）是描述 EtherCAT 网络配置的标准 XML 格式文件，通常由以下工具生成：

- **TwinCAT** (Beckhoff)
- **EtherCAT Studio** (Acontis)
- **其他 EtherCAT 配置工具**

**ENI 文件包含：**
- 从站信息（VendorId、ProductCode、RevisionNo）
- CoE（CANopen over EtherCAT）初始化命令
- 状态转换时机配置
- PDO 映射配置

### 处理过程

脚本解析 XML 文件，提取以下信息：

1. **从站配置**
   - 从站编号
   - 制造商 ID (VendorId)
   - 产品代码 (ProductCode)
   - 修订版本 (RevisionNo)

2. **CoE 初始化命令**
   - 命令类型（读/写）
   - SDO 索引和子索引
   - 命令数据
   - 执行时机（状态转换）
   - 超时设置

3. **数据缓冲区**
   - 自动生成数据缓冲区
   - 优化内存布局

### 输出：C 代码文件

生成包含以下内容的 C 代码：

- `ec_enit` 结构体定义
- `ec_enislavet` 从站配置数组
- `ec_enicoecmdt` 命令数组
- 静态数据缓冲区

## 使用方法

### 基本用法

```bash
# 转换 ENI 文件为 C 代码
python3 scripts/eniconv.py sample-eni.xml > eni_config.c

# 或直接输出到文件
python3 scripts/eniconv.py sample-eni.xml eni_config.c
```

### 命令行参数

```bash
python3 scripts/eniconv.py [选项] <eni文件> [输出文件]
```

**参数说明：**
- `eni文件`：输入的 ENI XML 文件（必需）
- `输出文件`：输出的 C 代码文件（可选，默认输出到标准输出）

### 示例

```bash
# 从示例文件生成配置
python3 scripts/eniconv.py samples/eni_test/sample-eni.xml eni_config.c

# 查看生成的代码
cat eni_config.c
```

## ENI 文件格式

### 基本结构

```xml
<?xml version="1.0"?>
<EtherCATConfig xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
                xsi:noNamespaceSchemaLocation="EtherCATConfig.xsd" 
                Version="1.3">
   <Config>
      <Master>
         <Info>
            <Name>SOEM Master</Name>
            <Destination>ffffffffffff</Destination>
            <Source>010101010101</Source>
         </Info>
      </Master>
      <Slave>
         <Info>
            <Name>Device A</Name>
            <VendorId>1</VendorId>
            <ProductCode>0x1234</ProductCode>
            <RevisionNo>0</RevisionNo>
            <SerialNo>0</SerialNo>
         </Info>
         <Mailbox>
            <Send>
               <Start>0x1000</Start>
               <Length>128</Length>
            </Send>
            <Receive>
               <Start>0x1100</Start>
               <Length>128</Length>
               <PollTime>5</PollTime>
            </Receive>
            <Protocol>CoE</Protocol>
            <CoE>
               <InitCmds>
                  <InitCmd>
                     <Comment>Clear SM3 PDO count</Comment>
                     <Transition>PS</Transition>
                     <Timeout>3</Timeout>
                     <Ccs>2</Ccs>
                     <Index>0x1c13</Index>
                     <SubIndex>0</SubIndex>
                     <Data>00</Data>
                  </InitCmd>
                  <!-- 更多 InitCmd -->
               </InitCmds>
            </CoE>
         </Mailbox>
      </Slave>
      <!-- 更多 Slave -->
   </Config>
</EtherCATConfig>
```

### InitCmd 元素说明

| 元素 | 说明 | 示例 |
|------|------|------|
| `Comment` | 命令注释（可选） | `Clear SM3 PDO count` |
| `Transition` | 状态转换时机 | `PS` (PRE_OP→SAFE_OP) |
| `Timeout` | 超时时间（秒） | `3` |
| `Ccs` | 命令类型 | `1`=读, `2`=写 |
| `Index` | SDO 索引 | `0x1c13` |
| `SubIndex` | SDO 子索引 | `0` |
| `Data` | 命令数据（十六进制） | `00` 或 `021a` |
| `CompleteAccess` | 完整访问标志 | `0`=否, `1`=是 |

### Transition 值说明

| 值 | 含义 | 状态转换 |
|----|------|----------|
| `IP` | INIT → PRE_OP | 初始化到预操作 |
| `PS` | PRE_OP → SAFE_OP | 预操作到安全操作 |
| `PI` | PRE_OP → INIT | 预操作到初始化 |
| `SP` | SAFE_OP → PRE_OP | 安全操作到预操作 |
| `SO` | SAFE_OP → OPERATIONAL | 安全操作到操作 |
| `SI` | SAFE_OP → INIT | 安全操作到初始化 |
| `OS` | OPERATIONAL → SAFE_OP | 操作到安全操作 |
| `OP` | OPERATIONAL → PRE_OP | 操作到预操作 |
| `OI` | OPERATIONAL → INIT | 操作到初始化 |

## 生成的代码结构

### 示例输出

```c
#include "soem/soem.h"

// 从站 1 的 CoE 数据缓冲区
static uint8 s1_coeData[6] = {
   0,
   2, 26,
   5, 26,
   2
};

// 从站 1 的 CoE 初始化命令数组
static ec_enicoecmdt s1_coeCmds[4] = {
   {
      // Clear SM3 PDO count
      .Transition = ECT_ESMTRANS_PS,
      .CA = FALSE,
      .Ccs = 2,
      .Index = 0x1C13,
      .SubIdx = 0,
      .Timeout = 3000,
      .DataSize = 1,
      .Data = (s1_coeData + 0)
   },
   {
      // SM3 RxPDO #1: 0x1A02
      .Transition = ECT_ESMTRANS_PS,
      .CA = FALSE,
      .Ccs = 2,
      .Index = 0x1C13,
      .SubIdx = 1,
      .Timeout = 3000,
      .DataSize = 2,
      .Data = (s1_coeData + 1)
   },
   {
      // SM3 RxPDO #2: 0x1A05
      .Transition = ECT_ESMTRANS_PS,
      .CA = FALSE,
      .Ccs = 2,
      .Index = 0x1C13,
      .SubIdx = 2,
      .Timeout = 3000,
      .DataSize = 2,
      .Data = (s1_coeData + 3)
   },
   {
      // Set SM3 PDO count
      .Transition = ECT_ESMTRANS_PS,
      .CA = FALSE,
      .Ccs = 2,
      .Index = 0x1C13,
      .SubIdx = 0,
      .Timeout = 3000,
      .DataSize = 1,
      .Data = (s1_coeData + 5)
   }
};

// 从站配置数组
static ec_enislavet eniSlave[2] = {
   {
      .Slave = 1,
      .VendorId = 1,
      .ProductCode = 0x1234,
      .RevisionNo = 0,
      .CoECmds = s1_coeCmds,
      .CoECmdCount = 4
   },
   {
      .Slave = 2,
      .VendorId = 1,
      .ProductCode = 0x10055aa,
      .RevisionNo = 0,
      .CoECmds = s2_coeCmds,
      .CoECmdCount = 1
   }
};

// ENI 主结构体
ec_enit ec_eni = {
   .slave = eniSlave,
   .slavecount = 2
};
```

### 代码结构说明

1. **数据缓冲区** (`s1_coeData`)
   - 静态分配的字节数组
   - 存储所有命令的数据
   - 通过偏移量引用

2. **命令数组** (`s1_coeCmds`)
   - `ec_enicoecmdt` 结构体数组
   - 包含所有 CoE 初始化命令
   - 按 ENI 文件中的顺序排列

3. **从站配置数组** (`eniSlave`)
   - `ec_enislavet` 结构体数组
   - 每个从站一个元素
   - 包含从站信息和命令指针

4. **ENI 主结构** (`ec_eni`)
   - `ec_enit` 结构体
   - 包含所有从站配置
   - 在应用中引用此结构

## 在应用中使用

### 基本集成步骤

#### 1. 生成配置文件

```bash
python3 scripts/eniconv.py my_config.eni eni_config.c
```

#### 2. 在代码中声明

```c
#include "soem/soem.h"

// 声明外部生成的 ENI 结构
extern ec_enit ec_eni;
```

#### 3. 关联到上下文

```c
static ecx_contextt ctx = {
    .ENI = &ec_eni,  // 关联 ENI 配置
};

int main(void)
{
    // 初始化 SOEM
    if (ecx_init(&ctx, "eth0")) {
        printf("SOEM initialized\n");
        
        // 配置从站（会自动执行 ENI 中的命令）
        if (ecx_config_init(&ctx) > 0) {
            printf("%d slaves configured\n", ctx.slavecount);
            
            // 在状态转换时，SOEM 会自动执行匹配的 ENI 命令
            // 例如：PRE_OP -> SAFE_OP 时执行 ECT_ESMTRANS_PS 命令
        }
        
        ecx_close(&ctx);
    }
    
    return 0;
}
```

### 完整示例

```c
#include <stdio.h>
#include "soem/soem.h"

// 生成的 ENI 配置
extern ec_enit ec_eni;

static uint8 IOmap[4096];
static ecx_contextt ctx = {
    .ENI = &ec_eni,
};

int main(int argc, char *argv[])
{
    if (argc < 2) {
        printf("Usage: %s <interface>\n", argv[0]);
        return 1;
    }
    
    // 初始化
    if (!ecx_init(&ctx, argv[1])) {
        printf("Failed to initialize SOEM\n");
        return 1;
    }
    
    // 配置从站（ENI 命令会自动执行）
    if (ecx_config_init(&ctx) > 0) {
        printf("Found %d slaves\n", ctx.slavecount);
        
        // 映射过程数据
        ecx_config_map_group(&ctx, IOmap, 0);
        
        // 配置分布式时钟
        ecx_configdc(&ctx);
        
        // 进入操作状态
        for (int i = 1; i <= ctx.slavecount; i++) {
            ctx.slavelist[i].state = EC_STATE_OPERATIONAL;
            ecx_writestate(&ctx, i);
            ecx_statecheck(&ctx, i, EC_STATE_OPERATIONAL, EC_TIMEOUTSTATE);
        }
        
        // 周期性数据交换
        for (int cycle = 0; cycle < 1000; cycle++) {
            ecx_send_processdata(&ctx);
            int wkc = ecx_receive_processdata(&ctx, EC_TIMEOUTRET);
            if (wkc > 0) {
                // 处理数据
            }
            osal_usleep(1000);
        }
    }
    
    ecx_close(&ctx);
    return 0;
}
```

## 状态转换时机

### 自动执行机制

当从站状态转换时，SOEM 会自动检查 ENI 配置，并执行匹配的命令：

```c
// 在 ecx_config_init() 中，PRE_OP -> SAFE_OP 转换时
if (context->ENI) {
    // 执行 ECT_ESMTRANS_PS 命令
    ecx_mbxENIinitcmds(context, slave, ECT_ESMTRANS_PS);
}
```

### 状态转换常量

| 常量 | 值 | 状态转换 | 说明 |
|------|-----|----------|------|
| `ECT_ESMTRANS_IP` | 0x0001 | INIT → PRE_OP | 初始化完成 |
| `ECT_ESMTRANS_PS` | 0x0002 | PRE_OP → SAFE_OP | **常用**：配置 PDO |
| `ECT_ESMTRANS_PI` | 0x0004 | PRE_OP → INIT | 返回初始化 |
| `ECT_ESMTRANS_SP` | 0x0008 | SAFE_OP → PRE_OP | 返回预操作 |
| `ECT_ESMTRANS_SO` | 0x0010 | SAFE_OP → OPERATIONAL | 进入操作 |
| `ECT_ESMTRANS_SI` | 0x0020 | SAFE_OP → INIT | 错误恢复 |
| `ECT_ESMTRANS_OS` | 0x0040 | OPERATIONAL → SAFE_OP | 退出操作 |
| `ECT_ESMTRANS_OP` | 0x0080 | OPERATIONAL → PRE_OP | 返回预操作 |
| `ECT_ESMTRANS_OI` | 0x0100 | OPERATIONAL → INIT | 完全重置 |

### 组合使用

可以在一个命令中指定多个转换时机：

```xml
<InitCmd>
   <Transition>PS</Transition>
   <Transition>SO</Transition>
   <!-- 命令会在 PS 和 SO 转换时都执行 -->
</InitCmd>
```

生成的代码：
```c
.Transition = (ECT_ESMTRANS_PS | ECT_ESMTRANS_SO)
```

## 典型应用场景

### 1. PDO 映射配置

**场景**：配置从站的 RxPDO 和 TxPDO 映射

**ENI 配置：**
```xml
<InitCmd>
   <Comment>Clear RxPDO mapping</Comment>
   <Transition>PS</Transition>
   <Ccs>2</Ccs>
   <Index>0x1c12</Index>
   <SubIndex>0</SubIndex>
   <Data>00</Data>
</InitCmd>
<InitCmd>
   <Comment>Add RxPDO 0x1600</Comment>
   <Transition>PS</Transition>
   <Ccs>2</Ccs>
   <Index>0x1c12</Index>
   <SubIndex>1</SubIndex>
   <Data>0016</Data>
</InitCmd>
<InitCmd>
   <Comment>Set RxPDO count</Comment>
   <Transition>PS</Transition>
   <Ccs>2</Ccs>
   <Index>0x1c12</Index>
   <SubIndex>0</SubIndex>
   <Data>01</Data>
</InitCmd>
```

**优势**：
- 无需手动编写 SDO 配置代码
- 配置集中管理
- 易于修改和维护

### 2. 从站参数初始化

**场景**：设置从站的工作模式、滤波器等参数

**ENI 配置：**
```xml
<InitCmd>
   <Comment>Set operation mode</Comment>
   <Transition>PS</Transition>
   <Ccs>2</Ccs>
   <Index>0x6060</Index>
   <SubIndex>0</SubIndex>
   <Data>08</Data>  <!-- 位置模式 -->
</InitCmd>
<InitCmd>
   <Comment>Set filter parameter</Comment>
   <Transition>PS</Transition>
   <Ccs>2</Ccs>
   <Index>0x2030</Index>
   <SubIndex>1</SubIndex>
   <Data>0001</Data>
</InitCmd>
```

### 3. 批量配置多个从站

**场景**：一次配置多个相同或不同的从站

**方法**：
1. 在 TwinCAT 等工具中配置所有从站
2. 导出 ENI 文件
3. 使用 `eniconv.py` 转换为 C 代码
4. 编译到应用中

**优势**：
- 统一管理所有从站配置
- 减少代码重复
- 配置与代码分离

## CMake 集成

### 使用 AddENI.cmake

SOEM 提供了 CMake 函数来简化 ENI 文件的集成：

```cmake
# 包含 ENI 支持
include(cmake/AddENI.cmake)

# 添加 ENI 文件到目标
add_eni(your_target your_config.eni)
```

### 完整示例

```cmake
cmake_minimum_required(VERSION 3.28)
project(my_ecat_app)

# 查找 SOEM
find_package(soem REQUIRED)

# 创建可执行文件
add_executable(my_app
    main.c
)

# 链接 SOEM
target_link_libraries(my_app soem)

# 添加 ENI 配置
include(cmake/AddENI.cmake)
add_eni(my_app config/my_network.eni)
```

### 工作原理

`AddENI.cmake` 会：
1. 查找 `eniconv.py` 脚本
2. 在构建时自动运行脚本
3. 生成 `.c` 文件
4. 将生成的文件添加到目标源文件列表

**构建输出：**
```
[构建时]
my_network.eni → eniconv.py → my_network.c

[编译时]
my_network.c + main.c → my_app
```

## 常见问题

### Q1: 生成的代码编译错误

**问题**：找不到 `ec_enit` 类型定义

**解决**：
```c
// 确保包含正确的头文件
#include "soem/soem.h"
```

### Q2: ENI 命令未执行

**可能原因：**
1. 未关联 ENI 到上下文
2. 从站匹配失败（VendorId/ProductCode 不匹配）
3. 状态转换时机不对

**检查方法：**
```c
// 1. 确认 ENI 已关联
if (ctx.ENI == NULL) {
    printf("ENI not set!\n");
}

// 2. 检查从站匹配
printf("Slave %d: VendorId=0x%08X, ProductCode=0x%08X\n",
       i, ctx.slavelist[i].eep_man, ctx.slavelist[i].eep_id);

// 3. 启用调试输出
// 编译时定义 EC_DEBUG
```

### Q3: 命令执行失败

**检查方法：**
```c
// 在 ecx_mbxENIinitcmds 中检查返回值
int result = ecx_mbxENIinitcmds(&ctx, slave, ECT_ESMTRANS_PS);
if (result == 0) {
    printf("ENI command failed for slave %d\n", slave);
}
```

### Q4: 数据格式错误

**问题**：ENI 文件中的 Data 字段格式不正确

**正确格式：**
```xml
<!-- 十六进制字符串，无空格 -->
<Data>0016</Data>      <!-- 正确 -->
<Data>00 16</Data>     <!-- 错误：有空格 -->
<Data>0x0016</Data>    <!-- 错误：有 0x 前缀 -->
```

### Q5: 从站匹配问题

**问题**：ENI 中的从站信息与实际从站不匹配

**解决**：
1. 检查 VendorId、ProductCode、RevisionNo
2. 使用 `slaveinfo` 工具查看实际从站信息
3. 更新 ENI 文件中的从站信息

## 最佳实践

### 1. 版本控制

**推荐做法：**
- 将 ENI XML 文件纳入版本控制
- 生成的 C 文件添加到 `.gitignore`
- 在构建时自动生成

```gitignore
# 生成的 ENI 配置
*.eni.c
eni_config.c
```

### 2. 配置管理

**建议：**
- 为不同环境创建不同的 ENI 文件
  - `config_production.eni`
  - `config_test.eni`
  - `config_development.eni`
- 使用 CMake 变量选择配置

```cmake
set(ENI_FILE "config_${ENV}.eni" CACHE STRING "ENI file to use")
add_eni(my_app ${ENI_FILE})
```

### 3. 错误处理

**在应用中添加错误检查：**

```c
// 检查 ENI 配置是否加载
if (ctx.ENI == NULL) {
    fprintf(stderr, "Error: ENI configuration not loaded\n");
    return 1;
}

// 检查从站配置数量
if (ctx.ENI->slavecount == 0) {
    fprintf(stderr, "Warning: No slaves in ENI configuration\n");
}
```

### 4. 调试技巧

**启用调试输出：**
```bash
# 编译时启用调试
cmake -DEC_DEBUG=ON ..
make

# 运行时查看 ENI 命令执行
# 在 ecx_mbxENIinitcmds 中添加日志
```

**手动测试命令：**
```c
// 手动执行 ENI 命令进行测试
ecx_mbxENIinitcmds(&ctx, slave, ECT_ESMTRANS_PS);
```

### 5. 性能考虑

**优化建议：**
- 只在必要的状态转换时执行命令
- 避免在周期性任务中执行 ENI 命令
- 使用 `CompleteAccess` 减少命令数量

### 6. 文档化

**在 ENI 文件中添加注释：**
```xml
<InitCmd>
   <Comment>Configure PDO mapping for servo drive</Comment>
   <!-- 详细说明命令的作用和参数 -->
   <Transition>PS</Transition>
   ...
</InitCmd>
```

## 总结

`eniconv.py` 是 SOEM 中连接配置工具和应用代码的重要桥梁。通过使用 ENI 文件，可以：

- ✅ 简化从站配置流程
- ✅ 实现配置与代码分离
- ✅ 提高可维护性
- ✅ 支持工具链集成
- ✅ 自动化状态转换配置

**工作流程：**
```
配置工具 (TwinCAT) 
  → ENI XML 文件 
  → eniconv.py 
  → C 代码 
  → SOEM 应用 
  → 自动执行配置
```

通过合理使用 `eniconv.py`，可以大大提高 EtherCAT 应用的开发效率和可维护性。

