# Template_FSBL_LRUN

> STM32N6 FSBL Load & Run 项目 — 从编译到烧录的完整流程

## 项目概述

- **芯片**：STM32N657X0H3QU
- **开发板**：NUCLEO-N657X0-Q (MB1940)
- **架构**：FSBL LRUN（Load & Run）
  - FSBL 运行在内部 RAM，负责初始化 OTP、配置外部 Flash、把 Appli 复制到 RAM 并跳转执行
  - Appli 运行在内部 RAM，绿色 LED (PG.00) 每 250ms 闪烁一次

## 编译

```bash
cd build/Debug
cube-cmake --build . --target all
```

清理：
```bash
cube-cmake --build . --target clean
```

编译产物：

| 子项目 | 输出位置 |
|--------|----------|
| FSBL | `FSBL/build/Template_FSBL_LRUN_FSBL.elf/.bin/.hex` |
| Appli | `Appli/build/Template_FSBL_LRUN_Appli.elf/.bin/.hex` |

---

## 签名（必须）

所有要烧录到外部 Flash 的 .bin 文件都必须先用 STM32 Signing Tool 加 header。

### 工具路径

```
C:\APPS\Programmer\bin\STM32_SigningTool_CLI.exe
```

### 命令格式

```powershell
# Appli
& "C:\APPS\Programmer\bin\STM32_SigningTool_CLI.exe" -bin "<Appli.bin>" -nk -of 0x80000000 -t fsbl -o "<Appli-trusted.bin>" -hv 2.3 -align -s

# FSBL
& "C:\APPS\Programmer\bin\STM32_SigningTool_CLI.exe" -bin "<FSBL.bin>" -nk -of 0x80000000 -t fsbl -o "<FSBL-trusted.bin>" -hv 2.3 -align -s
```

### 参数说明

| 参数 | 说明 |
|------|------|
| `-nk` | No Key，开发模式 |
| `-of 0x80000000` | 输出地址偏移 |
| `-t fsbl` | FSBL 类型 header |
| `-hv 2.3` | Header version（N6 必须用 2.3） |
| `-align` | **必须加**（CubeProgrammer v2.21+ 必须，payload 对齐到 0x400） |
| `-s` | 静默模式（不弹出确认覆盖提示） |

---

## 烧录地址

| 文件 | 地址 |
|------|------|
| `Appli-trusted.bin` | `0x7010 0000` |
| `FSBL-trusted.bin` | `0x7000 0000`（可选） |

用 CubeProgrammer 的 **Open File** → **Download** 烧录。

---

## 启动模式

| 模式 | BOOT0 | BOOT1 | 说明 |
|------|-------|-------|------|
| DEV（调试） | 任意 | 2-3 | IDE 加载 FSBL 到 RAM，可源码调试 |
| Boot from Flash | 1-2 | 1-2 | 独立运行，无法源码调试 |

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

---

## OTP 配置（第一次必须）

本项目需要配置 **VDDIO3_HSLV**（XSPIM_P2 高速模式使能）。

### 第一次烧录流程

1. **删除 FSBL/mx-generated.cmake 中的 `NO_OTP_FUSE`**
2. 编译 FSBL
3. 签名 FSBL 和 Appli
4. DEV 模式：IDE 加载 FSBL.elf 到 RAM → **自动配置 OTP**
5. 烧录 Appli-trusted.bin 到 `0x7010 0000`
6. Reset → 运行

### 后续烧录

OTP 配置好后，编译时可以加回 `NO_OTP_FUSE`（跳过 OTP 配置步骤）。

---

## 调试方法

### DEV 模式调试（推荐）

1. BOOT1 = 2-3
2. IDE 加载 `FSBL/build/Template_FSBL_LRUN_FSBL.elf` 到 RAM
3. 在 Appli 代码打断点
4. Debug/Run
5. FSBL → 加载 Appli → 断点生效

### Flash 模式

无法源码调试，只能靠 LED / UART / ITM。

---

## 跳过 FSBL

可以。如果 OTP 已配置、芯片出厂自带 FSBL、或 boot ROM 支持直接加载，直接烧录 Appli-trusted.bin 到 `0x7010 0000` 即可。

---

## 关键文件

| 文件 | 说明 |
|------|------|
| `FSBL/mx-generated.cmake` | 编译宏定义，`NO_OTP_FUSE` 控制 OTP 配置 |
| `Template_FSBL_LRUN.ioc` | CubeMX 配置文件 |
| `SKILL.md` | 详细技能文档 |
| `readme.html` / `README.md` | 官方说明文档 |

---

## 工具链路径（绝对路径）

| 工具 | 路径 |
|------|------|
| Signing Tool | `C:\APPS\Programmer\bin\STM32_SigningTool_CLI.exe` |
| GCC ARM | `C:\Users\Q\AppData\Local\stm32cube\bundles\gnu-tools-for-stm32\14.3.1+st.2` |
| Middlewares | `C:\Users\Q\STM32Cube\Repository\STM32Cube_FW_N6_V1.3.0\` |

---

## 路径问题说明

本项目使用 **Windows Junction** 解决固件库路径问题：

```
C:/APPS/Drivers/BSP/STM32N6xx_Nucleo → C:/Users/Q/STM32Cube/Repository/STM32Cube_FW_N6_V1.3.0/Drivers/BSP/STM32N6xx_Nucleo
C:/APPS/Middlewares → C:/Users/Q/STM32Cube/Repository/STM32Cube_FW_N6_V1.3.0/Middlewares
```

换电脑后需要重新创建这些 junction。

---

## 注意事项

1. **Middlewares 路径是绝对路径**，换电脑后需要重新配置
2. **OTP 配置不可逆**，第一次确认后再烧录
3. **签名必须加 `-align`**，否则芯片无法识别
4. **第一次必须禁用 `NO_OTP_FUSE`**，让 FSBL 自动配置 OTP
5. **签名推荐加 `-s` 静默模式**，避免弹出覆盖确认提示