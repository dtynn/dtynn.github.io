# DirectInput 阅读理解：使用 (4)
在这一篇中，我们将尝试继续深入了解不同的 Device Object 的输入事件，以及它们和实际硬件的对应关系。

## 观测结果
### 结论
代码比较长，同样先从结论开始。

以 ds4 的输入情况来看：

- 十字键在 `DirectInput` 中，是POV(Point-Of-View) Object。

  虽然在 ds4 上它们看起来像是四个独立的按键，但是在接入层实际是一个整体，输入为 [无，上，下，左，右，左上，右上，左下，右下]
  这样9个枚举值。

- 左右摇杆都是 Axis Object，但是并非严格对应成以GUID标识的 XAxis/YAxis 和 RYAxis/RYAxis；行程对应的输入值范围是 [0, 65535]，远点位置为 32767

- L2 和 R2 都会同时映射出一个 Button Object 和 一个 Axis Object，Button 的输入表示触发与否，Axis 的输入表示触发行程，值范围为 [0, 65535]

- 触摸板的触摸事件暂时没有找到对应事件，而触摸板的按下/释放被映射成一个 Button Object

- 其他实体都被映射成 Button Object

至此，作为对 `DirectInput` 以及自己的手柄设备的基本了解，这一系列 demo 代码的演变和解读已经基本达成我们的目标。

接下来我会尝试基于 `DirectInput` 进行封装，这个过程中的问题和解决可能会另开一个系列的文章进行记录。

### 代码
最终的观测代码：
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

        jdev.initialize()?;
        jdev.get_all_objects()?;

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
        // self.set_device_property(DIPROP_AUTOCENTER, DIPROPAUTOCENTER_OFF)?;

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

#[derive(Debug)]
pub enum DeviceObjectType {
    XAxis,
    YAxis,
    ZAxis,
    RxAxis,
    RyAxis,
    RzAxis,
    Slider,
    Button,
    Key,
    POV,
    Unknown(GUID),
}

impl From<GUID> for DeviceObjectType {
    fn from(v: GUID) -> Self {
        match v {
            GUID_XAxis => Self::XAxis,
            GUID_YAxis => Self::YAxis,
            GUID_ZAxis => Self::ZAxis,
            GUID_RxAxis => Self::RxAxis,
            GUID_RyAxis => Self::RyAxis,
            GUID_RzAxis => Self::RzAxis,
            GUID_Slider => Self::Slider,
            GUID_Button => Self::Button,
            GUID_Key => Self::Key,
            GUID_POV => Self::POV,
            GUID_Unknown => Self::Unknown(v),
            other => Self::Unknown(other),
        }
    }
}

#[derive(Debug)]
pub enum DeviceObjectFlag {
    NONE,
    ASPECTACCEL,
    ASPECTFORCE,
    ASPECTMASK,
    ASPECTPOSITION,
    ASPECTVELOCITY,
    FFACTUATOR,
    FFEFFECTTRIGGER,
    GUIDISUSAGE,
    POLLED,
    Unknown(u32),
}

impl From<u32> for DeviceObjectFlag {
    fn from(v: u32) -> Self {
        match v {
            0 => Self::NONE,
            DIDOI_ASPECTACCEL => Self::ASPECTACCEL,
            DIDOI_ASPECTFORCE => Self::ASPECTFORCE,
            DIDOI_ASPECTMASK => Self::ASPECTMASK,
            DIDOI_ASPECTPOSITION => Self::ASPECTPOSITION,
            DIDOI_ASPECTVELOCITY => Self::ASPECTVELOCITY,
            DIDOI_FFACTUATOR => Self::FFACTUATOR,
            DIDOI_FFEFFECTTRIGGER => Self::FFEFFECTTRIGGER,
            DIDOI_GUIDISUSAGE => Self::GUIDISUSAGE,
            DIDOI_POLLED => Self::POLLED,
            other => Self::Unknown(other),
        }
    }
}

pub struct JoyStickDeviceObject {
    _raw: DIDEVICEOBJECTINSTANCEW,
    typ: DeviceObjectType,
    flag: DeviceObjectFlag,
    name: String,
}

impl JoyStickDeviceObject {
    fn new(name: String, raw: DIDEVICEOBJECTINSTANCEW) -> Self {
        JoyStickDeviceObject {
            _raw: raw,
            typ: raw.guidType.into(),
            flag: raw.dwFlags.into(),
            name,
        }
    }
}

