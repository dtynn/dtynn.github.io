# DirectInput 阅读理解：使用
接下来我们将跟随 [Using DirectInput](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee416845(v=vs.85)) 即编程指引，尝试了解如何使用 `DirectInput`。

## 代码说明
文档中所使用的代码多使用 c++/c#。考虑到我们的目标，我会尝试使用 rust 来进行替换实现。

在此简单地介绍一下开发环境：
- 基于 Windows11 上的 WSL2

  使用的发行版本为 Ubuntu-20.04，内核版本

  `5.15.79.1-microsoft-standard-WSL2 #1 SMP Wed Nov 23 01:01:46 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux`

- 使用的 rust 版本为 `stable-x86_64-unknown-linux-gnu`，具体为 `1.65.0`；

- 为了在 linux 环境下交叉编译 windows 的结果，需要：
  - 添加 target `x86_64-pc-windows-msvc`
  - 使用 [xwin](https://github.com/Jake-Shadle/xwin) 添加必要的工具链，可参考 [1.6 跨平台编译 Cross-Compilation](https://zhuanlan.zhihu.com/p/516115248)
  - 通过 `cargo build --release --target x86_64-pc-windows-msvc` 这样的方式交叉编译，或使用 [cargo-xwin](https://github.com/rust-cross/cargo-xwin)

- 使用 [windows 库](https://crates.io/crates/windows) 调用接口

  使用 [winapi 库](https://crates.io/crates/winapi) 获取一些常量


## 初始化/创建 DirectInput 实例
通过 [DirectInput8Create 函数](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee416756(v=vs.85)) 创建 [IDirectInput8 Interface](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee417799(v=vs.85)) 实例。

```rust
use core::ffi::c_void;
use std::ptr::null_mut;

use windows::{
    core::GUID,
    Win32::{
        Devices::HumanInterfaceDevice::{DirectInput8Create, IDirectInput8W, DIRECTINPUT_VERSION},
        System::LibraryLoader::GetModuleHandleW,
    },
};

// value from https://docs.rs/winapi/latest/winapi/um/dinput/constant.IID_IDirectInput8W.html
const IID_IDIRECT_INPUT_8W: GUID = GUID::from_values(
    0xbf798031,
    0x483a,
    0x4da2,
    [0xaa, 0x99, 0x5d, 0x64, 0xed, 0x36, 0x97, 0x00],
);

fn main() -> Result<(), Box<dyn std::error::Error>> {
    unsafe {
        let mut p = null_mut::<IDirectInput8W>();
        let pptr: *mut *mut IDirectInput8W = &mut p;
        let hinstance = GetModuleHandleW(None)?;
        println!("get hinstance {:?}", hinstance);

        println!("DIRECT VERSION: {:x}", DIRECTINPUT_VERSION);
        println!("IID_IDirectInput8W: {:?}", IID_IDIRECT_INPUT_8W);

        DirectInput8Create(
            hinstance,
            DIRECTINPUT_VERSION,
            &IID_IDIRECT_INPUT_8W,
            pptr as *mut *mut c_void,
            None,
        )?;

        println!("created!");

        Ok(())
    }
}

```

并获得类似
```
get hinstance HINSTANCE(140697735069696)
DIRECT VERSION: 800
IID_IDirectInput8W: BF798031-483A-4DA2-AA99-5D64ED369700
created!
```
这样的输出结果。

## 遍历 `DirectInput` 设备
这里我们将使用 [IDirectInput8::EnumDevices 方法](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee417804(v=vs.85))
