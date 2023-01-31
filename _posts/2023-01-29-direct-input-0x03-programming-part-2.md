# DirectInput 阅读理解：使用 (2)
在前一篇文章中，我们了解了一下开发环境，并完成了一个简单的demo。

接下来我们将继续深入了解 `DirectInput` 相关接口的使用。


## 获取设备信息
我们改造一下之前的demo，使之罗列并输出已连接的游戏控制器设备信息，为我们之后获取设备事件做准备：


```rust
use core::ffi::c_void;
use std::mem::size_of;
use std::ptr::null_mut;

use windows::{
    core::{Interface, Vtable, GUID, PCWSTR},
    Win32::{
        Devices::HumanInterfaceDevice::*, Foundation::BOOL, System::LibraryLoader::GetModuleHandleW,
    },
};

type Result<T, E = Box<dyn std::error::Error>> = std::result::Result<T, E>;

const DIDFT_OPTIONAL: u32 = 0x80000000;
const NULL: *const GUID = 0 as *const GUID;

// from https://sourceforge.net/p/mingw-w64/mingw-w64/ci/ce5a9f624dfc691082dad2ea2af7b1985e3476b5/tree/mingw-w64-crt/libsrc/dinput_joy2.c
const RGODF_DI_JOY2_NUM: usize = 164;
const RGODF_DI_JOY2: [DIOBJECTDATAFORMAT; RGODF_DI_JOY2_NUM] = [
    DIOBJECTDATAFORMAT {
        pguid: &GUID_XAxis,
        dwOfs: 0x0,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTPOSITION,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_YAxis,
        dwOfs: 0x4,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTPOSITION,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_ZAxis,
        dwOfs: 0x8,
        dwType: DIDFT_ABSAXIS | DIDFT_RELAXIS | DIDFT_ANYINSTANCE | DIDFT_OPTIONAL,
        dwFlags: DIDOI_ASPECTPOSITION,
    },
    DIOBJECTDATAFORMAT {
        pguid: &GUID_RxAxis,
        dwOfs: 0xc,
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

    objects: Vec<JoyStickDeviceObject>,
}

impl Drop for JoyStickDevice {
    fn drop(&mut self) {
        if let Err(e) = unsafe { self.di.Unacquire() } {
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
        let mut jdev = JoyStickDevice {
            _raw: raw,
            di,
            cap: DIDEVCAPS {
                dwSize: size_of::<DIDEVCAPS>() as u32,
                ..Default::default()
            },
            product_name: pname,
            instance_name: iname,
            objects: Vec::new(),
        };

        jdev.initialize()?;
        jdev.get_all_objects()?;

        Ok(jdev)
    }

    unsafe fn initialize(&mut self) -> Result<()> {
        self.di
            .SetDataFormat(&DF_DI_JOYSTICK2 as *const DIDATAFORMAT as *mut DIDATAFORMAT)?;
        self.di.Acquire()?;

        self.di.GetCapabilities(&mut self.cap as *mut DIDEVCAPS)?;
        println!("device acquired");
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

        for (id, device) in sticks.devices.iter().enumerate() {
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
        }

        println!("finished");

        Ok(())
    }
}
```

这段 demo 组合使用了
- [IDirectInput8::EnumDevices](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee417804(v=vs.85))
- [IDirectInput8::CreateDevice](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee417803(v=vs.85))
- [IDirectInputDevice8::GetCapabilities](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee417892(v=vs.85))
- [IDirectInputDevice8::SetDataFormat](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee417925(v=vs.85))
- [IDirectInputDevice8::Acquire](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee417818(v=vs.85))
- [IDirectInputDevice8::Unacquire](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee417931(v=vs.85))
- [IDirectInputDevice8::EnumObjects](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee417889(v=vs.85))

等接口，定义了 `JoySticks -> JoyStickDevice -> JoyStickDeviceObject` 三层抽象，并演示了通过 Callback 传递对象引用（指针）的方式。

在我的开发机器上，程序执行将输出：

```
get hinstance HINSTANCE(140699685421056)
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
finished
device unacquired
```

其中的 `Wireless Controller` 就是我的 DS4 手柄。
