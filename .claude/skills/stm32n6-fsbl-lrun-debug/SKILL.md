---
name: stm32n6-fsbl-lrun-debug
description: |
  STM32N6 FSBL+Appli LRUN 项目调试技能。当用户提到以下情况时触发：
  - 配置 VS Code 调试 STM32N6（FSBL/Appli/LRUN）
  - launch.json、stlinkgdbtarget、stldr 配置
  - STM32N6 调试断点不生效、SWD 断联
  - FSBL+Appli 双项目调试流程
  - CubeProgrammer 签名、External Loader 选择
  - NUCLEO-N657 vs DK 板区别
---

# STM32N6 FSBL + Appli LRUN 调试技能

## 适用场景

- 芯片：STM32N657X0HxQ
- 开发板：**NUCLEO-N657X0-Q**（New Clone 板，非 DK 板）
- 架构：**FSBL LRUN**（Load & Run）
  - FSBL 运行在内部 AXISRAM2（`0x34180400`），负责初始化 XSPI → 把 Appli 复制到内部 RAM → 跳转
  - Appli 运行在内部 RAM（`0x34000000`），绿色 LED PG.0 每 250ms 闪烁

---

## 快速参考

### 工具链路径

| 工具 | 路径 |
|------|------|
| GCC ARM | `C:\Users\Q\AppData\Local\stm32cube\bundles\gnu-tools-for-stm32\14.3.1+st.2` |
| CubeProgrammer | `C:\APPS\Programmer` |
| ST-LINK GDB Server | `C:\Users\Q\AppData\Local\stm32cube\bundles\stlink-gdbserver\7.13.0+st.3\bin` |
| Signing Tool | `C:\APPS\Programmer\bin\STM32_SigningTool_CLI.exe` |
| SVD | `C:\APPS\Programmer\SVD\STM32N657.svd` |

### 编译命令

```bash
# 编译 FSBL + Appli（通过 CMake 扩展 Build 按钮，或命令行）
cd build/Debug
cube-cmake --build . --target all

# 清理
cube-cmake --build . --target clean
```

### 烧录地址

| 文件 | 地址 |
|------|------|
| FSBL-trusted.bin | `0x70000000`（可选） |
| Appli-trusted.bin | `0x70100000` |

---

## launch.json 配置（stlinkgdbtarget）

> **重要：使用 `stlinkgdbtarget` 类型（ST-LINK GDB Server），不要用 `cortex-debug`**。教程原文配置即为此类型。

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "stlinkgdbtarget",
            "request": "launch",
            "name": "NUCLEO-N657X0-Q Debug (FSBL + Appli)",
            "origin": "snippet",
            "cwd": "${workspaceFolder}",
            "preBuild": "${command:st-stm32-ide-debug-launch.build}",
            "runEntry": "BOOT_Application",
            "svdPath": "C:\\APPS\\Programmer\\SVD\\STM32N657.svd",
            "imagesAndSymbols": [
                {
                    "symbolFileName": "${workspaceFolder}/Appli/build/Template_FSBL_LRUN_Appli.elf",
                    "imageFileName": "${workspaceFolder}/Appli/build/Appli-trusted.elf"
                },
                {
                    "symbolFileName": "${workspaceFolder}/FSBL/build/Template_FSBL_LRUN_FSBL.elf",
                    "imageFileName": "${workspaceFolder}/FSBL/build/Template_FSBL_LRUN_FSBL.elf"
                }
            ],
            "serverExtLoader": [
                {
                    "loader": "C:\\APPS\\Programmer\\bin\\ExternalLoader\\MX25UM51245G_STM32N6570-NUCLEO.stldr",
                    "initialize": false
                }
            ]
        }
    ]
}
```

### 字段说明

| 字段 | 值 | 说明 |
|------|-----|------|
| `type` | `stlinkgdbtarget` | ST-LINK GDB Server 类型，教程标准配置 |
| `runEntry` | `BOOT_Application` | 在 XSPI 内存映射完成后停（外部 Flash 就绪后） |
| `svdPath` | `STM32N657.svd` | 外设寄存器查看 |
| `imagesAndSymbols[0]` | Appli.elf / Appli-trusted.elf | Appli 符号 + 外部 Flash 镜像 |
| `imagesAndSymbols[1]` | FSBL.elf | FSBL 符号 + RAM 镜像 |
| `serverExtLoader.loader` | `MX25UM51245G_STM32N6570-NUCLEO.stldr` | **NUCLEO 板**的 External Loader |

### ⚠️ stldr 文件选择（NUCLEO vs DK）

| 开发板 | stldr 文件 |
|--------|-----------|
| **NUCLEO-N657X0-Q**（New Clone） | `MX25UM51245G_STM32N6570-NUCLEO.stldr` |
| STM32N6570-DK | `MX66UW1G45G_STM32N6570-DK.stldr` |

**如果板子是 NUCLEO-N657X0-Q，切勿使用 DK 的 stldr**，Flash 芯片不同会导致烧录失败。

### ⚠️ svdPath 路径前缀

`svdPath` 和 `serverExtLoader.loader` 中的路径前缀需要根据实际安装目录替换。`...\\STM32CubeProgrammer\\` 是占位符，应替换为实际路径（如 `C:\\APPS\\Programmer\\`）。

---

## CMakeLists.txt 后构建步骤

### FSBL/CMakeLists.txt

在签名步骤后，增加 `.bin → .elf` 转换（供 `stlinkgdbtarget` 的 `imageFileName` 使用）：

```cmake
# Post-build: sign .bin for external Flash
find_program(SIGNING_TOOL C:/APPS/Programmer/bin/STM32_SigningTool_CLI.exe)
set(FSBL_OUTPUT "${CMAKE_BINARY_DIR}/FSBL-trusted.bin")
add_custom_command(TARGET ${CMAKE_PROJECT_NAME} POST_BUILD
    COMMAND ${SIGNING_TOOL} -bin ${CMAKE_BINARY_DIR}/${CMAKE_PROJECT_NAME}.bin -nk -of 0x80000000 -t fsbl -o ${FSBL_OUTPUT} -hv 2.3 -align -s
    COMMENT "Signing FSBL binary -> FSBL-trusted.bin"
)

