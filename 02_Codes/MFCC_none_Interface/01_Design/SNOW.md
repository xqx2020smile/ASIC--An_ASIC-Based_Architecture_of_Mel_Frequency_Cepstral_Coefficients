# MFCC ASIC 设计（无接口版本）

基于 Verilog HDL RTL 级的音频信号处理梅尔频率倒谱系数（MFCC）特征提取硬件实现。

## 项目概述

本项目是"基于 ASIC 的梅尔频率倒谱系数（MFCC）架构"研究的一部分,由越南胡志明市科学技术部资助。目标是开发用于越南语自动语音识别的人工神经网络（ANN）数字硬件架构。

该实现提供了一个可重配置并适应实时应用的 **MFCC 动态 VLSI 架构**。该设计针对 ASIC 实现进行了优化,通过完整的 MFCC 处理流水线处理音频信号,从预加重到二阶增量系数。

这是 MFCC 硬件设计的**无接口版本**,不包含 AHB 总线接口（独立处理核心）。

## 技术栈

- **硬件描述语言**: Verilog HDL (IEEE 1364)
- **设计级别**: RTL（寄存器传输级）
- **目标平台**: ASIC 实现
- **运算精度**: IEEE 754 32位浮点
- **架构类型**: 同步时序设计,带状态机

### 核心设计特性

- **浮点运算单元**:
  - 32位 IEEE 754 浮点加法器
  - 32位 IEEE 754 浮点乘法器
  - 浮点对数和指数运算
  
- **存储器架构**:
  - 系数存储器（窗函数、FFT 旋转因子、Mel 滤波器、倒谱）
  - 输入/输出数据存储器
  - 双端口存储器控制器以优化流水线

## 项目结构

```
MFCC_none_Interface/01_Design/
│
├── 00_Coefficients/          # 预计算系数存储器
│   ├── mem_input_data.v      # 输入语音数据存储器
│   ├── mem_window_cof.v      # 窗函数系数
│   ├── mem_w_real.v          # FFT 旋转因子（实部）
│   ├── mem_w_image.v         # FFT 旋转因子（虚部）
│   ├── mem_mel_cof_*.v       # Mel 滤波器组系数（8个滤波器）
│   ├── mem_cepstral_cof.v    # 倒谱变换的 DCT 系数
│   ├── mem_exponent.v        # 指数查找表
│   └── mem_mantissa.v        # 尾数查找表
│
├── 00_Common/                # 共享运算和控制模块
│   ├── floating_point_adder.v      # IEEE 754 浮点加法器
│   ├── floating_point_multiple.v   # IEEE 754 浮点乘法器
│   ├── add_fp_clk.v                # 时钟化浮点加法封装
│   ├── mul_fp_clk.v                # 时钟化浮点乘法封装
│   ├── log_fp_clk.v                # 浮点对数运算
│   ├── max_min_fp_clk.v            # 最小/最大值比较
│   ├── counter*.v                  # 各种计数器模块
│   ├── cla_*bit.v                  # 超前进位加法器（6-15位）
│   ├── mem_ctrl.v                  # 存储器控制器
│   └── change_addr*.v              # 地址映射/转换
│
├── 01_Pre_emphasis/          # 预加重滤波器（高通）
│   ├── pre_emphasis.v        # 主预加重模块
│   └── preem_state_ctrl.v    # 状态机控制器
│
├── 02_Window/                # 帧加窗（Hamming/Hanning）
│   ├── window.v              # 加窗模块
│   └── window_state_ctrl.v   # 状态机控制器
│
├── 03_FFT/                   # 快速傅里叶变换
│   ├── top_fft.v             # FFT 顶层模块
│   ├── fft_core.v            # FFT 蝶形运算核心
│   ├── fft_core_control.v    # FFT 级控制器
│   ├── arrange.v             # 位反转输入排序
│   ├── prepare.v             # 数据准备
│   ├── network_control.v     # 蝶形网络控制
│   └── control_floating_point.v  # 浮点操作序列控制
│
├── 04_Amplitude/             # 幅度谱计算
│   ├── amplitude.v           # 幅度计算（sqrt(real²+imag²)）
│   ├── amplitude_state_ctrl.v    # 状态机
│   ├── max_pair_compare.v    # 最大值跟踪器
│   └── min_pair_compare.v    # 最小值跟踪器
│
├── 05_Mel/                   # Mel 刻度滤波器组
│   ├── mel.v                 # Mel 滤波器组应用
│   └── mel_state_ctrl.v      # 状态机控制器
│
├── 06_Ceptrum/               # 倒谱系数（DCT）
│   ├── ceptrum.v             # 离散余弦变换
│   └── ceptrum_state_ctrl.v  # 状态机控制器
│
├── 07_Delta/                 # 一阶增量（速度）
│   ├── delta.v               # 增量系数计算
│   └── delta_state_ctrl.v    # 状态机控制器
│
├── 08_Delta_2nd/             # 二阶增量（加速度）
│   ├── delta_2nd.v           # 二阶增量计算
│   └── delta_2nd_state_ctrl.v    # 状态机控制器
│
├── top/                      # 顶层集成
│   └── top.v                 # 完整 MFCC 系统集成
│
└── top_ctrl/                 # 系统控制和时序
    ├── top_ctrl.v            # 主控制器
    ├── top_state.v           # 顶层状态机
    ├── copy_energy.v         # 能量系数处理
    └── copy_energy_state_ctrl.v  # 能量拷贝状态机
```

