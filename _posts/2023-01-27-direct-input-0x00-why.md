# DirectInput 阅读理解： WHY
想用手柄当作 RoboMasterEP 的控制器，大致列出了推进的步骤：
1. 在 Windows 中侦测到手柄、捕获手柄的按键事件；
2. 将 Windows 系统中的手柄映射到 WSL 中；
3. 将手柄的按键事件通过 SDK 映射成 RoboMasterEP 的控制指令；

鼓捣了一番，在第一步就发现了几个问题。

## 问题
首先说说问题。

直观的表现就是，我的手柄使用蓝牙连接到系统为 Win11 的电脑上时，使用 `XInput` 接口的 rust 库无法侦测出手柄。
然而，在玩 Steam 中的游戏，或者《原神》时，DS4 都是能被正常识别、使用的。

在看了一些文章
- [Wiki: Controller:DualShock_4](https://www.pcgamingwiki.com/wiki/Controller:DualShock_4)
- [How to use a DualShock 4 PS4 controller on PC](https://www.pcgamer.com/how-to-use-a-ps4-controller-on-pc/)


后，得出的初步结论是：在 Windows 上， 我自己的 DualShock 4 手柄似乎原生使用的是 `DirectInput` 接口，而不是较新一些的 `XInput` 接口。

## 解题
第一步就是去查找支持 `DirectInput` 的 rust 库。

遗憾的是，通过[关键词`directinput`搜索](https://crates.io/search?q=directinput) 仅获得一个有帮助的结果：[dhc](https://crates.io/crates/dhc)。
它存在一些非常致命的问题：
1. 最后更新于三年前，缺乏维护
2. 使用了一些 cpp 代码来接入 windows 接口，同时编译方式上有些繁琐
3. 这个库是作为一个独立的程序/dll，为 `DirectInput` 设备提供转接层，而我更需要的是一个简单的接入层


这样看来，我可能需要自行编写一个 `DirectInput` 手柄的接入库。
在一番尝试之后，我没有找到什么相对便捷的途径。
这意味着，我很可能需要从头阅读 `DirectInput` 的[文档](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee416842(v=vs.85))。


在搜索过程中，我还发现了其他一些见过但没研究过的概念：
- SDL(Simple DirectMedia Layer) 2.0
- HID(Human Interface Devices)
- libusb/hidapi
- libudev
