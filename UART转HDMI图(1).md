UART转HDMI图像传输实验报告

## 一、实验目的

1. 掌握 ZYNQ7020 PS 与 PL 通过 AXI BRAM 进行数据交互的原理与实现方法
2. 理解 PS 端串口接收图像数据并解析写入 BRAM 的软件流程
3. 掌握 PL 端从 BRAM 读取图像并通过 HDMI 放大显示的硬件逻辑
4. 熟悉自定义 UART 图像传输协议的格式与通信机制
5. 学会 ZYNQ 软硬件协同设计的完整开发流程

## 二、实验环境

* 开发板：Xilinx ZYNQ7020 (xc7z020clg400-2)
* 开发工具：Vivado 2017.4、Xilinx SDK 2017.4
* PC 端环境：Anaconda 2024.02、Python 3.13、串口调试助手
* 显示设备：1280×720 分辨率 HDMI 显示器
* 辅助设备：JTAG 下载器、USB 转串口线、HDMI 数据线
* 实验工程：`sobel_03_uart_hdmi.xpr`

## 三、实验原理

### 3.1 系统整体数据流

本实验实现了 PC 到 ZYNQ 的串口图像传输与 HDMI 显示，完整数据流如下：

1. PC 端通过 USB 串口以 115200 波特率发送 128×72 RGB888 格式图像
2. ZYNQ PS 端 UART 外设接收字节流，解析帧头和行头协议
3. PS 端通过 AXI GP0 总线将解析后的像素数据写入 Block RAM
4. PL 端 HDMI 显示逻辑从 BRAM 的 PORTB 端口读取像素数据
5. 像素数据经过 10 倍最近邻插值放大后，通过 RGB2DVI IP 编码输出到 HDMI 显示器

### 3.2 硬件设计原理

#### 3.2.1 Block Design 架构

硬件平台基于 ZYNQ7 Processing System，通过 SmartConnect 连接 AXI BRAM Controller，控制一个 32 位宽、64KB 大小的双端口 Block RAM：

* ​**PORTA**​：连接 PS 端 AXI 总线，用于 PS 写入像素数据
* ​**PORTB**​：连接 PL 端 HDMI 逻辑，用于 PL 读取像素数据

BRAM 地址映射与像素格式：

* 基地址：`0x40000000`
* 地址范围：64KB
* 像素存储格式：`0x00RRGGBB`（32 位，低 24 位为 RGB888）
* 地址计算公式：`address = 0x40000000 + ((y * 128 + x) << 2)`

#### 3.2.2 PL 端 HDMI 显示逻辑

PL 端显示逻辑由以下模块组成：

1. `video_clock` IP：生成 74.25MHz 的 720p 标准视频时钟
2. `hdmi_bram_display.v`：产生标准 720p 显示时序，计算 BRAM 读取地址，实现 10 倍图像放大
3. `rgb2dvi_0` IP：将并行 RGB 信号转换为 TMDS 串行差分信号输出到 HDMI 接口

### 3.3 软件设计原理

#### 3.3.1 PS 端串口接收程序

PS 端`main.c`实现了完整的图像接收与解析功能：

1. 初始化 UART 外设，配置波特率为 115200
2. 等待并验证 7 字节帧头（`0x55 0xAA 0x80 0x00 0x48 0x00 0x18`）
3. 逐行解析 4 字节行头（`0x33 0xCC row_l row_h`）
4. 接收 128 个 RGB888 像素，写入对应的 BRAM 地址
5. 处理传输过程中的帧头错误、行号错误、超时等异常情况

#### 3.3.2 PC 端图像发送脚本

`camera_uart_sender.py`支持两种发送模式：

* ​**摄像头实时发送**​：从 PC 摄像头采集图像，缩放为 128×72 后按协议发送
* ​**单张图片发送**​：读取本地图片文件，转换为 RGB888 格式后发送

串口通信参数：

* 波特率：115200
* 数据位：8
* 校验位：无
* 停止位：1
* 流控：无

### 3.4 帧率分析

UART 115200 波特率下，每个字节传输需要 10 位（1 起始位 + 8 数据位 + 1 停止位），有效字节率为 11520 字节 / 秒。一帧 128×72 RGB888 图像大小为 27648 字节，加上帧头和行头约 27KB，理论最高帧率约 0.42fps。为保证传输稳定性，实际使用建议设置为 0.2fps（每 5 秒一帧）。

## 四、实验步骤

1. **打开 Vivado 工程** 打开路径为`D:\Github\FPGA-course\zynq7020-image-processing\sobel_03_uart_hdmi\sobel_03_uart_hdmi.xpr`的工程。若 Block Design 缺失，在 Tcl Console 中执行：
   ```tcl
   cd D:/Github/FPGA-course/zynq7020-image-processing/sobel_03_uart_hdmi
   source create_ps_uart_bram_hdmi_bd.tcl
   ```
2. **综合与实现** 在 Vivado 中依次执行：
   1. Run Synthesis（运行综合）
   2. Run Implementation（运行实现）
   3. Generate Bitstream（生成比特流）
3. **导出硬件到 SDK** 点击`File -> Export -> Export Hardware`，勾选`Include bitstream`，导出硬件平台到 SDK 工作区。
4. **编译并运行 PS 程序** 打开 SDK 后执行以下操作：
   1. 右键`ps_uart_bram_app_bsp` -> `Re-generate BSP Sources`
   2. 右键`ps_uart_bram_app` -> `Clean Project`
   3. 右键`ps_uart_bram_app` -> `Build Project`
   4. 点击`Xilinx -> Program FPGA`下载比特流
   5. 右键`ps_uart_bram_app` -> `Run As -> Launch on Hardware (System Debugger)`
