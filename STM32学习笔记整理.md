# STM32 学习笔记整理

> 根据 8 页手写笔记整理，内容主要针对 **STM32F103 系列 + Keil MDK + STM32 标准外设库（SPL）**。已重新编排章节、补全示例代码，并将容易写错或需要依据芯片型号确认的地方标注出来。
>
> **说明**：代码示例是根据笔记内容整理出的教学示例，并非对原笔记逐字转录。实际工程中的头文件、启动文件、外设引脚需按芯片型号与电路板调整。

## 目录

1. [新建 STM32 工程](#1-新建-stm32-工程)
2. [STM32 电源与引脚基础](#2-stm32-电源与引脚基础)
3. [GPIO 基本概念与初始化](#3-gpio-基本概念与初始化)
4. [GPIO 的八种工作模式](#4-gpio-的八种工作模式)
5. [GPIO 输出与输入函数](#5-gpio-输出与输入函数)
6. [按键电路、上下拉与电平判断](#6-按键电路上下拉与电平判断)
7. [C 语言基础补充](#7-c-语言基础补充)
8. [模块化编程与头文件](#8-模块化编程与头文件)
9. [EXTI 外部中断入门](#9-exti-外部中断入门)
10. [易错点速查](#10-易错点速查)

---

## 1. 新建 STM32 工程

以下对应笔记第 1 页的 Keil 新建工程流程。

1. **Project → New µVision Project**：创建工程并指定工程名和路径。
2. 在器件选择界面选择具体的 STM32 芯片型号（如所用 STM32F103xx）。
3. 在工程目录下建立 `Start`、`Library`、`User` 等文件夹，便于管理。
4. 添加 **CMSIS、标准外设库和设备支持文件**，并将相关 `.c` 文件加入 Keil 工程组。
5. 在 `Start` 组中加入对应芯片型号的启动文件，例如某些 STM32F103 中容量型号使用 `startup_stm32f10x_md.s`。
6. 在 `User` 组中加入 `main.c`、`stm32f10x_conf.h`、`stm32f10x_it.c` 等必要文件（具体以实际工程组织为准）。
7. 配置编译器的 C/C++ Include Paths，使头文件能被正确找到。
8. 在预处理器宏中配置需要的芯片系列和外设库选项，例如 `STM32F10X_MD`、`USE_STDPERIPH_DRIVER`（以目标型号为准）。
9. 选择 **Debug** 调试器为 `ST-Link Debugger`，进入 **Settings** 检查连接与下载配置；在 **Utilities / Flash Download** 检查烧录算法，执行下载或复位运行。

### 1.1 参考目录结构

```text
STM32_Project/
├── Project/                     # Keil 工程文件
├── Start/                       # CMSIS、系统文件、启动汇编文件
├── Library/                     # STM32F10x 标准外设库
│   ├── inc/                     # 各外设头文件
│   └── src/                     # 各外设源文件
└── User/
    ├── main.c
    ├── stm32f10x_conf.h
    └── stm32f10x_it.c
```

### 1.2 常见库文件

- `stm32f10x.h`：主要器件头文件。
- `stm32f10x_gpio.h/.c`：GPIO 外设相关声明与实现。
- `stm32f10x_rcc.h/.c`：时钟控制。
- `stm32f10x_exti.h/.c`：外部中断控制器。
- `stm32f10x_afio.h` 的相关 AFIO 功能在 STM32F1 的 GPIO/AFIO 库接口中提供（标准外设库一般使用 `stm32f10x_gpio.h` 中的 AFIO 函数）。
- 启动汇编文件：初始化向量表与启动过程，**必须与芯片型号、存储容量匹配**。

> 原笔记中的复制路径和文件数量因库版本不同可能变化。应以手上 **STM32F10x Standard Peripheral Library** 的实际目录为准，不要机械照搬路径。

---

## 2. STM32 电源与引脚基础

### 2.1 常见电源引脚

| 引脚/信号 | 作用 | 连接要点 |
|---|---|---|
| `VDD` | 数字电源输入 | STM32F103 通常接 **3.3 V** |
| `VSS` | 数字地 | 接 `GND` |
| `VDDA` | 模拟电源 | 按数据手册连接，常用 3.3 V 并进行滤波 |
| `VSSA` | 模拟地 | 按硬件设计要求连接地 |
| `VBAT` | 备份域电源 | 不使用独立后备电池时，通常接到合适的主电源轨（依数据手册设计） |
| `FT` 标记 | 5 V tolerant（部分数字引脚耐受 5 V） | **不等于芯片用 5 V 供电**；是否支持及限制需查引脚表 |

> 笔记中特别提醒：`VDD` 不是 5 V 电源。STM32F103 的 GPIO 高低电平门限、模拟功能与 5 V 耐受都应按具体引脚的数据手册判断。

### 2.2 GPIO 的分组

- `GPIOA` 对应 `PA0`、`PA1`、`PA2`……
- `GPIOB` 对应 `PB0`、`PB1`、`PB2`……
- `GPIOC` 对应 `PC0`、`PC1`、`PC2`……

例如：**PA0 对应 GPIOA 的 Pin 0，PB2 对应 GPIOB 的 Pin 2**。

---

## 3. GPIO 基本概念与初始化

以 **PA0 配置为推挽输出** 为例。

### 3.1 初始化顺序

1. 开启 GPIOA 的外设时钟。
2. 定义 `GPIO_InitTypeDef` 结构体。
3. 为结构体的 `GPIO_Pin`、`GPIO_Mode`、`GPIO_Speed` 字段赋值。
4. 调用 `GPIO_Init()` 完成引脚初始化。

### 3.2 示例代码

```c
#include "stm32f10x.h"

void LED_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;

    // 1. 开启 GPIOA 时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

    // 2. 选择 PA0
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;

    // 3. 配置为推挽输出
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;

    // 4. 最大输出速度配置为 50 MHz（并非实际翻转频率）
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;

    // 5. 应用配置
    GPIO_Init(GPIOA, &GPIO_InitStructure);
}
```

**函数拆解**

```c
RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
```

- 第一个参数：要控制的外设时钟（这里是 GPIOA）。
- 第二个参数：`ENABLE` 开启，`DISABLE` 关闭。

```c
GPIO_Init(GPIOA, &GPIO_InitStructure);
```

- `GPIOA`：配置哪一组 GPIO。
- `&GPIO_InitStructure`：把初始化结构体的**地址**传入函数。

> `GPIO_Speed_50MHz` 是 F1 输出驱动速度档位，不代表 MCU 会自动输出 50 MHz 方波。

---

## 4. GPIO 的八种工作模式

在 STM32F1 标准外设库中，经常见到以下八种模式。

| 枚举值 | 中文名称 | 典型用途 |
|---|---|---|
| `GPIO_Mode_AIN` | 模拟输入 | ADC 模拟信号采集 |
| `GPIO_Mode_IN_FLOATING` | 浮空输入 | 外部已有确定电平驱动的数字输入 |
| `GPIO_Mode_IPD` | 下拉输入 | 默认低电平的按键、数字信号 |
| `GPIO_Mode_IPU` | 上拉输入 | 默认高电平的按键、数字信号 |
| `GPIO_Mode_Out_OD` | 通用开漏输出 | 需外部上拉的信号线 |
| `GPIO_Mode_Out_PP` | 通用推挽输出 | LED、普通数字输出 |
| `GPIO_Mode_AF_OD` | 复用开漏输出 | 部分外设复用输出（如 I²C 的典型需求） |
| `GPIO_Mode_AF_PP` | 复用推挽输出 | USART TX、定时器 PWM 等复用输出 |

### 4.1 推挽输出 vs. 开漏输出

- **推挽输出**：内部上下管主动驱动高电平或低电平，适合一般数字输出。
- **开漏输出**：通常只能主动拉低；释放后需依赖上拉电阻获得高电平，可用于共享信号线等场景。

### 4.2 输入上拉/下拉

- **上拉输入**：未被外部驱动时趋于高电平。
- **下拉输入**：未被外部驱动时趋于低电平。
- **浮空输入**：无内部上拉/下拉，若无稳定外部驱动，电平可能不确定。

---

## 5. GPIO 输出与输入函数

### 5.1 输出函数

| 函数 | 作用 |
|---|---|
| `GPIO_SetBits(GPIOx, GPIO_Pin)` | 将指定引脚置为高电平 |
| `GPIO_ResetBits(GPIOx, GPIO_Pin)` | 将指定引脚置为低电平 |
| `GPIO_WriteBit(GPIOx, GPIO_Pin, BitVal)` | 将指定引脚写成 `Bit_SET` 或 `Bit_RESET` |
| `GPIO_Write(GPIOx, PortVal)` | 向 GPIO 端口的 16 位输出数据寄存器写入值 |

```c
GPIO_SetBits(GPIOA, GPIO_Pin_0);                 // PA0 置 1
GPIO_ResetBits(GPIOA, GPIO_Pin_0);               // PA0 置 0
GPIO_WriteBit(GPIOA, GPIO_Pin_0, Bit_SET);       // PA0 置 1
GPIO_WriteBit(GPIOA, GPIO_Pin_0, Bit_RESET);     // PA0 置 0
GPIO_Write(GPIOA, 0x0001);                       // 写入整个 GPIOA 输出端口
```

### 5.2 位掩码（Pin Mask）

| 引脚 | 位掩码 | 二进制最低位示意 |
|---|---|---|
| `PA0` | `0x0001` | `...0001` |
| `PA1` | `0x0002` | `...0010` |
| `PA2` | `0x0004` | `...0100` |
| `PA3` | `0x0008` | `...1000` |

组合引脚可使用按位或：

```c
GPIO_Pin_0 | GPIO_Pin_1
```

即选中 PA0 与 PA1 两个位。**按位或 `|` 用于组合位标志，不是算术加法的通用替代品。**

> `GPIO_Write(GPIOA, PortVal)` 是对**整个端口**的输出数据进行写入，而非只改变某一位。修改单个引脚通常优先使用 `SetBits`、`ResetBits` 或 `WriteBit`。

### 5.3 输入与输出状态读取

| 函数 | 作用 |
|---|---|
| `GPIO_ReadInputDataBit(GPIOx, GPIO_Pin)` | 读取一个引脚的**实际输入电平**（IDR） |
| `GPIO_ReadInputData(GPIOx)` | 读取整个端口的输入数据寄存器（IDR） |
| `GPIO_ReadOutputDataBit(GPIOx, GPIO_Pin)` | 读取一个引脚的**输出数据寄存器位**（ODR） |
| `GPIO_ReadOutputData(GPIOx)` | 读取整个端口的输出数据寄存器（ODR） |

```c
uint8_t level = GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_2);
```

**重要区别**：`ReadInputDataBit()` 读取管脚输入状态，`ReadOutputDataBit()` 读取输出寄存器中保存的逻辑值，两者并不总是相同。

---

## 6. 按键电路、上下拉与电平判断

笔记讨论了上拉按键和下拉按键的基本电路。

### 6.1 上拉按键：按下为低电平

```text
3.3V
  |
[上拉电阻]
  |
  +------ GPIO 输入
  |
[按键 K1]
  |
 GND
```

- 松开：输入节点通过上拉电阻接 3.3 V，读取 **1**。
- 按下：输入节点与地相连，读取 **0**。
- 可使用外部上拉，也可在合适的电路中使用 STM32 内部上拉。

### 6.2 下拉按键：按下为高电平

```text
3.3V
  |
[按键 K1]
  |
  +------ GPIO 输入
  |
[下拉电阻]
  |
 GND
```

- 松开：输入节点被下拉，读取 **0**。
- 按下：输入节点接 3.3 V，读取 **1**。

### 6.3 电平判断示例

以 **PB2 接上拉式按键** 为例：

```c
void Key_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_2;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
    GPIO_Init(GPIOB, &GPIO_InitStructure);
}

uint8_t Key_IsPressed(void)
{
    // 上拉按键：低电平表示按下
    return GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_2) == Bit_RESET;
}
```

> 若外部电路的连接方式不同，按键的有效电平也会不同。真实按键还需要考虑**机械抖动（消抖）**。GPIO 不是拿来直接给 LED/负载供大电流的，LED 应串联限流电阻。

---

## 7. C 语言基础补充

### 7.1 常见整数类型与固定宽度类型

笔记整理了常见 C 类型与 `stdint.h` 中类型的关系：

| C 类型 | STM32 常见环境下的位宽 | 对应固定宽度类型 |
|---|---:|---|
| `char` / `signed char` | 8 位 | `int8_t`（有符号时） |
| `unsigned char` | 8 位 | `uint8_t` |
| `short` | 16 位 | `int16_t` |
| `unsigned short` | 16 位 | `uint16_t` |
| `int` | 32 位 | `int32_t` |
| `unsigned int` | 32 位 | `uint32_t` |
| `long long` | 64 位 | `int64_t` |
| `unsigned long long` | 64 位 | `uint64_t` |

> 注意：C 标准**不保证** `char` 一定是有符号的，也不保证 `int` 在所有平台都是 32 位。开发嵌入式程序时，若需要明确位宽，推荐使用 `uint8_t`、`uint16_t`、`uint32_t` 等类型。

### 7.2 `#define`：宏定义

```c
#define ABC 12345

int a = ABC; // 预处理后相当于 int a = 12345;
```

- 主要用于预处理阶段的文本替换。
- 宏定义本身通常**不带分号**。
- 它不是变量，也不天然具有类型信息。

### 7.3 `typedef`：类型别名

```c
#include <stdint.h>

typedef unsigned char uint8_alias_t;
uint8_alias_t a = 10;
```

`typedef` 给已有类型起一个新名字，并不生成一种全新的底层数据类型。

### 7.4 `struct`：结构体

```c
struct Student
{
    char grade;
    int age;
    float score;
};

struct Student s;
s.grade = 'A';

struct Student *p = &s;
p->grade = 'B';
```

- 结构体变量访问字段：`.`，例如 `s.grade`。
- 指向结构体的指针访问字段：`->`，例如 `p->grade`。
- `p->grade` 等价于 `(*p).grade`。

### 7.5 `enum`：枚举

```c
enum Week
{
    MON = 1,
    TUES = 2,
    WEDS           // 自动递增为 3
};

enum Week today = MON;
```

- 枚举成员可以手动赋整数值。
- 后续未显式赋值的枚举成员，默认等于前一个成员的值加一。
- 在 C 语言中，若未使用 `typedef`，变量通常声明为 `enum Week today;`，不能直接写 `Week today;`。

---

## 8. 模块化编程与头文件

笔记提到 `.c` 用于函数实现，`.h` 用于对外声明。

### 8.1 `LED.h`

```c
#ifndef __LED_H
#define __LED_H

void LED_Init(void);

#endif
```

这里的 `#ifndef / #define / #endif` 是**头文件保护**，用于避免同一个头文件被重复包含时造成重复声明或定义问题。

### 8.2 `LED.c`

```c
#include "stm32f10x.h"
#include "LED.h"

void LED_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);
}
```

### 8.3 `main.c`

```c
#include "stm32f10x.h"
#include "LED.h"

int main(void)
{
    LED_Init();

    while (1)
    {
        // 主循环
    }
}
```

> `LED.c` 需要加入 Keil 的工程编译列表，不能仅仅放在磁盘目录中。

---

## 9. EXTI 外部中断入门

### 9.1 中断的基本流程

```text
程序正常运行
      |
检测到满足条件的外部事件
      |
触发 EXTI 中断请求
      |
NVIC 允许且中断条件有效
      |
进入中断服务函数（ISR）
      |
清除对应的中断挂起标志
      |
返回被中断的程序
```

笔记中的中断草图强调：**发生事件 → 暂停当前执行流 → 处理中断 → 返回原程序**。

### 9.2 STM32F1 的 EXTI 线与 GPIO 引脚

- `EXTI0` 对应编号为 **0** 的引脚，例如 `PA0`、`PB0`、`PC0` 等之一。
- `EXTI6` 对应编号为 **6** 的引脚，例如 `PA6`、`PB6`、`PC6` 等之一。
- `EXTI7` 对应编号为 **7** 的引脚。
- 对 STM32F1 的 GPIO 外部中断映射而言，**同一条 EXTI 线一次只能选择一个 GPIO 端口作为来源**：例如无法同时把 `PA0` 和 `PB0` 配置为两个独立的 `EXTI0` 输入。
- 不同编号的引脚，如 `PA6` 和 `PA7`，可分别使用 `EXTI6` 与 `EXTI7`；但某些 EXTI 线共用 NVIC 中断入口（如 `EXTI9_5_IRQn`），需要在服务函数中分别判断挂起位。

### 9.3 常用触发方式

| 触发方式 | 含义 |
|---|---|
| `EXTI_Trigger_Rising` | 上升沿触发（低 → 高） |
| `EXTI_Trigger_Falling` | 下降沿触发（高 → 低） |
| `EXTI_Trigger_Rising_Falling` | 上升沿和下降沿均触发 |

按键示例：上拉按键从松开到按下通常发生**下降沿**，适合 `EXTI_Trigger_Falling`；但实际应用仍需要消抖处理。

### 9.4 STM32F1 外部中断配置思路

1. 打开对应 GPIO 和 **AFIO** 的 APB2 时钟。
2. 将目标 GPIO 配置为合适的输入模式。
3. 通过 `GPIO_EXTILineConfig()` 选择 EXTI 线对应的 GPIO 端口。
4. 配置 `EXTI_InitTypeDef`：选择中断线、模式、触发方式并使能。
5. 配置 NVIC，允许对应中断通道。
6. 编写对应 `EXTIx_IRQHandler()` 函数，并在处理中清除中断挂起标志。

下面是 **PA0 下降沿触发 EXTI0** 的简化示例：

```c
#include "stm32f10x.h"

void EXTI0_Config(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    EXTI_InitTypeDef EXTI_InitStructure;
    NVIC_InitTypeDef NVIC_InitStructure;

    RCC_APB2PeriphClockCmd(
        RCC_APB2Periph_GPIOA | RCC_APB2Periph_AFIO,
        ENABLE
    );

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    GPIO_EXTILineConfig(GPIO_PortSourceGPIOA, GPIO_PinSource0);

    EXTI_InitStructure.EXTI_Line = EXTI_Line0;
    EXTI_InitStructure.EXTI_Mode = EXTI_Mode_Interrupt;
    EXTI_InitStructure.EXTI_Trigger = EXTI_Trigger_Falling;
    EXTI_InitStructure.EXTI_LineCmd = ENABLE;
    EXTI_Init(&EXTI_InitStructure);

    NVIC_InitStructure.NVIC_IRQChannel = EXTI0_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 1;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 1;
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);
}

void EXTI0_IRQHandler(void)
{
    if (EXTI_GetITStatus(EXTI_Line0) != RESET)
    {
        // 在此处理按键事件（避免长时间阻塞）
        EXTI_ClearITPendingBit(EXTI_Line0);
    }
}
```

> 此示例展示配置流程；是否需要时钟初始化、优先级分组、按键消抖及与其他模块联动，应依完整工程补充。

---

## 10. 易错点速查

1. **PA0 和 PB0 不是同一个引脚**：端口 A/B 不同，只是编号都为 0。
2. **GPIO 时钟要先开启**：使用 `GPIOA` 通常先开 `RCC_APB2Periph_GPIOA`。
3. **`GPIO_Init()` 第二个参数是指针**：写 `&GPIO_InitStructure`。
4. **GPIO 输入、输出读取不同**：`ReadInputDataBit()` 读管脚状态；`ReadOutputDataBit()` 读输出寄存器位。
5. **上拉按键按下通常读到 0**，下拉按键按下通常读到 1；以实际接线为准。
6. **`GPIO_Write()` 修改整个端口**，不是只改某一个引脚。
7. **GPIO 速度 50 MHz 不等于自动输出 50 MHz 方波**。
8. **`VDD` 常用 3.3 V，不要因为 `FT` 就给 MCU 电源脚接 5 V**。
9. **EXTI 同编号的 GPIO 无法独立同时映射到同一 EXTI 线**。
10. **中断处理函数应及时清除挂起位**，并尽量避免耗时操作。
11. **头文件写声明、源文件写实现**，模块 `.c` 文件需要加入工程。
12. **代码中的函数名大小写和下划线必须准确**：例如 `GPIO_InitTypeDef`、`GPIO_InitStructure`、`GPIO_ReadInputDataBit`。

---

## 原始笔记页面对应

| 图片 | 主要内容 |
|---|---|
| `IMG_3931.jpeg` | 新建 Keil 工程、复制库文件、配置项目与 ST-Link |
| `IMG_3932.jpeg` | STM32 电源引脚、GPIO 时钟与初始化 |
| `IMG_3934.jpeg` | GPIO 八种模式、PA0/PB2、LED 引脚 |
| `IMG_3935.jpeg` | GPIO 操作与输入输出读取 API、位掩码 |
| `IMG_3936.jpeg` | 上拉/下拉电阻、按键输入与电平判断 |
| `IMG_3937.jpeg` | C 类型、宏、typedef、struct |
| `IMG_3938.jpeg` | enum、头文件保护与 .c/.h 模块化 |
| `IMG_3940.jpeg` | 中断流程与 EXTI 线映射 |

**整理原则**：以原笔记的知识点为主；将不清晰的手写函数名统一为 STM32F1 标准外设库常见正式命名，并针对可能产生误解的地方增加解释。
