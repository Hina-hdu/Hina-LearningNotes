---
title: ENVO0323 01 APP 调度与观测复原大纲
created: 2026-10-02
updated: 2026-10-02
projects:
  - ENVO0323
tags:
  - 编写大纲
  - APP
  - 调度
  - VOFA
status: 待学习
stage: 01-APP
---

# ENVO0323 01 APP 调度与观测复原大纲

[[ENVO0323-Learning-Roadmap|返回总路线]] · [[学习目录|学习目录]] · [[#步骤导航|步骤导航]]

前一阶段：[[EEPROM-Driver-Outline]]；下一阶段：[[ENVO0323-02-ADC-PWM-Outline]]。

## 总目标与前置条件

共 11 步。沿用 MT6816 的手把手复原方式：先说清职责、输入输出与调用时机，再逐个写声明和函数，检查后继续。本篇不提供完整实现，也不要求现在清空源码。

前置：理解 MT6816 异步读取和 EEPROM 阻塞读写，能区分私有状态与中断共享量。利用你已有的 C2000 主循环／ISR 经验，重点学习本工程如何接到 HAL 回调。

## 现有事实与复原目标

现有：`App_Init` 依次调用 BringUp、VOFA、MotorControl 初始化；`App_Run` 依次调用 BringUp、MotorControl、VOFA 的 Run。`App_MotorControl_Run` 当前为空，高速控制由 ADC 回调进入。`app_bringup.c` 同时承接板级自检、采样和 BSP 完成回调，职责并不等于只闪灯。

目标：理解并复原现有连接关系，建立可靠的观测解释。暂不改成 RTOS，也不假设工程已有独立算法层。

## 步骤导航

1. [[#第 1 步：画出程序入口|程序入口]]
2. [[#第 2 步：规划 APP 公共接口|公共接口]]
3. [[#第 3 步：整理初始化顺序|初始化顺序]]
4. [[#第 4 步：接回 EEPROM 自检|EEPROM 自检]]
5. [[#第 5 步：复原主循环任务|主循环任务]]
6. [[#第 6 步：接回 ADC 完成回调|ADC 回调]]
7. [[#第 7 步：接回编码器完成回调|编码器回调]]
8. [[#第 8 步：设计观测统计|观测统计]]
9. [[#第 9 步：准备 VOFA 模块|VOFA 模块]]
10. [[#第 10 步：复原发送流程|发送流程]]
11. [[#第 11 步：检查模块衔接|模块衔接]]

## 第 1 步：画出程序入口

任务：从 `main` 找到外设初始化、`App_Init` 和循环内 `App_Run`，另画 ADC、SPI 中断入口。完成标准：能解释循环频率并不是电流环频率，外设配置也不等于已经开始采样或输出。

## 第 2 步：规划 APP 公共接口

任务：复原 `app_main.h` 的两个声明，再整理 BringUp 与 VOFA 的 Init／Run 接口。职责：总入口负责调用，具体模块拥有自己的工作。完成标准：声明与定义一致，不把模块私有计时量塞入头文件。

## 第 3 步：整理初始化顺序

当前主要复原任务：[[ENVO0323-App-BringUp-Init-Guide|App_BringUp_Init 复原学习指引]]。2026-10-04 已备份并移除此函数的声明与实现，其他代码保留；从入口与 DWT 计时准备开始，由 Hina 按功能块实现。

任务：拆解 `App_BringUp_Init` 的 DWT 开启、ADC 校准与注入中断启动、EEPROM 测试、MT6816 绑定。MotorControl 初始化在其后启动采样定时器。完成标准：说明为何回调接收方和采样条件必须先准备；MT6816 绑定实际是 `hspi2`。

## 第 4 步：接回 EEPROM 自检

任务：理解 `App_BringUp_TestEepromByte` 和 `App_BringUp_TestEepromMulti` 的备份、写入、比对、恢复职责；先阅读上一阶段，不重复重写 BSP。完成标准：能区分首次测试失败与恢复失败，并知道多字节测试何时才运行。

## 第 5 步：复原主循环任务

任务：用 `HAL_GetTick` 的无符号时间差复原 `App_BringUp_Run` 闪灯；两种 EEPROM 测试都成功才使用较慢周期。完成标准：不靠长延时维持周期；LED 只表达这项自检，不能作为整个电机系统通过的标志。

## 第 6 步：接回 ADC 完成回调

任务：沿 `ADC1_2_IRQHandler`、HAL、`HAL_ADCEx_InjectedConvCpltCallback` 追踪三相读取、换算、窗口检查与 `App_MotorControl_OnAdcSample`。每 4 次回调尝试启动一次 MT6816 读取。完成标准：能指出何种错误阻止控制入口继续执行，不在此处加入阻塞串口或 EEPROM 写入。

## 第 7 步：接回编码器完成回调

任务：复原 APP 对 `BSP_MT6816_AngleReadyCallback` 的实现，区分 BSP 的弱默认回调与 APP 接收函数。仅 OK 且无失磁才传入有效标志。完成标准：不能把 NO_MAGNET 时可更新的原始角度误当有效反馈；启动请求成功也不是读取成功。

## 第 8 步：设计观测统计

任务：解释请求数、BUSY 数、完成数、有效数、错误数及 ADC 计数各自何时递增。完成标准：用计数变化定位“未触发、来不及完成、数据无效”；错误数包含不同入口，不能直接当失败帧比例。`volatile` 方便观察共享变化，但不保证一组变量是同一帧。

## 第 9 步：准备 VOFA 模块

任务：复原 `App_Vofa_Init` 与私有发送时刻。当前代码把 CAN_EN 拉低选择 USART 模式，使用 USART1、115200 波特率。完成标准：能区分板级通信选择与串口初始化；此处行为需结合实际硬件连接确认。

## 第 10 步：复原发送流程

任务：在 `App_Vofa_Run` 判断已有 ADC 样本及 100 ms 周期，按 A／B／C 顺序放入三个 float，再接 `00 00 80 7F` 尾部，共 16 字节。当前用阻塞 `HAL_UART_Transmit`，超时配置 10 ms。完成标准：成功与错误各自统计；说明发送周期限制了观测速度。

## 第 11 步：检查模块衔接

任务：逐个列出入口、所属执行环境、输入、输出和错误出口，再做原工程编译检查。完成标准：无重复 HAL 回调定义；BSP 能通知 APP，APP 能通知控制模块，VOFA 无需反向驱动采样。复原每个小函数后先解释再继续，不一次复制整文件。

## 常见边界

VOFA 逐个读取三个共享电流值，当前没有统一快照，可能跨两个 ADC 帧；它适合慢速观察，不能证明高速波形细节。串口阻塞会占用主循环时间，即使中断仍可运行，也不能称为 DMA 后台发送。EEPROM 自检会写入非易失存储，实机前需确认测试区可用，避免反复上电写测试。

## 阶段验收

能口述“初始化→采样触发→ADC 回调→BSP 请求→SPI 完成→APP 接收→低速观测”，并解释任意计数的含义，才进入 ADC/PWM。后续实机默认 PWM 关闭；本篇只做源码规划，未执行复原、编译或硬件验收。

## 依据与相关笔记

- [APP 总入口](<C:/Users/Hina/Desktop/ENVO INTERN/Software/ENVO0323/Src/app_main.c>)、[BringUp](<C:/Users/Hina/Desktop/ENVO INTERN/Software/ENVO0323/Src/app_bringup.c>)、[VOFA](<C:/Users/Hina/Desktop/ENVO INTERN/Software/ENVO0323/Src/app_vofa.c>)。
- [中断入口](<C:/Users/Hina/Desktop/ENVO INTERN/Software/ENVO0323/Src/stm32g4xx_it.c>)与当前本地 HAL ADC／SPI 实现是回调依据。
- [[EEPROM-Driver-Outline]] · [[ENVO0323-02-ADC-PWM-Outline]] · [[ENVO0323-Learning-Roadmap]]