5. **验证 PS 程序运行** 打开串口调试助手，配置为 115200 8N1，确认串口输出启动信息。
6. **配置 PC 端 Python 环境** 打开 Anaconda Prompt 执行：
   ```bash
   conda create -n fpga python=3.13 -y
   conda activate fpga
   cd D:\Github\FPGA-course\zynq7020-image-processing\host_camera_uart
   pip install -r requirements.txt
   ```
7. **发送图像并验证显示** 关闭串口调试助手，运行发送脚本：
   ```bash
   python camera_uart_sender.py --port COM4 --baud 115200 --image test.jpg --once --preview
   ```
   
   观察 HDMI 显示器是否显示发送的图像。
8. **扩展实验** 修改`hdmi_bram_display.v`，实现红色背景和白色边框功能。

## 五、实验结果与分析

### 5.1 综合与实现结果

综合和实现均成功完成，无任何错误和警告：

* 综合完成时间：2026/6/15 10:56 AM，耗时 38 秒
* 实现完成时间：2026/6/15 10:59 AM，耗时 3 分钟
* 比特流生成成功，可正常下载到开发板

​**综合完成界面截图**​：
![](fpga\03\微信图片_20260615105723_461_42.png)

### 5.2 资源利用率

各模块资源使用情况如下表所示：

| 模块名称                      | Slice LUTs | Slice Registers | F7 Muxes | Block RAM Tile | Bonded IOB | OLOGIC | BUFGCTRL | MMCM2\_ADV |
| ------------------------------- | ------------ | ----------------- | ---------- | ---------------- | ------------ | -------- | ---------- | ------------ |
| top                           | 2679       | 2367            | 5        | 16             | 10         | 8      | 4        | 1          |
| hdmi\_pl\_top\_i              | 246        | 209             | 3        | 0              | 0          | 8      | 3        | 1          |
| ps\_uart\_bram\_hdmi\_wrapper | 2433       | 2158            | 2        | 16             | 0          | 0      | 1        | 0          |
| 总计                          | 2679       | 2367            | 5        | 16             | 10         | 8      | 4        | 1          |

资源利用率较低（LUT 使用率 5%，寄存器使用率 2%），剩余大量资源可用于后续添加图像处理算法。

​**资源利用率截图**​：
![](fpga\03\微信图片_20260615105919_462_42.png)

### 5.3 时序分析

时序分析结果显示所有用户指定的时序约束均被满足：

* 最坏负裕量 (WNS)：8.117ns
* 总负裕量 (TNS)：0.000ns
* 最坏保持裕量 (WHS)：0.033ns
* 总保持裕量 (THS)：0.000ns
* 失败端点数量：0

充足的时序裕量保证了系统在 74.25MHz 视频时钟下的稳定运行。

​**时序总结截图**​：
![](fpga\03\微信图片_20260615110003_463_42.png)

### 5.4 PS 程序运行结果

下载比特流并运行 PS 程序后，SDK 终端成功输出启动信息：

```Plain
PS UART PL Sobel HDMI display
BRAM base: 0x40000000, baud: 115200, image: 128x72
waiting for frame header
waiting for frame header
waiting for frame header
```

`waiting for frame header`表示 PS 程序已正常启动，正在等待 PC 端发送图像帧。

​**SDK 串口输出截图**​：
![](fpga\03\微信图片_20260621121909_471_42.jpg)

### 5.5 基础实验显示结果

运行 PC 端发送脚本后，HDMI 显示器成功显示发送的图像：

* 图像清晰无闪烁，颜色准确
* 128×72 图像正确放大 10 倍显示在 1280×720 屏幕上
* 每次发送新图像，HDMI 画面会在约 2 秒后更新
![](fpga\03\微信图片_20260621122239_904_18.jpg)
### 5.6 扩展实验显示结果

通过修改`hdmi_bram_display.v`中的像素输出逻辑，实现了以下扩展功能：

1. ​**红色背景**​：将非图像区域的 RGB 输出从`24'h000000`改为`24'hFF0000`
2. ​**白色边框**​：在图像显示区域周围添加了宽度为 10 像素的白色边框

修改后显示效果：

* 屏幕背景变为纯红色
* 图像周围出现一圈均匀的白色边框
* 图像内容保持不变，显示正常

​**扩展实验显示效果截图**​：
![](fpga\03\03kz.png)

### 5.7 帧率与稳定性测试

测试了不同帧率参数下的传输稳定性：

| 帧率设置             | 实际帧率 | 传输稳定性 | 错误情况         |
| ---------------------- | ---------- | ------------ | ------------------ |
| 0.1fps（10 秒 / 帧） | \~0.1fps | 非常稳定   | 无错误           |
| 0.2fps（5 秒 / 帧）  | \~0.2fps | 稳定       | 偶尔出现行号错误 |
| 0.5fps（2 秒 / 帧）  | \~0.3fps | 不稳定     | 频繁出现超时错误 |

测试结果表明，在 115200 波特率下，0.2fps 是兼顾传输速度和稳定性的最佳选择。

## 六、实验总结

通过本次 UART 转 HDMI 图像传输实验，我掌握了 ZYNQ7020 PS 与 PL 协同设计的完整流程，理解了 AXI BRAM 在 PS-PL 数据交互中的核心作用，学会了 PS 端串口编程和 PL 端 HDMI 显示逻辑的设计方法。

