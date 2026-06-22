# STM32F103 平衡车

> 基于 STM32F103C8T6 的两轮自平衡小车项目，采用 MPU6050 姿态传感器 + PID 控制算法。

## 硬件平台

- **主控**: STM32F103C8T6 (Cortex-M3, 72MHz)
- **姿态传感器**: MPU6050 (6轴陀螺仪+加速度计)
- **电机驱动**: PWM + TB6612 / L298N
- **编码器**: 霍尔编码器电机 (速度闭环)
- **显示**: 0.96寸 OLED (I2C)
- **通信**: USART2 蓝牙/串口遥控
- **电源**: 2S/3S 锂电池

## 软件架构

```
user/
├── main.c              # 主程序入口
├── app_control.c       # 平衡控制 (直立环 + 速度环 PID)
├── app_mpu6050.c       # MPU6050 姿态解算 (加速度/陀螺仪数据获取)
├── app_motor.c         # 电机控制 (PWM输出 + 死区)
├── app_encoder.c       # 编码器测速 (M法/T法)
├── app_pwm.c           # PWM 驱动
├── app_bat.c           # ADC 电池电压检测
├── app_button.c        # 按键处理 (模式切换)
├── app_rc.c            # 蓝牙/UART 遥控
├── app_usart2.c        # USART2 串口通信
└── stm32f10x_it.c      # 中断服务函数

my_lib/                 # 硬件驱动库
├── pid.c/h             # PID 控制器
├── oled.c/h            # OLED 显示驱动
├── qmath.c/h           # 数学工具 (卡尔曼/互补滤波)
├── i2c.c/h / si2c.c/h  # I2C (软件/硬件)
├── spi.c/h             # SPI 驱动
├── usart.c/h           # USART 驱动
├── button.c/h          # 按键驱动
└── delay.c/h           # 延时函数
```

## 控制原理

```
MPU6050 角度 → 互补滤波 → PID 直立环 (PD)
                            +
编码器速度 → PID 速度环 (PI)  → PWM → 电机

                   蓝牙遥控
```

## 编译工具

- **IDE**: Keil MDK-ARM V5
- **编译链**: ARMCC V5/V6
- **库**: STM32F10x Standard Peripheral Library V3.5

## 快速开始

1. 用 Keil MDK 打开 `balance_car/balance_car/balance_car.uvprojx`
2. 编译下载到 STM32F103C8T6
3. 上电后小车自动进入平衡模式

## 笔记

📖 详细开发笔记请参阅 [平衡车.md](平衡车.md)

---

*项目地址: [sghxxx/stm32f103-balance_car-](https://github.com/sghxxx/stm32f103-balance_car-)*