# Post-build: convert signed FSBL .bin back to .elf (for debugger)
add_custom_command(TARGET ${CMAKE_PROJECT_NAME} POST_BUILD
    COMMAND ${OBJCOPY} -I binary -O elf32-littlearm -B arm --change-addresses=0x70000000 ${FSBL_OUTPUT} ${CMAKE_BINARY_DIR}/FSBL-trusted.elf
    COMMENT "Converting FSBL-trusted.bin -> FSBL-trusted.elf @ 0x70000000"
)
```

### Appli/CMakeLists.txt

同上，输出文件名为 `Appli-trusted.bin/elf`，地址为 `0x70100000`。

### 签名参数说明

| 参数 | 说明 |
|------|------|
| `-nk` | No Key，开发模式 |
| `-of 0x80000000` | 输出地址偏移 |
| `-t fsbl` | FSBL 类型 header |
| `-hv 2.3` | Header version（N6 必须用 2.3） |
| `-align` | **必须加**，CubeProgrammer v2.21+ 要求 payload 对齐到 0x400 |
| `-s` | 静默模式，不弹出覆盖确认 |

---

## LRUN 模式调试断点失效的根因

### 现象

在 Appli 的 `main()` 里设了断点，但 F5 运行时灯亮了（代码在跑）但断点不停。只有手动暂停一次后，断点才生效。

### 根因

调试器在**启动时**一次性设好所有断点。Appli 的断点地址指向内部 RAM（`0x3400xxxx`），但此时 Appli 还在外部 Flash，尚未被 FSBL 复制到 RAM——那片 RAM 是空的，断点设不上。

FSBL 跑完 → 复制 Appli 到 RAM → 跳转执行 → 但断点早已失效。

手动暂停后，调试器重新扫描内存，发现代码已在 RAM 中，补设断点成功，所以继续运行就能命中。

### 正确的调试流程

1. F5 启动 → 停在 `BOOT_Application`（XSPI 映射完成，外部 Flash 就绪）
2. F5 继续 → FSBL 加载 Appli → LED 开始闪烁
3. **手动暂停一次** → 调试器在 Appli 代码所在 RAM 补齐断点
4. 在 Appli 代码设断点 → F5 继续 → 命中 ✓

> 这是 LRUN 模式固有的时序问题，XIP 模式不存在（Appli 代码地址上电即可访问）。

---

## 启动模式

| 模式 | BOOT0 | BOOT1 | 说明 |
|------|-------|-------|------|
| DEV（调试） | 任意 | 2-3 | IDE 加载 FSBL 到 RAM，可源码调试 |
| Boot from Flash | 1-2 | 1-2 | 独立运行，无法源码调试 |

调试时确保 **BOOT1 = 2-3**。

---

## 第一次烧录（OTP 配置）

1. 删除 FSBL/mx-generated.cmake 中的 `NO_OTP_FUSE`
2. 编译 FSBL
3. 签名 FSBL 和 Appli
4. DEV 模式：IDE 加载 FSBL.elf 到 RAM → **自动配置 OTP**
5. 烧录 Appli-trusted.bin 到 `0x70100000`
6. Reset → 运行

后续烧录可加回 `NO_OTP_FUSE`（跳过 OTP 配置步骤）。

---

## 常见问题

### Q: GDB Server 启动失败 "Failed to launch ST-LINK GDB Server: Error: spawn ST-LINK_gdbserver.exe ENOENT"
A: `searchDir` 中 `C:\ST` 目录不存在，删除它。确保 `serverpath` 指向正确的 ST-LINK_gdbserver.exe 路径。

### Q: 调试时 "Target is not responding"
A: 通常是 Appli 的 SysTick 未正常工作导致死循环，或 D-Cache 开启导致 SWD halt 时卡死。先检查 SysTick 初始化，或尝试在 Appli 的 while(1) 前加 `SCB_DisableDCache();`。

### Q: stldr 加载失败
A: 确认使用的是 `MX25UM51245G_STM32N6570-NUCLEO.stldr`（NUCLEO 板），不是 DK 板的 `MX66UW1G45G_STM32N6570-DK.stldr`。