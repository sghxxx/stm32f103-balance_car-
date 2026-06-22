# button.c 回调驱动框架总结

## 一、框架概览

`button.c` / `button.h` 实现了一套**事件驱动的按钮管理框架**，核心思想是"机制与策略分离"：驱动层负责消抖、状态检测、长按/连击识别等底层机制；应用层通过注册回调函数注入业务行为。

## 二、架构设计

### 数据结构

```c
typedef struct {
    // 配置参数
    GPIO_TypeDef *GPIOx;
    uint16_t GPIO_Pin;

    // 四个回调函数指针（核心）
    void (*button_pressed_cb)(void);              // 按下回调
    void (*button_released_cb)(void);             // 松开回调
    void (*button_clicked_cb)(uint8_t clicks);    // 单击/连击回调
    void (*button_long_pressed_cb)(uint8_t ticks); // 长按回调

    // 状态机变量（内部私有）
    uint8_t  LastState, ChangePending;
    uint32_t PendingTime, LastPressedTime, LastReleasedTime;
    uint8_t  LongPressTicks, ClickCnt;
} Button_TypeDef;
```

### 分层架构

```
┌─────────────────────────────────────┐
│  app_button.c（应用层）               │
│  注册回调：OnUserKey_Clicked()        │
│  只关心"按钮被点击了要做什么"          │
├─────────────────────────────────────┤
│  button.c（驱动层）                   │
│  消抖 → 状态检测 → 事件分类 → 调用回调  │
│  不关心"回调里到底做了什么"            │
├─────────────────────────────────────┤
│  GPIO 硬件抽象                        │
└─────────────────────────────────────┘
```

## 三、回调机制原理

### 1. 函数指针作为配置项

`Button_TypeDef` 结构体中直接内嵌函数指针成员。初始化时默认为 `NULL`（空），应用层调用 Setter 注入：

```c
My_Button_SetClickCb(&userKey, OnUserKey_Clicked);
```

### 2. 事件分发

驱动层在状态机流转到特定状态时，检查回调是否非空并调用：

```c
static void OnButtonPressed(Button_TypeDef *Button) {
    if (Button->button_pressed_cb != 0)
        Button->button_pressed_cb();  // 不知道也不关心被调者做了什么
}
```

### 3. 支持的事件类型

| 事件 | 回调签名 | 触发条件 |
|------|----------|----------|
| 按下 | `void (*)(void)` | 消抖后确认电平变低 |
| 松开 | `void (*)(void)` | 消抖后确认电平变高 |
| 单击/连击 | `void (*)(uint8_t clicks)` | 松开后 `ClickInterval` 内无新按下，`clicks` 为累计点击次数 |
| 长按 | `void (*)(uint8_t ticks)` | 按下超过阈值后首次触发，之后按 `TickInterval` 周期性触发，`ticks` 递增 |

## 四、状态机流程

```
          ┌──────────┐
          │  空闲等待  │ ← 轮询 GPIO
          └────┬─────┘
               │ 电平变化
               ▼
          ┌──────────┐
          │ 消抖等待  │ (PendingTime + 10ms)
          └────┬─────┘
               │ 确认变化
          ┌────▼────┐
     ┌───→│ 已按下   │───→ 回调: pressed_cb()
     │    └────┬────┘
     │         │ 长按超时?
     │         ▼
     │    ┌──────────┐
     │    │ 长按触发  │───→ 回调: long_pressed_cb(ticks++)  循环
     │    └────┬────┘
     │         │ 松开
     │         ▼
     │    ┌──────────┐
     └────│ 已松开   │───→ 回调: released_cb()
          └────┬────┘
               │ ClickInterval 超时?
               ▼
          ┌──────────┐
          │ 连击触发  │───→ 回调: clicked_cb(ClickCnt)
          └──────────┘
```

## 五、在企业项目中的应用价值

### 1. 解耦（Decoupling）— 最核心价值

驱动层与业务层完全解耦。button.c 可以原封不动地复用到任何 STM32 项目中，无论这个按钮是用来控制电机、切换 LED 还是发送网络请求。**写一次，到处用。**

### 2. 依赖倒置（Dependency Inversion）

高层模块（app_button.c）不依赖低层模块（button.c）的具体实现，两者都依赖抽象（回调函数签名）。这符合 SOLID 原则中的 DIP。

### 3. 单一职责（Single Responsibility）

- `button.c`：只负责"何时触发什么事件"
- `app_button.c`：只负责"事件发生后做什么"

每个文件职责单一，修改业务逻辑不影响驱动稳定性。

### 4. 开闭原则（Open/Closed）

添加新的按钮行为只需：
- 写一个新的回调函数
- 调用 Setter 注册

**无需修改 button.c 一行代码**。对扩展开放，对修改封闭。

### 5. 面向对象模拟（OOP in C）

用 C 语言模拟了面向对象的核心特性：

| OOP 概念 | C 实现 |
|----------|--------|
| 封装 | `Button_TypeDef` 结构体聚合数据 + 函数指针 |
| 多态 | 不同按钮注册不同回调，运行时行为不同 |
| 接口 | 回调函数签名即为"接口契约" |

### 6. 可测试性

可以注入 mock 回调来验证驱动层行为，也可以绕过驱动层单独测试业务回调。这在需要做单元测试的企业项目中至关重要。

### 7. 多实例支持

每个 `Button_TypeDef` 实例独立维护自己的状态和回调。项目中可以同时使用多个按钮，互不干扰：

```c
Button_TypeDef btn1, btn2, btn3;
My_Button_SetClickCb(&btn1, OnBtn1_Click);
My_Button_SetClickCb(&btn2, OnBtn2_Click);
My_Button_SetLongPressCb(&btn3, OnBtn3_LongPress);
```

## 六、与主流框架的类比

| 框架 | 相似机制 |
|------|----------|
| **Linux 内核 input 子系统** | 驱动报告事件，应用层注册 handler |
| **Qt 信号槽** | `connect(button, SIGNAL(clicked()), obj, SLOT(onClick()))` |
| **JavaScript 事件监听** | `button.addEventListener('click', callback)` |
| **Zephyr RTOS GPIO callback** | `gpio_add_callback()` 注册中断回调 |

## 七、延伸思考

这个模式不仅限于按钮，可以推广到任何"事件源 + 策略注入"的场景：

- **串口接收** → 回调处理不同协议帧
- **ADC 采样完成** → 回调进行数据处理
- **定时器超时** → 回调执行周期任务
- **编码器转动** → 回调更新位置/速度

核心公式：**驱动层提供事件 + 应用层注入策略 = 高复用、低耦合的嵌入式架构**

---

*基于项目 `sghxxx/stm32f103-balance_car-` 中 button.c 的实现分析*
