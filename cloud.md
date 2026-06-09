# 云端路径配置说明

> 本文档记录本项目依赖的固件库路径配置，供克隆后重新配置参考。

## 固件库存放路径

```text
C:/Users/Q/STM32Cube/Repository/STM32Cube_FW_N6_V1.3.0/
```

本项目依赖 STM32Cube_FW_N6_V1.3.0 固件库，克隆后需确保该路径存在并包含完整固件内容。

## 原来使用的 Junction路径（已废弃）

```text
C:/APPS/Drivers/BSP/STM32N6xx_Nucleo → 固件库 Drivers/BSP/STM32N6xx_Nucleo
C:/APPS/Middlewares                    → 固件库 Middlewares
```

>⚠️ Junction 容易断裂，已改用绝对路径。

## 当前 cmake 路径配置

### FSBL — `FSBL/mx-generated.cmake`

```cmake
# Middlewares（STM32_ExtMem_Manager）路径
C:/Users/Q/STM32Cube/Repository/STM32Cube_FW_N6_V1.3.0/Middlewares/ST/STM32_ExtMem_Manager
```

### Appli — `Appli/mx-generated.cmake`

```cmake
# BSP（STM32N6xx_Nucleo）路径
C:/Users/Q/STM32Cube/Repository/STM32Cube_FW_N6_V1.3.0/Drivers/BSP/STM32N6xx_Nucleo
```

## 换电脑后配置步骤

1.确认固件库存放路径（如上）
2. 修改 `FSBL/mx-generated.cmake` 中的 Middlewares 路径（全局替换 `C:/Users/Q/STM32Cube` 为实际路径）
3. 修改 `Appli/mx-generated.cmake` 中的 BSP 路径
4. 重新编译验证

## 相关文件

- [FSBL/mx-generated.cmake](../FSBL/mx-generated.cmake)
- [Appli/mx-generated.cmake](../Appli/mx-generated.cmake)
- [CLAUDE.md](../CLAUDE.md) — 项目总体说明