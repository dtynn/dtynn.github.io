# DirectInput 阅读理解：概述

## DirectInput 阅读理解

`DirectInput` 阅读理解会以 [微软 DirectInput 英文文档](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee416842(v=vs.85)) 为基准，参考微软的中文文档及其他资料。

阅读理解的目标是 *编写一个可以监听通过蓝牙连接的DS4手柄的按键事件的程序* 。

在本篇文章中，主要会接触和理解一些基础概念。

## DirectInput 简介
`DirectInput` 是 微软 `DirectX` API 中的一个组件，用于处理来自键盘、鼠标、手柄或其他游戏控制器，以及力回馈设备等输入输出设备的数据。

按照文档的描述，它具备如下特征：
- 通过一个可以运行于后台的应用来获取数据，支持几乎所有类型的输入设备；
- 通过动作映射，应用无需知晓输入设备的具体种类；
- 适用于游戏、模拟以及其他 Windows 系统上的实时交互程序；
- 在使用键盘进行文本输入、使用鼠标进行定位等目标上，`DirectInput` 并不提供增益。

基于此，我们可以认为，`DirectInput` 是一个面向不同的设备，提供统一的输入事件抽象的接口组合。
后面我们会验证这个看法是否准确。

### 常见概念
- `DirectInput object`：`DirectInput` 根接口
- `Device`： 具体的输入设备
- `DirectInputDevice object`：具体输入设备的编码表示
- `Device object`：输入设备上的输入硬件/实体，如按键、扳机等

### 使用步骤
当我们需要实现一个基于 `DirectInput` API 的程序时，通常遵循以下步骤：

1. 创建 `DirectInput object`；
2. 枚举 `Devices`；
3. 为希望使用的 `Device` 创建 `DirectInputDevice object`；
4. 设置选定的 `Device`，诸如协作方式、数据格式、缓冲大小等；
5. 获取 `Device`，告知 `DirectInput` 已经准备好接收数据；
6. 获取数据——状态或事件
7. 响应数据
8. 释放与清理

此外，还可以通过动作映射。这样，我们可以专注于获取的事件，而不用关心事件来源的 `Device object`。

## 理解 DirectInput
主要是理解 `DirectInput` 的底层结构，以及与消息系统（Microsoft Windows message system）的关系

### DirectInput objects
一个仅输入的 `DirectInput` 实现包含：
- 提供 `IDirectInput8` 接口的 `DirectInput object`；
- 一个 `Component Object Model` 接口
- 每一个提供数据的输入设备提供一个 `DirectInputDevice object`；
- 每个 `DirectInputDevice object` 都会包含一些 `device object`。

`DirectInputDevice object` 是 [IDirectInputDevice8 Interface](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee417816(v=vs.85)) 实例。

应用通过 [IDirectInputDevice8::EnumObjects](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee417889(v=vs.85)) 方法获知输入设备上的 `device object` 数量与类型。

独立的 `device object` 使用 [DIDEVICEOBJECTINSTANCE](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee416612(v=vs.85)) 进行描述。


### 与 Windows 系统进行交互
- `DirectInput` 直接与设备驱动打交道，因此它会抑制或者干脆忽略系统中的鼠标与键盘消息。

   它同样也会忽略控制面板中的鼠标键盘设置，但会使用游戏控制器的校准设置。

- `DirectInput` 不会遵循键盘关于字符重复输入的设置。

   在接收缓冲数据的情况下，它仅仅会将按下与释放各视为一个独立的事件。

   而在使用即时数据的情况下，它仅仅会关注每个按键的物理状态。

- `DirectInput` 不会对输入字符进行转换或转译，它不对我们常说的修饰键如 `SHIFT` 等进行特别处理，也不根据系统语言区别处理输入数据。

- `DirectInput` 会绕过 Windows 系统对于窗口数据的解释系统，因此无法用于系统光标和导航。

- `DirectInput` 会忽略控制面板中的相关设置，但会遵循设备驱动中的设置。

综上，`DirectInput` 仅接收硬件和驱动层面控制的输入信号。


## DirectInput 与 XInput 对比
根据 [Comparison of XInput and DirectInput features](https://learn.microsoft.com/en-us/windows/win32/xinput/xinput-and-directinput) 和 [XInput 和 DirectInput 功能比较](https://learn.microsoft.com/zh-cn/windows/win32/xinput/xinput-and-directinput) 文档中的描述，可知：

XInput 是跨 Xbox 与 Windows 平台的输入标准，同样由 DirectX SDK 提供，相比 `DirectInput` 具备以下优点：
  1. 更易于使用，设置更少
  2. 跨平台
  3. 支持已安装的 Xbox 控制器
  4. 支持振动
  5. 支持尚未发布的针对 Xbox 主机的控制器

Xbox 控制器是可以通过 `DirectInput` api 使用的，但会缺失一些功能，如：
- 线性扳机
- 振动效果
- 查找耳机设备

如果应用仅支持 `XInput`，那么将无法使用旧的 `DirectInput` 设备。
同时支持两种标准可以避免这种情况。

但是由于 `XInput` 设备会同时被 `XInput` 和 `DirectInput` 感知到，因此需要甄别区分。
