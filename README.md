# 双通道 PDM 霍尔角度传感器系统

本工程基于 Zynq-7000（`xc7z020clg400-1`）实现霍尔角度信号采集、处理与输出。外部前端提供两路 1 bit PDM 码流及数据有效信号；PL 侧完成 CIC 解码、补偿、滤波、归一化、相位处理、角度计算与插值校准，并通过 Hall 接口输出 PWM 或 SENT 信号。PS 侧通过 AXI-Lite 配置和读取寄存器，使用 UART ASCII 协议 v6 与电脑端上位机通信。

工程采用 Vivado/Vitis 2024.2。设计顶层为 `angle_sensor_system_wrapper`，目标器件为 `xc7z020clg400-1`。主要文件如下：

| 目录 | 内容 |
| --- | --- |
| `src/bd/`、`src/ip/`、`src/constraints/` | Block Design、自定义 AXI IP 和引脚约束；Vivado 重建使用这些文件。 |
| `src/rtl/` | 便于阅读的 RTL 文件；其中与自定义 IP 同名的模块不应重复加入 Vivado 工程。 |
| `src/ps/`、`src/host/` | Zynq PS standalone 程序和 Python 上位机。 |
| `sim/` | CIC 与系统 wrapper 的测试平台、导入脚本和波形文件。 |
| `build/` | Vivado 重建脚本、综合与实现报告。 |
| `board/` | bitstream、XSA、ELF、接线说明及上板结果。 |
| `data/` | 分度头和温度测试数据，按原始表、校准前和校准后分类。 |
| `skill/` | PS 串口与 AXI-Lite 链路排障流程。 |

## 复现步骤

### 1. 从源码建立 Vivado 工程

在本目录打开终端，使用 Vivado 2024.2 执行以下命令。参数是一个新的工程输出目录，请换成当前电脑上的实际路径：

```powershell
vivado -mode batch -source build/scripts/rebuild_project.tcl -tclargs C:/vivado_build/hall_angle_sensor
```

脚本注册 `src/ip/` 中的自定义 IP，导入 BD 和 XDC，生成 `angle_sensor_system_wrapper`，然后运行综合、实现和 bitstream 生成。生成的 Vivado 工程位于指定的输出目录。需要在 GUI 中查看仿真时，打开该工程并按 [`sim/README.md`](sim/README.md) 导入测试平台。

### 2. 连接并运行板级系统

按照 [`board/setup/hardware_setup.md`](board/setup/hardware_setup.md) 连接两路 PDM 输入、`data_vaild_0`、UART 和 Hall 输出，接好电源与公共地。使用 Vivado Hardware Manager 将 [`board/artifacts/angle_sensor_system_wrapper.bit`](board/artifacts/angle_sensor_system_wrapper.bit) 下载到 FPGA。随后在 Vitis 中基于随包提供的 [`angle_sensor_system_wrapper.xsa`](board/artifacts/angle_sensor_system_wrapper.xsa) 配置 Zynq standalone 平台，通过 JTAG 运行 [`angle_sensor_app.elf`](board/artifacts/angle_sensor_app.elf)。PS 应保持运行状态。

### 3. 启动上位机并检查输出

电脑端需要带 Tk 的 Python 3，以及 `pyserial`、`numpy`、`matplotlib`、`pillow`：

```powershell
python -m pip install pyserial numpy matplotlib pillow
python src/host/angle_sensor_ps_gui.py
```

在上位机选择实际串口并打开连接；串口参数为 **115200 baud、8N1**。使用“连接检测”检查 PS 协议 v6，再读取寄存器状态或采集 CIC/DSP 数据。设置 PWM/SENT 模式后，可在 `hall_out_0` 测量输出波形。上板照片及波形位于 [`board/results/`](board/results/)，分度头与温度测试表位于 [`data/`](data/)，资源和时序报告位于 [`build/reports/`](build/reports/)。
