---
title: ENVO0323 App_BringUp_Init 复原学习指引
created: 2026-10-04
status: 复原中
tags:
  - ENVO0323
  - APP
  - 复原学习
---

# App_BringUp_Init 复原学习指引

对应 [[ENVO0323-01-APP-Outline#第 3 步：整理初始化顺序|APP 第 3 步]]。当前主要任务仅为 App_BringUp_Init 的声明和实现。

## 范围与当前状态

- app_main.h/.c 已检查通过。本任务不再重复复原总入口。
- 已移除 app_bringup.h 中的 Init 声明和 app_bringup.c 中的 Init 完整实现，留下定位注释。
- 保留头文件枚举、extern 声明、源文件变量、EEPROM 测试辅助函数、Run 和全部回调。
- app_main.c 仍调用此入口，因此复原完成前工程暂时不能完整编译通过。不用修改调用方来绕过练习。
- 原文件备份：C:\Users\Hina\AppData\Local\Temp\ENVO0323-BringUpInit-BeforeRewrite-20261004。
- 本阶段只做源码学习；未运行、烧录或执行 EEPROM 写测试。

源码：[头文件](<C:/Users/Hina/Desktop/ENVO INTERN/Software/ENVO0323/Inc/app_bringup.h>)、[实现文件](<C:/Users/Hina/Desktop/ENVO INTERN/Software/ENVO0323/Src/app_bringup.c>)。

## 教学方式

每项先解释主要功能和设计理由，再由 Hina 实现；保存后发 `1`，检查“哪里正确、哪里需要改、下一项”。不提供完整函数，不直接填写实现。按下面的功能块推进，不拆成逐条语句练习。不默认熟悉 C2000。

## 函数角色和依赖

调用链：main 完成 MX 外设配置 → App_Init → App_BringUp_Init → VOFA 初始化 → MotorControl 初始化并启动采样定时器。

它负责应用开始前的准备。ADC 配置来自 adc.c，I2C/SPI 句柄来自对应外设模块；EEPROM 与 MT6816 的通信实现仍归 BSP。该函数不执行周期性控制，不启动 TIM1，也不使能功率输出。

初始化顺序：DWT 计时准备 → ADC 校准与注入转换中断准备 → EEPROM 绑定、连接检查和自检 → LED 计时准备 → MT6816 绑定。

## 1. 公共入口与 DWT 计时准备（当前第一项）

任务：在 .h 恢复公共入口声明，在 .c 创建匹配的函数定义，并完成 DWT 周期计数准备。其余功能暂时留空，后续追加。

理解：DWT 是处理器提供的调试与跟踪单元，CYCCNT 用于计数处理器时钟周期。ADC 回调用开始与结束计数的差值观察执行耗时；它不负责触发 ADC，也不是毫秒计时。

准备顺序：通过 CoreDebug 的 DEMCR 打开跟踪功能访问 → 将 DWT 的 CYCCNT 清零 → 在 DWT 的 CTRL 中使能周期计数。相关位标识为 CoreDebug_DEMCR_TRCENA_Msk、DWT_CTRL_CYCCNTENA_Msk。

约束：打开所需位时保留寄存器其他位；公共入口无参数、无返回值。DWT 寄存器定义由当前工程 CMSIS 头文件提供，不另写寄存器地址。

完成标准：声明与定义一致；能解释“开启访问、清零、开启计数”分别做什么，以及为什么在 ADC 回调工作前准备。

## 2. ADC 校准和注入转换中断准备

任务：使用现有 hadc1 完成单端校准，成功后才使能注入转换及完成中断。

可查接口：HAL_ADCEx_Calibration_Start、HAL_ADCEx_InjectedStart_IT。采用互斥分支：校准失败记录 g_adc_abc_error=1；校准成功但启动失败记录 2；两者都成功才设置 g_adc_sync_started=1。

理解：校准用于降低 ADC 内部转换偏差，不等于三相电流零点标定。Start_IT 使 ADC 准备接受外部触发，并不启动 TIM1。

原版行为：ADC 失败后仍继续后面的 EEPROM 和编码器准备，没有提前退出整个 Init。本次按原版复原，不自行改变失败策略。

完成标准：校准失败时不会调用启动接口；成功标志不会在失败分支置位；能区分配置、校准、等待触发、实际转换。

## 3. EEPROM 绑定、检查和自检调度

任务：先将 hi2c1 绑定给 BSP，调用 BSP_EEPROM_CheckConnection 并记录 g_eeprom_status。连接成功才调用现有单字节测试；单字节测试成功才调用现有多字节测试。

依赖：BSP_EEPROM_Init、BSP_EEPROM_CheckConnection、App_BringUp_TestEepromByte、App_BringUp_TestEepromMulti。结果分别保存到已有对应变量。

理解：本函数只组织测试顺序，不重写读写、自检或恢复算法。未执行的测试保持当前启动初值 NOT_RUN，不能当作通过。旧名称 IsReady 不再使用。

完成标准：连接失败时两个测试都不执行；单字节测试失败时多字节测试不执行；成功路径依次执行，分别记录结果。

## 4. LED 结果提示与计时起点

任务：单字节和多字节测试都成功时，把 led_period_ms 设置为 500 ms；其他情况设置为 100 ms。使用 HAL_GetTick 记录 led_last_tick。

理解：两个变量属于模块私有状态，继续保留在 .c。此处只准备周期和起点，真正翻转 LED 由 Run 完成。不用 HAL_Delay。

完成标准：判断使用“两个结果都成功”；能解释此结果仅表示 EEPROM 自检，不代表 ADC、编码器或电机系统通过。

## 5. MT6816 绑定及整项检查

任务：将 hspi2、SPI2_CSN_GPIO_Port、SPI2_CSN_Pin 交给 BSP_MT6816_Init。

理解：绑定驱动所需句柄和片选信息，不在此处发起读取。实际使用 SPI2，不因原理图网络名 SPI1 而改用 hspi1。

完成标准：五项顺序与原版一致；解释后续 MotorControl 启动采样定时器前为什么要先准备驱动；保存源码后检查，再由用户编译。

## 验收边界

源码检查通过、用户报告编译通过、实机验证分别记录。当前不做实机操作。原版 Init 会调用有写入动作的 EEPROM 自检，学习和编译不会执行这些写入。

Init 整项完成后，再进入 APP 大纲第 4 步，详细学习现有 EEPROM 自检辅助函数。
