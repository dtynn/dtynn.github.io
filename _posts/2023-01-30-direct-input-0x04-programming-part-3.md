# DirectInput 阅读理解：使用 (3)
此前，我们已经可以感知到游戏控制器，并进行操作如信息获取等。

接下来我们将要尝试与游戏控制器进行数据交互。

## 数据交互

### 数据类型
根据 [Buffered and Immediate Data](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee416236(v=vs.85)) 文档中的描述可知，
存在两种数据类型：
- Buffered data，类似事件流
- Immediate data，是状态快照

### 交互方式
常见的交互方式有两种：
- Polling：定期轮询
- Event Notification：事件通知

其中，`Event Notification` 的方式需要基于 windows 的消息系统，可以感知以下状态变化：
- 任一轴（axis)的位置变化
- 按键的状态变化，按下/释放
- POV 的方向变化
- 设备失联

## 实例
在之前代码的基础上，我们需要：
- 参考 [windows-rs/samples/windows/kernel_event](https://github.com/microsoft/windows-rs/blob/1685363e52e0be9052937096ba8605efb0a23a11/crates/samples/windows/kernel_event/src/main.rs)，了解 Event 机制；
- 参考 [Device Setup](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee416596(v=vs.85)) 进行完整的设备初始化；
- 使用 [IDirectInputDevice8::GetDeviceData](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee417894(v=vs.85)) 获取设备状态变更
- 相应状态变化数据

### 思考
代码比较长，先说结论。

在测试过程发现了一些现象和问题：
- 事件中的对象与硬件实体的对应关系
- 相比按键的触发、释放，轴位置的数据比较复杂，如何处理

这也引发了我的思考：
在真正的应用中，是否需要将按键事件和轴位置事件放进两个不同的事件循环，以不同的灵敏度进行侦测并相应。

### 代码
最终代码如下：

```rust
use core::ffi::c_void;
use std::mem::size_of;
use std::ptr::null_mut;

use windows::{
    core::{Interface, Vtable, GUID, PCWSTR},
    Win32::{
        Devices::HumanInterfaceDevice::*,
        Foundation::{CloseHandle, BOOL, HANDLE, WAIT_OBJECT_0},
        System::LibraryLoader::GetModuleHandleW,
        System::Threading::{CreateEventW, WaitForSingleObject},
    },
};

type Result<T, E = Box<dyn std::error::Error>> = std::result::Result<T, E>;

// from https://github.com/glfw/glfw/blob/master/deps/mingw/dinput.h
const DIDFT_OPTIONAL: u32 = 0x80000000;
const NULL: *const GUID = ref_guid(0);

const DIPROP_BUFFERSIZE: RefGUID = ref_guid(1);
const DIPROP_AXISMODE: RefGUID = ref_guid(2);

const DIPROPAXISMODE_ABS: u32 = 0;
const DIPROPAXISMODE_REL: u32 = 1;

const DIPROP_GRANULARITY: RefGUID = ref_guid(3);
const DIPROP_RANGE: RefGUID = ref_guid(4);
const DIPROP_DEADZONE: RefGUID = ref_guid(5);
const DIPROP_SATURATION: RefGUID = ref_guid(6);
const DIPROP_FFGAIN: RefGUID = ref_guid(7);
const DIPROP_FFLOAD: RefGUID = ref_guid(8);
const DIPROP_AUTOCENTER: RefGUID = ref_guid(9);

const DIPROPAUTOCENTER_OFF: u32 = 0;
const DIPROPAUTOCENTER_ON: u32 = 1;

const DIPROP_CALIBRATIONMODE: RefGUID = ref_guid(10);

const DIPROPCALIBRATIONMODE_COOKED: u32 = 0;
const DIPROPCALIBRATIONMODE_RAW: u32 = 1;

const DIPROP_CALIBRATION: RefGUID = ref_guid(11);
const DIPROP_GUIDANDPATH: RefGUID = ref_guid(12);
const DIPROP_INSTANCENAME: RefGUID = ref_guid(13);
const DIPROP_PRODUCTNAME: RefGUID = ref_guid(14);

const DIPROP_JOYSTICKID: RefGUID = ref_guid(15);
const DIPROP_GETPORTDISPLAYNAME: RefGUID = ref_guid(16);

const DIPROP_PHYSICALRANGE: RefGUID = ref_guid(18);
const DIPROP_LOGICALRANGE: RefGUID = ref_guid(19);

const DIPROP_KEYNAME: RefGUID = ref_guid(20);
const DIPROP_CPOINTS: RefGUID = ref_guid(21);
const DIPROP_APPDATA: RefGUID = ref_guid(22);
const DIPROP_SCANCODE: RefGUID = ref_guid(23);
const DIPROP_VIDPID: RefGUID = ref_guid(24);
const DIPROP_USERNAME: RefGUID = ref_guid(25);
const DIPROP_TYPENAME: RefGUID = ref_guid(26);

const DEVICE_DATA_BUFFER_SIZE: usize = 8;

type RefGUID = *const GUID;

const fn ref_guid(num: u32) -> RefGUID {
    num as RefGUID
}

// from https://sourceforge.net/p/mingw-w64/mingw-w64/ci/ce5a9f624dfc691082dad2ea2af7b1985e3476b5/tree/mingw-w64-crt/libsrc/dinput_joy2.c
const RGODF_DI_JOY2_NUM: usize = 164;
const RGODF_DI_JOY2: [DIOBJECTDATAFORMAT; RGODF_DI_JOY2_NUM] = [
    DIOBJECTDATAFORMAT {
        pguid: &GUID_XAxis,
        dwOfs: 0x00,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTPOSITION,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_YAxis,
        dwOfs: 0x04,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTPOSITION,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_ZAxis,
        dwOfs: 0x08,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTPOSITION,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_RxAxis,
        dwOfs: 0x0c,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTPOSITION,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_RyAxis,
        dwOfs: 0x10,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTPOSITION,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_RzAxis,
        dwOfs: 0x14,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTPOSITION,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_Slider,
        dwOfs: 0x18,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTPOSITION,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_Slider,
        dwOfs: 0x1c,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTPOSITION,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_POV,
        dwOfs: 0x20,
        dwType: DIDFT_POV | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_POV,
        dwOfs: 0x24,
        dwType: DIDFT_POV | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_POV,
        dwOfs: 0x28,
        dwType: DIDFT_POV | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_POV,
        dwOfs: 0x2c,
        dwType: DIDFT_POV | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x30,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x31,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x32,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x33,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x34,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x35,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x36,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x37,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x38,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x39,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x3a,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x3b,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x3c,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x3d,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x3e,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x3f,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x40,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x41,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x42,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x43,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x44,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x45,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x46,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x47,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x48,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x49,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x4a,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x4b,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x4c,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x4d,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x4e,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x4f,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x50,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x51,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x52,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x53,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x54,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x55,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x56,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x57,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x58,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x59,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x5a,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x5b,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x5c,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x5d,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x5e,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x5f,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x60,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x61,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x62,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x63,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x64,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x65,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x66,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x67,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x68,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x69,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x6a,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x6b,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x6c,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x6d,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x6e,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x6f,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x70,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x71,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x72,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x73,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x74,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x75,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x76,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x77,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x78,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x79,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x7a,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x7b,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x7c,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x7d,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x7e,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x7f,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x80,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x81,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x82,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x83,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x84,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x85,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x86,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x87,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x88,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x89,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x8a,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x8b,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x8c,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x8d,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x8e,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x8f,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x90,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x91,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x92,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x93,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x94,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x95,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x96,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x97,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x98,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x99,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x9a,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x9b,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x9c,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x9d,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x9e,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0x9f,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0xa0,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0xa1,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0xa2,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0xa3,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0xa4,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0xa5,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0xa6,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0xa7,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0xa8,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0xa9,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0xaa,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0xab,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0xac,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0xad,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0xae,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: NULL,
        dwOfs: 0xaf,
        dwType: DIDFT_BUTTON | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: 0x0,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_XAxis,
        dwOfs: 0xb0,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTVELOCITY,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_YAxis,
        dwOfs: 0xb4,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTVELOCITY,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_ZAxis,
        dwOfs: 0xb8,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTVELOCITY,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_RxAxis,
        dwOfs: 0xbc,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTVELOCITY,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_RyAxis,
        dwOfs: 0xc0,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTVELOCITY,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_RzAxis,
        dwOfs: 0xc4,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTVELOCITY,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_Slider,
        dwOfs: 0x18,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTVELOCITY,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_Slider,
        dwOfs: 0x1c,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTVELOCITY,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_XAxis,
        dwOfs: 0xd0,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTACCEL,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_YAxis,
        dwOfs: 0xd4,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTACCEL,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_ZAxis,
        dwOfs: 0xd8,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTACCEL,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_RxAxis,
        dwOfs: 0xdc,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTACCEL,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_RyAxis,
        dwOfs: 0xe0,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTACCEL,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_RzAxis,
        dwOfs: 0xe4,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTACCEL,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_Slider,
        dwOfs: 0x18,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTACCEL,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_Slider,
        dwOfs: 0x1c,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTACCEL,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_XAxis,
        dwOfs: 0xf0,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTFORCE,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_YAxis,
        dwOfs: 0xf4,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTFORCE,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_ZAxis,
        dwOfs: 0xf8,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTFORCE,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_RxAxis,
        dwOfs: 0xfc,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTFORCE,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_RyAxis,
        dwOfs: 0x100,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTFORCE,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_RzAxis,
        dwOfs: 0x104,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTFORCE,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_Slider,
        dwOfs: 0x18,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTFORCE,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_Slider,
        dwOfs: 0x1c,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTFORCE,
    },
];

const DF_DI_JOYSTICK2: DIDATAFORMAT = DIDATAFORMAT {
    dwSize: size_of::<DIDATAFORMAT>() as u32,
    dwObjSize: size_of::<DIOBJECTDATAFORMAT>() as u32,
    dwFlags: DIDF_ABSAXIS,
    dwDataSize: size_of::<DIJOYSTATE2>() as u32,
    dwNumObjs: RGODF_DI_JOY2.len() as u32,
    rgodf: RGODF_DI_JOY2.as_ptr() as *mut DIOBJECTDATAFORMAT,
};

pub struct JoySticks {
    di: IDirectInput8W,
    devices: Vec<JoyStickDevice>,
}

impl JoySticks {
    fn new(di: IDirectInput8W) -> Self {
        JoySticks {
            di,
            devices: Vec::new(),
        }
    }

    unsafe fn get_all_devices(&mut self) -> Result<()> {
        let ptr = self as *mut JoySticks as *mut c_void;
        self.di.EnumDevices(
            DI8DEVCLASS_GAMECTRL,
            Some(enum_device_callback),
            ptr,
            DIEDFL_ATTACHEDONLY,
        )?;
        Ok(())
    }

    unsafe fn add_device(&mut self, info: DIDEVICEINSTANCEW) -> Result<()> {
        let pname = PCWSTR::from_raw(info.tszProductName.as_ptr()).to_string()?;
        let iname = PCWSTR::from_raw(info.tszInstanceName.as_ptr()).to_string()?;

        let mut maybe_device = None;
        self.di
            .CreateDevice(&info.guidInstance, &mut maybe_device, None)?;
        let di = maybe_device.ok_or(format!(
            "unable to create device with {:?}",
            info.guidInstance
        ))?;

        let jdev = JoyStickDevice::new(pname, iname, info, di)?;

        self.devices.push(jdev);
        Ok(())
    }
}

pub struct JoyStickDevice {
    _raw: DIDEVICEINSTANCEW,
    di: IDirectInputDevice8W,
    cap: DIDEVCAPS,
    product_name: String,
    instance_name: String,
    event_handler: HANDLE,

    objects: Vec<JoyStickDeviceObject>,
}

impl Drop for JoyStickDevice {
    fn drop(&mut self) {
        if let Err(e) = unsafe { self.deinit() } {
            println!("failed to unacquire device: {:?}", e);
        };
        println!("device unacquired");
    }
}

impl JoyStickDevice {
    unsafe fn new(
        pname: String,
        iname: String,
        raw: DIDEVICEINSTANCEW,
        di: IDirectInputDevice8W,
    ) -> Result<Self> {
        let event_handler = CreateEventW(None, false, true, None)?;
        let mut jdev = JoyStickDevice {
            _raw: raw,
            di,
            cap: DIDEVCAPS {
                dwSize: size_of::<DIDEVCAPS>() as u32,
                ..Default::default()
            },
            product_name: pname,
            instance_name: iname,
            event_handler,
            objects: Vec::new(),
        };

        jdev.get_all_objects()?;
        jdev.initialize()?;

        Ok(jdev)
    }

    unsafe fn set_property(&mut self, prop: RefGUID, how: u32, val: u32) -> Result<()> {
        let mut val = DIPROPDWORD {
            diph: DIPROPHEADER {
                dwSize: size_of::<DIPROPDWORD>() as u32,
                dwHeaderSize: size_of::<DIPROPHEADER>() as u32,
                dwObj: 0,
                dwHow: how,
            },
            dwData: val,
        };

        self.di
            .SetProperty(prop, &mut val.diph as *mut DIPROPHEADER)?;

        Ok(())
    }

    unsafe fn set_device_property(&mut self, prop: RefGUID, val: u32) -> Result<()> {
        self.set_property(prop, DIPH_DEVICE, val)
    }

    unsafe fn initialize(&mut self) -> Result<()> {
        self.di
            .SetDataFormat(&DF_DI_JOYSTICK2 as *const DIDATAFORMAT as *mut DIDATAFORMAT)?;
        self.di.GetCapabilities(&mut self.cap as *mut DIDEVCAPS)?;

        self.set_device_property(DIPROP_BUFFERSIZE, DEVICE_DATA_BUFFER_SIZE as u32)?;
        self.set_device_property(DIPROP_AXISMODE, DIPROPAXISMODE_ABS)?;

        self.di.SetEventNotification(Some(self.event_handler))?;
        self.di.Acquire()?;

        println!("device acquired");
        Ok(())
    }

    unsafe fn deinit(&mut self) -> Result<()> {
        self.di.Unacquire()?;
        self.di.SetEventNotification(None)?;
        CloseHandle(self.event_handler);
        Ok(())
    }

    unsafe fn get_all_objects(&mut self) -> Result<()> {
        let ptr = self as *mut JoyStickDevice as *mut c_void;
        self.di
            .EnumObjects(Some(enum_device_object_callback), ptr, DIDFT_ALL)?;
        Ok(())
    }

    unsafe fn add_object(&mut self, info: DIDEVICEOBJECTINSTANCEW) -> Result<()> {
        let name = PCWSTR::from_raw(info.tszName.as_ptr()).to_string()?;

        let obj = JoyStickDeviceObject::new(name, info);
        self.objects.push(obj);
        Ok(())
    }

    fn di_poll_required(&self) -> bool {
        self.cap.dwFlags & DIDC_POLLEDDATAFORMAT != 0
    }

    unsafe fn wait_for_buffered_states(
        &mut self,
        timeout_millis: Option<u32>,
    ) -> Result<([DIDEVICEOBJECTDATA; DEVICE_DATA_BUFFER_SIZE], usize, bool)> {
        match WaitForSingleObject(self.event_handler, timeout_millis.unwrap_or(u32::MAX)) {
            WAIT_OBJECT_0 => {}
            other => return Err(format!("unexpected event signal {:?}", other).into()),
        };

        let mut objects: [DIDEVICEOBJECTDATA; DEVICE_DATA_BUFFER_SIZE] = Default::default();
        let mut read_size = objects.len() as u32;

        match self.di.GetDeviceData(
            size_of::<DIDEVICEOBJECTDATA>() as u32,
            objects.as_mut_ptr(),
            &mut read_size,
            0,
        ) {
            Ok(_) => Ok((objects, read_size as usize, false)),
            Err(e) if e.code().0 == DI_BUFFEROVERFLOW => Ok((objects, read_size as usize, true)),
            Err(e) => Err(e.into()),
        }
    }
}

pub struct JoyStickDeviceObject {
    _raw: DIDEVICEOBJECTINSTANCEW,
    name: String,
}

impl JoyStickDeviceObject {
    fn new(name: String, raw: DIDEVICEOBJECTINSTANCEW) -> Self {
        JoyStickDeviceObject { _raw: raw, name }
    }
}

unsafe extern "system" fn enum_device_callback(
    infop: *mut DIDEVICEINSTANCEW,
    extra: *mut c_void,
) -> BOOL {
    if !infop.is_null() {
        let stick_ptr = extra.cast::<JoySticks>();
        if let Err(e) = (*stick_ptr).add_device(*infop) {
            println!("failed to add device: {:?}", e);
        }
    } else {
        println!("null device info");
    }
    BOOL::from(true)
}

unsafe extern "system" fn enum_device_object_callback(
    infop: *mut DIDEVICEOBJECTINSTANCEW,
    extra: *mut c_void,
) -> BOOL {
    if !infop.is_null() {
        let device_ptr = extra.cast::<JoyStickDevice>();
        if let Err(e) = (*device_ptr).add_object(*infop) {
            println!("failed to add object: {:?}", e);
        }
    } else {
        println!("null object info");
    }
    BOOL::from(true)
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    unsafe {
        let hinstance = GetModuleHandleW(None)?;
        println!("get hinstance {:?}", hinstance);

        let mut res = null_mut();
        let res_ptr: *mut *mut c_void = &mut res as *mut *mut c_void;

        let refiid = <IDirectInput8W as Interface>::IID;

        println!("DIRECT VERSION: {:x}", DIRECTINPUT_VERSION);
        println!("IID: {:?}", refiid);

        DirectInput8Create(hinstance, DIRECTINPUT_VERSION, &refiid, res_ptr, None)?;

        let di = IDirectInput8W::from_raw(res);
        println!("created!");

        println!("start to enum attached gamectrl devices");
        let mut sticks = JoySticks::new(di);
        if let Err(e) = sticks.get_all_devices() {
            println!("get all devices: {:?}", e);
        }

        for (id, device) in sticks.devices.iter_mut().enumerate() {
            println!(
                "#{:03} {}:{}, data format poll required: {}",
                id,
                device.product_name,
                device.instance_name,
                device.di_poll_required(),
            );

            for (io, object) in device.objects.iter().enumerate() {
                println!("\t#{:03} {}", io, object.name);
            }

            let mut captured = 0;
            loop {
                if captured >= 10 {
                    break;
                }

                match device.wait_for_buffered_states(Some(3000)) {
                    Ok(r) => {
                        if r.1 != 0 {
                            for state in r.0.into_iter().take(r.1) {
                                // cares only about butoo press & release
                                if state.dwOfs >= 0x30 {
                                    println!(
                                        "button 0x{:04x}: {}",
                                        state.dwOfs,
                                        if state.dwData == 0 {
                                            "released"
                                        } else {
                                            "pressed"
                                        }
                                    );
                                    captured += 1;
                                }
                            }
                        }
                    }
                    Err(e) => {
                        println!("failed to wait for states: {:?}", e);
                    }
                };
            }
        }

        println!("finished");

        Ok(())
    }
}
```

使用这段示例代码，可以捕获并输出手柄上按键的按下和释放，测试输出如下：
```
get hinstance HINSTANCE(140695589355520)
DIRECT VERSION: 800
IID: BF798031-483A-4DA2-AA99-5D64ED369700
created!
start to enum attached gamectrl devices
device acquired
#000 Wireless Controller:Wireless Controller, data format poll required: false
        #000 Z 旋转
        #001 Z 轴
        #002 Y 轴
        #003 X 轴
        #004 帽状开关
        #005 按钮 0
        #006 按钮 1
        #007 按钮 2
        #008 按钮 3
        #009 按钮 4
        #010 按钮 5
        #011 按钮 6
        #012 按钮 7
        #013 按钮 8
        #014 按钮 9
        #015 按钮 10
        #016 按钮 11
        #017 按钮 12
        #018 按钮 13
        #019 Y 旋转
        #020 X 旋转
        #021 集合 0 - 游戏板
button 0x0035: pressed
button 0x0035: released
button 0x0034: pressed
button 0x0034: released
button 0x0036: pressed
button 0x0036: released
button 0x0037: pressed
button 0x0037: released
button 0x0031: pressed
button 0x0031: released
finished
device unacquired
```