#[derive(Debug, Clone, Copy)]
enum POVInput {
    None,
    Up,
    Down,
    Left,
    Right,
    UpAndLeft,
    UpAndRight,
    DownAndLeft,
    DownAndRight,
    Unknown(u32),
}

impl From<u32> for POVInput {
    fn from(v: u32) -> Self {
        match v as i32 {
            -1 => Self::None,
            0 => Self::Up,
            18000 => Self::Down,
            27000 => Self::Left,
            9000 => Self::Right,
            31500 => Self::UpAndLeft,
            4500 => Self::UpAndRight,
            22500 => Self::DownAndLeft,
            13500 => Self::DownAndRight,
            _other => Self::Unknown(v),
        }
    }
}

#[repr(u8)]
#[derive(Debug, Clone, Copy)]
enum ButtonInput {
    Pressed = 1,
    Released = 0,
}

impl From<u32> for ButtonInput {
    fn from(v: u32) -> Self {
        if v == 0 {
            Self::Released
        } else {
            Self::Pressed
        }
    }
}

#[derive(Debug, Clone, Copy)]
struct TriggerInput(u32);

impl From<u32> for TriggerInput {
    fn from(v: u32) -> Self {
        TriggerInput(v)
    }
}

#[derive(Debug, Clone, Copy)]
struct AxisInput(i32);

impl From<u32> for AxisInput {
    fn from(v: u32) -> Self {
        Self(v as i32 - 32767)
    }
}

#[derive(Debug, Clone, Copy)]
enum DeviceObjectInput {
    Pov(POVInput),
    Button(ButtonInput),
    Trigger(TriggerInput),
    Axis(AxisInput),
    Slider(u32),
    Unknown(u32, u32),
}

impl From<(u32, u32)> for DeviceObjectInput {
    fn from(v: (u32, u32)) -> Self {
        match v.0 {
            0x00 | 0x04 | 0x08 | 0x14 => Self::Axis(v.1.into()),
            0x0c | 0x10 => Self::Trigger(v.1.into()),
            0x18 | 0x1c => Self::Slider(v.1),
            0x20 | 0x24 | 0x28 => Self::Pov(v.1.into()),
            0x30..=0xaf => Self::Button(v.1.into()),
            other => Self::Unknown(other, v.1),
        }
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

        let device = match sticks.devices.last_mut() {
            Some(d) => d,
            None => {
                return {
                    println!("no device found");
                    Ok(())
                }
            }
        };

        println!(
            "{}:{}, data format poll required: {}",
            device.product_name,
            device.instance_name,
            device.di_poll_required(),
        );

        for (io, object) in device.objects.iter().enumerate() {
            println!(
                "\t#{:03} {}({:?}-{:?}), offset: 0x{:04x}",
                io, object.name, object.typ, object.flag, object._raw.dwOfs
            );
        }

        let mut states = std::collections::HashMap::new();

        let mut captured = 0;
        loop {
            if captured > 150 {
                break;
            }

            match device.wait_for_buffered_states(Some(3000)) {
                Ok(r) => {
                    if r.1 != 0 {
                        for state in r.0.into_iter().take(r.1) {
                            // cares only about butoo press & release
                            let input = DeviceObjectInput::from((state.dwOfs, state.dwData));
                            let prev = states.insert(state.dwOfs, (state.dwData, input));
                            match prev {
                                None => {
                                    println!("0x{:04x} new state: {:?}", state.dwOfs, input);
                                    captured += 1;
                                }

                                Some((abs, ai @ DeviceObjectInput::Axis(_))) => {
                                    if state.dwData.abs_diff(abs) > 5000 {
                                        println!(
                                            "0x{:04x} state changed: {:?} => {:?}",
                                            state.dwOfs, ai, input
                                        );
                                        captured += 1;
                                    }
                                }

                                Some((abs, ai @ DeviceObjectInput::Trigger(_))) => {
                                    if state.dwData.abs_diff(abs) > 500 {
                                        println!(
                                            "0x{:04x} state changed: {:?} => {:?}",
                                            state.dwOfs, ai, input
                                        );
                                        captured += 1;
                                    }
                                }

                                Some(other) => {
                                    println!(
                                        "0x{:04x} state changed: {:?} => {:?}",
                                        state.dwOfs, other.1, input
                                    );
                                    captured += 1;
                                }
                            }
                        }
                    }
                }
                Err(e) => {
                    println!("failed to wait for states: {:?}", e);
                }
            };
        }

        println!("finished");

        Ok(())
    }
}
```
