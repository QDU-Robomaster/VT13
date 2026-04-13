# VT13

## 1. 模块作用

VT13 图传链路遥控解析模块。将 21 字节协议帧转换为 CMD 输入和事件。
Manifest 描述：VT13 link receiver parsing

## 2. 主要函数说明

1. ThreadVT13: UART 线程读取并分发控制数据。
2. ParseRC: 校验帧头与 CRC，解析通道/按键/拨轮字段并生成 CMD::Data。
3. VerifyCRC / LibXR::CRC16::Calculate: 协议校验实现（CRC16/CCITT-FALSE）。
4. CheckoutOffline / Offline: 链路离线检测与失控保护输出。
5. GetEvent: 对外暴露事件绑定入口。

## 3. 接入步骤

1. 添加模块并确保 vt13 串口与 DMA 配置正确。
2. 将 VT13 输出的控制数据喂给 CMD。
3. 在 EventBinder 中绑定 VT13 拨杆和按键事件。

标准命令流程：

```bash
xrobot_add_mod VT13 --instance-id vt13
xrobot_gen_main
cube-cmake --build /home/Xiaoyu/Documents/bsp-dev-c/build/debug --
```

## 4. 配置示例（YAML）

```yaml
module: VT13
entry_header: Modules/VT13/VT13.hpp
constructor_args:
  - CMD: '@cmd'
  - task_stack_depth_uart: 1536
  - thread_priority_uart: LibXR::Thread::Priority::HIGH
template_args:
[]
```

## 5. 依赖与硬件

Required Hardware:

- vt13
- dma
- uart

Depends:

- qdu-future/CMD

## 6. 代码入口

Modules/VT13/VT13.hpp
