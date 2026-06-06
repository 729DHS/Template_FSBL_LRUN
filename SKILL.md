# Template_FSBL_LRUN 项目技能

> STM32N6 FSBL Load & Run 项目 — 从编译到烧录的完整流程

---

## 项目概述

这是一个 **FSBL LRUN (Load & Run)** 模板，用于 STM32N657X0H3QU (NUCLEO-N657X0-Q)：

- **FSBL**：运行在内部 RAM，负责初始化 OTP、配置外部 Flash、把 Appli 复制到 RAM 并跳转执行
- **Appli**：实际应用代码，运行在内部 RAM，绿色 LED (PG.00) 每 250ms 闪烁一次

---

## 编译

```bash
# 清理并重新编译
cd build/Debug
cube-cmake --build . --target clean
cube-cmake --build . --target all
```

编译产物：

| 文件 | 位置 |
|------|------|
| FSBL.elf / .bin / .hex | `FSBL/build/` |
| Appli.elf / .bin / .hex | `Appli/build/` |

---

## 签名

所有要烧录到外部 Flash 的 .bin 文件都必须先用 STM32 Signing Tool 加 header。

### 签名工具路径

```
C:\APPS\Programmer\bin\STM32_SigningTool_CLI.exe
```

### 签名命令

```powershell
# Appli
& "C:\APPS\Programmer\bin\STM32_SigningTool_CLI.exe" -bin "C:\APPS\COPRO\MCU\CUBEMX\N6_MX\Template_FSBL_LRUN\Appli\build\Template_FSBL_LRUN_Appli.bin" -nk -of 0x80000000 -t fsbl -o "C:\APPS\COPRO\MCU\CUBEMX\N6_MX\Template_FSBL_LRUN\Appli\build\Template_FSBL_LRUN_Appli-trusted.bin" -hv 2.3 -align -s

# FSBL
& "C:\APPS\Programmer\bin\STM32_SigningTool_CLI.exe" -bin "C:\APPS\COPRO\MCU\CUBEMX\N6_MX\Template_FSBL_LRUN\FSBL\build\Template_FSBL_LRUN_FSBL.bin" -nk -of 0x80000000 -t fsbl -o "C:\APPS\COPRO\MCU\CUBEMX\N6_MX\Template_FSBL_LRUN\FSBL\build\Template_FSBL_LRUN_FSBL-trusted.bin" -hv 2.3 -align -s
```

### 参数说明

| 参数 | 含义 |
|------|------|
| `-bin` | 输入的 .bin 文件 |
| `-nk` | No Key（开发模式，不做真正的签名验证） |
| `-of 0x80000000` | 输出地址偏移 |
| `-t fsbl` | 生成 FSBL 类型的 header |
| `-hv 2.3` | Header version 2.3（N6 必须用这个） |
| `-align` | **必须加**（CubeProgrammer v2.21+ 必须，payload 对齐到 0x400） |
| `-s` | 静默模式（不弹出确认覆盖提示） |

---

## 烧录地址

| 文件 | 地址 | 说明 |
|------|------|------|
| `Appli-trusted.bin` | `0x7010 0000` | Appli 烧录位置 |
| `FSBL-trusted.bin` | `0x7000 0000` | FSBL 烧录位置（可选） |

用 CubeProgrammer 的 **Open File** + **Download** 烧录。

---

## 启动模式

### LRUN（Load & Run）

程序从外部 Flash **复制到** 内部 RAM（0x34000400），然后从 RAM 执行。
- FSBL 干的就是这个活：初始化 XSPI → 把 Appli 复制到 RAM → 跳转
- Appli 的 `.bin/.hex` 需要签名后烧录

### XSPI（eXecute in Place）

程序不复制，直接在外部 Flash 里原地执行，CPU 通过 XSPI 接口直接取指。
- ROM 地址：`0x90100400`（外部 XSPI Flash 基地址 0x9010_0000 + header 0x400）
- Appli 的 linker 文件用 `STM32N657XX_ROMxspi*.ld`

### 链接文件名规则

格式：`ROM在哪里_RUN_RAM在哪里`

| 链接文件 | 含义 |
|---------|------|
| `STM32N657XX_LRUN` | Appli 放在内部 RAM（由 FSBL 加载） |
| `STM32N657XX_ROMxspi1` | Appli 在外部 XSPI1 Flash，原地执行 |
| `STM32N657XX_LRUN_RAMxspi2` | Appli 放 RAM，但从 XSPI2 的 Flash 加载 |
| `STM32N657XX_ROMxspi1_RAMxspi3` | Appli 在 XSPI1，RAM 用 XSPI3 |

### DEV 模式（调试用）

- **BOOT1 = 2-3**，BOOT0 任意
- FSBL 通过 IDE 加载到内部 RAM 运行
- Appli 烧到 `0x7010 0000`
- 可以源码调试 FSBL

