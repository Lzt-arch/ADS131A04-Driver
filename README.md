# ADS131A04 驱动

> from lzt + ChatGPT 5.6 sol  
> 初版撰写：2026.7.20

---

## 移植说明

如果移植，需要移植 `ADS131A0x` 和 `ADS_FFT` 共 **2个文件夹、8个文件**：

| 文件夹 | 说明 |
|--------|------|
| `ADS131A0x` | 采样相关的程序 |
| `ADS_FFT` | 后续数据处理的程序 |

---

## 基本调用流程

当前例程实现的是 **4通道同步采样**，处理1-3通道数据，计算1-2通道的相位差。

### 平台信息

- **MCU**：STM32H743IIT6
- **SPI**：硬件 SPI2
- **时钟**：TIM4_CH1 输出方波提供 16MHz 时钟
- 配置参考对应 Word 文档

---

## 应用层实现

### 采样点数
由 `.h` 文件中宏设定。

### 采样频率
计算公式：

$$
f_{sample} = \frac{f_{clk}}{ICLK\_DIV \times MODCLK\_DIV \times OSR}
$$

通过函数 `ADS_Set_Sample_Rate` 设定。

### 通道使能
通过 **4位二进制数** 选择，`1` 为启用，`0` 为禁用，通过函数 `ADS_Set_CH_Enable` 设定。

---

## 使用流程

```
启用时钟（软件启用） → 设置采样频率 → 使能采样通道 → 初始化 → 开启采样
```

> 在上一次数据处理完成后循环调用即为**连续采样模式**。

---

## 注意事项

### 1. SPI 传输速率

SPI 传输速率是影响模块正常传输数据的关键。当 4 路 ADC 全部开启时，每帧数据量：

```
STATUS + ADC1 + ADC2 + ADC3 + ADC4 = 5 × 32 = 160 bit
```

SPI 传输速率需要满足：

$$
SPI\_Rate > 160 \times f_{sample}
$$

**示例**：采样频率 83.33kHz 时：

$$
160 \times 83.33\text{k} = 13.33\text{ MHz} < \text{SPI 24MHz}
$$

但由于 ADS131A04 的最大 SPI 速率只有 **25MHz**，这个速率是很极限的，实际使用时建议在 **12-20MHz** 之间。

### 2. Sinc3 滤波器幅度衰减

芯片 ADC 前端有固定的 **sinc3 滤波器**，当测量频率升高时会有对应的幅度衰减。实测除幅度不准外，其它数据仍然准确。

### 3. 采样率与带宽余量

| 配置 | 传输时间 | 采样周期 | 余量 | 状态 |
|------|---------|---------|------|------|
| 83kSPS（当前） | 6.67μs | 12μs | ~44% | ✅ 非常充裕 |
| OSR=32（125kSPS 最快） | 6.67μs | 8μs | ~16% | ⚠️ 实测会因 SPI 带宽不足而漏帧 |

---

## 具体实现

### 初始化

```c
/* ADS输入CLK由TIM4_CH1(PD12)输出；在这里传入期望频率。 */
ADS_Start_Input_Clock(ADS_DEFAULT_INPUT_CLOCK_HZ);

HAL_Delay(10U);

/* 参数依次为CLK1分频、CLK2调制时钟分频、OSR。 */
ADS_Set_Sample_Rate(ADS_DEFAULT_ICLK_DIVIDER, ADS_DEFAULT_MODCLK_DIVIDER, ADS_DEFAULT_OSR);

/* 0x0F(1111)：同步启用CH1~CH4。 */
ADS_Set_CH_Enable(0x0FU);

ADS_Init();

ADS_Start();
```

### 主循环处理

```c
if (ADS_Samples_Ready() != 0U)              // 采样结束
{
    if (ADS_Process_Only() != ADS_OK)       // 读取结束
    {
        printf("ADS data processing failed\r\n");
    }
    else
    {
        /* 0x07：处理CH1~CH3；0x03：计算CH2相对CH1的相位差。 */
        if (deal_FFT(0x07U, 0x03U) == ADS_FFT_OK)
        {
            printf("CH1: Freq=%.3f Hz Phase=%.3f deg Vpp=%.3f V THD=%.3f%%\r\n",
                (double)ADC_Res1.freq,
                (double)ADC_Res1.phaseDeg,
                (double)ADC_Res1.adcVpp,
                (double)(ADC_Res1.thd * 100.0f));
            printf("CH2: Freq=%.3f Hz Phase=%.3f deg Vpp=%.3f V THD=%.3f%%\r\n",
                (double)ADC_Res2.freq,
                (double)ADC_Res2.phaseDeg,
                (double)ADC_Res2.adcVpp,
                (double)(ADC_Res2.thd * 100.0f));
            printf("CH3: Freq=%.3f Hz Phase=%.3f deg Vpp=%.3f V THD=%.3f%%\r\n",
                (double)ADC_Res3.freq,
                (double)ADC_Res3.phaseDeg,
                (double)ADC_Res3.adcVpp,
                (double)(ADC_Res3.thd * 100.0f));
            printf("Phase CH2-CH1=%.3f deg\r\n",
                (double)ADC_Res1.phaseDiffDeg1_2);
        }
        else
        {
            printf("ADS FFT processing failed\r\n");
        }
    }
}
```
