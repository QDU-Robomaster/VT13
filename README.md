# VT13

VT13 图传链路遥控解析模块：从 UART 接收 21 字节协议帧，转换为 `CMD::Data` 喂给 CMD，
并把挡位开关、自定义按键、拨轮和键鼠变化作为事件发出。

## 行为

- 构造时把 UART 设为 921600 bit/s、无校验、8 数据位、1 停止位，并创建线程
  `uart_vt13`（栈深与优先级见 `Param`）。
- 线程每次读取 64 字节（读超时 64 ms），用字节流状态机对齐帧头 `0xA9 0x53` 并拼出
  21 字节帧，随后休眠 2 ms。帧校验：前 19 字节的 CRC16（LibXR `CRC16`，即
  CRC-16/MCRF4XX：多项式 0x1021 反射、初值 0xFFFF）与帧尾 2 字节小端值比较；摇杆与拨轮
  通道须在 364-1684 内、挡位不超过 S。通过校验的帧经
  `cmd.FeedRC(CMD::RCInputSource::RC_INPUT_VT13, ...)` 喂给 CMD。
- 超过 100 ms 没有有效帧时判定离线：向 CMD 喂入一次控制量全零、
  `chassis_online` / `gimbal_online` 为 `false` 的数据，并清除各切换状态。恢复后的第一帧
  只用来建立边沿基线，不触发事件。
- 控制源默认为遥控器模式；Shift+Ctrl+Q 切到遥控器模式，Shift+Ctrl+E 切到键鼠模式。
  - 遥控器模式：底盘 `x` = 左摇杆 Y，`y` = 左摇杆 X，`z` = 右摇杆 X；云台 `yaw` = 右摇杆 X，
    `pit` = 右摇杆 Y；均归一化到 [-1, 1]；扳机按下时开火。
  - 键鼠模式：W/S、A/D 使底盘 `y`、`x` 为 ±1，`z` = 0，按住 Shift 时底盘模式为 `BOOST`；
    云台 `pit` = 鼠标 Y × 1000/32768，`yaw` = 鼠标 X × 1000/32768；鼠标左键按下时开火。

## 事件

`GetEvent()` 返回 VT13 的 `LibXR::Event`，EventBinder 等模块用它绑定下列事件 ID：

- 挡位开关变化：`VT13_SW_POS_C` / `N` / `S`（0-2）。
- 自定义左键、右键、暂停键、扳机的按下 / 松开：`VT13_KEY_PRESSED_*` /
  `VT13_KEY_RELEASE_*`（`0x100`-`0x107`）。
- 切换结果（`0x130`-`0x135`）：自定义左键每次按下翻转一次，发出
  `VT13_KEY_CUSTOM_L_TOGGLE_ON/OFF`；自定义右键同理发出 `VT13_KEY_CUSTOM_R_TOGGLE_ON/OFF`；
  暂停键翻转后发出 `VT13_KEY_PAUSE_TOGGLE_ON/OFF`，但右键切换为 ON 时暂停键总是发出 OFF。
- 拨轮（阈值为中值 ±180）：上拨后 500 ms 内回中发出 `VT13_DIAL_UP_SHORT`，上拨保持
  500 ms 发出 `VT13_DIAL_UP_LONG`，下拨发出 `VT13_DIAL_DOWN_TOUCH`（`0x120`-`0x122`）。
- 键盘按下（上升沿）：`Key::KEY_W` ... `KEY_B`；同时按住 Shift / Ctrl / Shift+Ctrl 时，
  事件 ID 分别加上 1 / 2 / 3 倍 `KEY_NUM`，可用 `ShiftWith()`、`CtrlWith()`、
  `ShiftCtrlWith()` 计算。
- 鼠标（仅键鼠模式）：`KEY_L_PRESS`、`KEY_R_PRESS`、`KEY_M_PRESS` 及对应的
  `*_RELEASE`。

## 依赖

- `QDU-Robomaster/CMD`：接收解析后的控制数据（`CMD::FeedRC`）。

## 构造接口

```cpp
VT13(LibXR::UART& uart,
     CMD& cmd,
     const Param& param = {.task_stack_depth_uart = 1536,
                           .thread_priority_uart = LibXR::Thread::Priority::HIGH});
```

依赖：

- `uart`：连接 VT13 链路的 `LibXR::UART`（模块会重新设置波特率与校验）。
- `cmd`：CMD 模块实例，填写 CMD 的实例 id。

配置（`Param` 字段）：

- `task_stack_depth_uart`：接收线程栈深，默认 1536。
- `thread_priority_uart`：接收线程优先级，默认 `LibXR::Thread::Priority::HIGH`。

## 使用

```sh
xrobot module add QDU-Robomaster/VT13
xrobot setup
xrobot instance add QDU-Robomaster/VT13
```

`xrobot instance add` 在 `User/xrobot.yaml` 中写入一个实例，依赖项留空，默认值按源码写出；
把 `uart` 填为 BSP 中用 `XR_REGISTER` 注册的 UART 对象名，把 `cmd` 填为前面列出的
`QDU-Robomaster/CMD` 实例的 id：

```yaml
modules:
  - module: QDU-Robomaster/CMD
    id: cmd
    args:
      - mode: CMD::Mode::CMD_OP_CTRL
      - chassis_cmd_topic_name: '"chassis_cmd"'
      - gimbal_cmd_topic_name: '"gimbal_cmd"'
      - launcher_cmd_topic_name: '"launcher_cmd"'
  - module: QDU-Robomaster/VT13
    id: vt13_0
    args:
      - uart: uart_vt13
      - cmd: cmd
      - param:
          task_stack_depth_uart: '1536'
          thread_priority_uart: LibXR::Thread::Priority::HIGH
```

BSP 侧：

```cpp
XR_REGISTER(uart_vt13, LibXR::UART);
```

`cmd` 是 `QDU-Robomaster/CMD` 的实例 id，该实例必须在 `modules:` 中排在 VT13 前面。
DR16 与 VT13 可以同时使用同一个 CMD 实例，CMD 会在两路遥控之间选择活动源。

填好后再次运行 `xrobot setup`，生成 `User/xrobot_main.hpp`。

`xrobot module show .`（在本仓库中）或 `xrobot module show Modules/QDU-Robomaster/VT13`
（在 BSP 中）打印 manifest 和当前的构造函数。