### Boot from Flash 模式（独立运行）

- **BOOT0 = 1-2**，BOOT1 = 1-2
- FSBL 和 Appli 都烧到 Flash
- 上电自动运行，无法源码调试

---

## OTP 配置（第一次必须）

### 什么是 OTP

One-Time Programmable fuse。一次写入，不可逆。
本项目需要配置 **VDDIO3_HSLV**（XSPIM_P2 高速模式使能）。

### 第一次烧录流程

```
1. FSBL 编译时 **禁用 NO_OTP_FUSE**（让 FSBL 自动配置 OTP）
   → 修改 FSBL/mx-generated.cmake，删除 NO_OTP_FUSE 行

2. 签名 FSBL-trusted.bin

3. DEV 模式：IDE 加载 FSBL.elf 到 RAM 运行
   → FSBL 会自动配置 OTP

4. 烧录 Appli-trusted.bin 到 0x7010 0000

5. Reset，Appli 开始运行
```

### 之后的烧录流程

OTP 配置好后，编译时可以**加回 NO_OTP_FUSE**（跳过 OTP 配置步骤）：
- 修改 FSBL/mx-generated.cmake，添加 NO_OTP_FUSE 行
- 签名 → 烧录 → 运行

---

## 调试方法

### DEV 模式调试（推荐）

1. BOOT1 = 2-3（DEV 模式）
2. IDE 加载 `FSBL/build/Template_FSBL_LRUN_FSBL.elf` 到 RAM
3. 在 Appli 代码打断点
4. Debug/Run
5. FSBL 运行 → 加载 Appli → 断点生效

### 无调试模式

Flash 模式无法源码调试，只能靠：
- **LED 闪烁**（观察程序是否运行）
- **UART 输出**（初始化调试串口）
- **ITM/SWO**（如果配置了）

---

## 常见问题

### Q: 编译后没有生成 .bin 和 .hex？

检查 `FSBL/CMakeLists.txt` 和 `Appli/CMakeLists.txt` 末尾是否有 post-build 命令：

```cmake
find_program(OBJCOPY arm-none-eabi-objcopy PATHS
    C:/Users/Q/AppData/Local/stm32cube/bundles/gnu-tools-for-stm32/14.3.1+st.2/bin
    NO_DEFAULT_PATH
)

add_custom_command(TARGET ${CMAKE_PROJECT_NAME} POST_BUILD
    COMMAND ${OBJCOPY} -O binary $<TARGET_FILE:${CMAKE_PROJECT_NAME}> ${CMAKE_BINARY_DIR}/${CMAKE_PROJECT_NAME}.bin
    COMMENT "Generating binary"
)
add_custom_command(TARGET ${CMAKE_PROJECT_NAME} POST_BUILD
    COMMAND ${OBJCOPY} -O ihex $<TARGET_FILE:${CMAKE_PROJECT_NAME}> ${CMAKE_BINARY_DIR}/${CMAKE_PROJECT_NAME}.hex
    COMMENT "Generating HEX"
)
```

### Q: 签名时报 "binary file does not exist"？

检查路径是否正确，PowerShell 里要用 `& "..."` 或在 CMD 里运行。

### Q:签名时弹出 "Do you want to replace this file?"？

加 `-s` 参数（静默模式）即可。

### Q: Boot from Flash 模式不运行？

1. 检查 `-align` 参数是否加了
2. 检查地址是否正确（0x7000 0000 / 0x7010 0000）
3. 检查签名命令是否正确
4. OTP 是否配置好（第一次需要）

### Q: 芯片无响应 (Target is not responding)？

1. 按 Reset 板子
2. 重新连接调试器
3. 试试 Mass Erase 后再烧录

---

## 跳过了 FSBL 可以吗？

可以。如果：
- OTP 已经配置好
- 芯片出厂自带 FSBL 在 0x7000 0000
- 或者 boot ROM 支持直接加载

可以直接烧录 Appli-trusted.bin 到 0x7010 0000，Reset 后芯片会从外部 Flash 启动。

---

## 流程速查

### 完整烧录（第一次）

```
1. 删除 FSBL/mx-generated.cmake 中的 NO_OTP_FUSE
2. 编译
3. 签名 FSBL 和 Appli（加 -s 静默模式）
4. DEV 模式：IDE 加载 FSBL.elf 到 RAM 运行（配置 OTP）
5. 烧录 Appli-trusted.bin 到 0x7010 0000
6. Reset → 运行
```

### 后续开发

```
1. 编译
2. 签名（加 -s）
3. 烧录 Appli-trusted.bin 到 0x7010 0000
4. （可选）烧录 FSBL-trusted.bin 到 0x7000 0000
5. Boot from Flash 模式：BOOT0=1-2, BOOT1=1-2, Reset
   或 DEV 模式：BOOT1=2-3, IDE 加载 FSBL.elf
```