## 核心功能

### MFCC 处理流水线

1. **预加重** - 高通滤波器,放大高频成分
2. **帧加窗** - 应用 Hamming/Hanning 窗以减少频谱泄漏
3. **FFT** - 快速傅里叶变换,转换到频域
4. **幅度谱** - 从复数 FFT 输出计算幅度
5. **Mel 滤波器组** - 在 Mel 刻度上应用三角滤波器
6. **倒谱变换** - 离散余弦变换（DCT）得到 MFCC
7. **Delta 系数** - 一阶导数（速度特征）
8. **Delta-Delta 系数** - 二阶导数（加速度特征）

### 硬件优化

- **流水线架构** - 重叠的计算阶段以提高吞吐量
- **存储器乒乓缓冲** - 双缓冲方案实现连续处理
- **状态机控制** - 每个处理阶段的模块化 FSM
- **可配置参数** - 帧大小、FFT 点数、滤波器数量可调
- **浮点精度** - IEEE 754 单精度以保证准确性

### 可配置性

该设计支持运行时配置:
- 帧大小和重叠
- FFT 点数
- Mel 滤波器数量
- 倒谱系数数量
- 预加重系数 (α)
- Delta 计算的帧数

## 架构

### 处理流程

```
音频输入 → 预加重 → 加窗 → FFT → 幅度谱
                                    ↓
                               Mel 滤波器组
                                    ↓
                            倒谱变换（DCT）
                                    ↓
                              Delta（一阶）
                                    ↓
                              Delta（二阶）
                                    ↓
                            MFCC 特征向量
```

### 控制层次结构

```
top_ctrl（主控制器）
    ↓
    ├── top_state（顺序阶段激活）
    │
    ├── preem_state_ctrl → pre_emphasis
    ├── window_state_ctrl → window
    ├── fft_core_control → top_fft
    ├── amplitude_state_ctrl → amplitude
    ├── mel_state_ctrl → mel
    ├── ceptrum_state_ctrl → cep
    ├── delta_state_ctrl → delta
    └── delta_2nd_state_ctrl → delta_2nd
```

### 存储器组织

- **系数存储器**: 只读存储器,存储预计算的系数
- **乒乓缓冲器**: 用于级间数据传输的双存储器
- **地址转换**: 灵活访问模式的动态地址映射
- **存储器控制器**: 仲裁读/写操作

### 运算单元

所有算术运算使用 **IEEE 754 32位浮点**:
- **符号位**: 1位
- **指数**: 8位（偏移127）
- **尾数**: 23位（隐含前导1）

特殊运算:
- **对数**: 查找表 + 插值
- **平方根**: Newton-Raphson 迭代（用于幅度）
- **最大/最小**: 成对比较树

## 设计参数

### 典型配置

| 参数 | 默认值 | 描述 |
|------|--------|------|
| `DATA_WIDTH` | 32 | IEEE 754 浮点位宽 |
| `ADDR_WIDTH` | 12 | 存储器地址位宽（4096个位置） |
| `sample_in_frame` | 256-512 | 每帧采样数 |
| `fft_num` | 256-1024 | FFT 大小 |
| `mel_num` | 20-40 | Mel 滤波器数量 |
| `cep_num` | 13 | 倒谱系数数量 |
| `frame_num` | 可变 | 要处理的帧数 |
| `alpha` | 0.97（典型值） | 预加重系数 |

## 快速开始

### 前置要求

- **Verilog 仿真器**: ModelSim、VCS、Icarus Verilog 或类似工具
- **综合工具**: （可选）Synopsys Design Compiler、Cadence Genus
- **波形查看器**: GTKWave、ModelSim wave viewer 或 DVE

### 综合

ASIC 综合示例:
```tcl
# Design Compiler 脚本示例
read_verilog -r top/top.v
read_verilog -r top_ctrl/*.v
read_verilog -r 01_Pre_emphasis/*.v
read_verilog -r 02_Window/*.v
read_verilog -r 03_FFT/*.v
read_verilog -r 04_Amplitude/*.v
read_verilog -r 05_Mel/*.v
read_verilog -r 06_Ceptrum/*.v
read_verilog -r 07_Delta/*.v
read_verilog -r 08_Delta_2nd/*.v
read_verilog -r 00_Common/*.v
read_verilog -r 00_Coefficients/*.v

# 设置顶层模块
current_design top

# 应用约束并综合
source constraints.sdc
compile_ultra
```

### 使用方法

顶层模块 `top.v` 提供主接口:

**输入端口**:
- `clk` - 系统时钟
- `rst_n` - 低电平有效异步复位
- `top_state_en` - 使能信号,启动处理
- `frame_num` - 要处理的帧数
- `sample_in_frame` - 每帧采样数
- 配置参数（alpha、fft_num、mel_num、cep_num 等）
- 用于系数/数据加载的系统存储器接口信号

**输出端口**:
- `finish_flag` - 指示所有帧处理完成
- 用于访问结果的存储器读/写信号
- 每个处理阶段的状态信号

**基本操作**:
1. 将输入音频采样加载到 `mem_input_data`
2. 将系数加载到相应的存储器
3. 用配置参数置位 `top_state_en`
4. 监控 `finish_flag` 等待完成
5. 从结果存储器读取 MFCC 特征

## 开发

### 模块接口约定

所有处理模块遵循此模式:
```verilog
module <stage_name> (
    clk, rst_n,              // 时钟和复位
    <stage>_state_en,        // 来自控制器的使能
    <config_params>,         // 配置输入
    <data_input>,            // 输入数据/系数
    <mem_read_addr>,         // 存储器读地址输出
    <mem_write_addr>,        // 存储器写地址输出
    <data_output>,           // 处理后的数据输出
    write_<stage>_en         // 写使能输出
);
```

### 状态机模式

每个模块包含一个状态控制器:
```verilog
// 状态: IDLE → PROCESSING → DONE
// 为数据通路生成控制信号
// 完成时置位完成标志
```

### 添加自定义模块

1. 遵循接口约定创建模块
2. 添加用于时序控制的状态控制器
3. 在 `top.v` 中实例化
4. 在 `top_ctrl.v` 中添加控制信号
5. 更新 `top_state.v` 状态序列

## 性能特征

### 时序
- **时钟频率**: 通常 50-100 MHz（取决于综合约束）
- **延迟**: ~N 帧处理时间（流水线阶段）
- **吞吐量**: 每个流水线周期 1 帧（初始延迟后）

### 面积
- 主要由以下部分构成:
  - 浮点乘法器（~40%）
  - 存储器块（~30%）
  - 浮点加法器（~20%）
  - 控制逻辑（~10%）

### 功耗
- 动态功耗主要来自运算单元
- 静态功耗来自存储器保持
- 空闲阶段的时钟门控机会

## 相关项目

这是更大的 MFCC 实现套件的一部分:
- **MFCC_AHB_Interface**: 带 AMBA AHB 总线接口的 MFCC（用于 SoC 集成）
- **MFCC_Matlab_Software**: 用于验证的 MATLAB 参考实现
- **出版物**: "A High Performance Dynamic ASIC-Based Audio Signal Feature Extraction (MFCC)"

## 许可证

Apache License 2.0

详见项目根目录中的 [LICENSE](../../../LICENSE) 文件。

## 参考文献

- 上级项目: "An ASIC Based Architecture of Mel Frequency Cepstral Coefficients (MFCC)"
- 资助方: 越南胡志明市科学技术部
- 研究目标: 基于 ANN 的越南语自动语音识别

## 注意事项

- 这是一个**学术研究项目**,展示了音频 DSP 的 ASIC 设计技术
- 该设计使用**浮点运算**以保证准确性（对于超低功耗 ASIC 并不典型）
- 父归档中提供了**测试平台和验证环境**
- 对于系统集成,请参见带总线接口的 **MFCC_AHB_Interface** 版本

---

*详细架构文档请参见 `01_Publications_Docs/` 中的论文演示文稿*
