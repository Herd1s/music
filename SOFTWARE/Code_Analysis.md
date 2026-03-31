# STM32 音频频谱分析工程代码详解

## 1. 工程概览
**项目目标**: 实现音频信号的采集、频域转换（FFT）及 OLED 频谱显示。
**核心硬件**: STM32G431（MCU）, CS4398（DAC/Codec）, OLED 屏幕（I2C/SPI）。
**核心库**: CMSIS-DSP（部分使用）, U8G2（图形库）, HAL库。

## 2. 核心模块与代码关联

### 2.1 主循环 (Core/Src/main.c)
程序的“心脏”。
```c
while (1)
{
  if(g_buffer_ready == 1) // 1. 等待 DMA 数据采集完成
  {
      if (g_display_mode == 0)
      {
          Audio_FFT(); // 2. 模式0：进入FFT处理流程
      }
      else if (g_display_mode == 1)
      {
          Audio_Wave_Display(); // 3. 模式1：直接显示波形
      }
      g_buffer_ready = 0; // 4. 清除标志位，等待下一帧数据
  }
}
```

### 2.2 音频处理 (User/audio.c)
这是最复杂的逻辑部分。

#### 数据流向
1.  **采集**: `HAL_I2S_Transmit_DMA` 触发 I2S 传输，数据填入 `audio_buffer`。
2.  **准备**: `Audio_FFT()` 函数被调用。
3.  **转换**: 将 16位整数音频数据 (`int16_t`) 转换为 浮点数 (`float`) 存入 `FFT_input`。
4.  **加窗**: `arm_mult_f32(..., hanning_window, ...)`。
    *   *解释*: 就像拍照要聚焦一样，加汉宁窗（Hanning Window）是为了让截取的一段音频首尾平滑过渡，减少频谱显示的“拖尾”和噪声。
5.  **计算**: 调用 `FFT()` 进行运算。
6.  **取模**: 调用 `FFT_Modulo()` 计算复数的模（也就是声音的“响度”）。

#### 显示算法 (`Audio_FFT_Display`)
```c
// 遍历屏幕的 128 个像素列
for (uint16_t x = 0; x < 128; x++)
{
    // 计算分贝值：20 * log10(幅值)
    // +1 是为了防止 log10(0) 数学错误
    if ((uint16_t) (20 * log10f(FFT_output[x] + 1)) > 63) {
        column_height = 63; // 限幅，不能超过屏幕高度
    } else {
        column_height = (uint8_t) (20 * log10f(FFT_output[x] + 1));
    }

    // 画柱状图
    // DrawHVLine(x, 63, column_height, ...) 
    // 这里的逻辑推测是：在 X 位置，从 Y=63（屏幕底部）向上画 column_height 长度的线。
    u8g2_DrawHVLine(&u8g2, x, 63, column_height, 3);
    
    // 处理掉落点 (Peak Hold效果)
    // falling_point[x] 记录了点当前的 Y 坐标
    // 0 是屏幕顶部，63 是屏幕底部
    
    // 如果新的柱子高度比点还高（注意坐标系，高度越高，Y值越小，所以是 63 - height）
    if (falling_point[x] > 63 - column_height) 
    {
        falling_point[x] = 63 - column_height; // 把点顶上去
    }
    else
    {
        falling_point[x] += 0.7f; // 点因重力下落（Y值增加）
    }
    u8g2_DrawPixel(&u8g2, x, (uint8_t) falling_point[x]);
}
```

### 2.3 自定义 FFT (User/FFT.c)
并没有完全使用 ARM 官方库的 `arm_cfft_f32`，而是手写了一个 FFT 实现。
*   **核心思想**: 蝶形运算。
*   **Step 函数 / Rebit 函数**: 这是 FFT 算法中的“倒位序”操作。因为 FFT 算法需要把输入数据的顺序打乱（二进制位反转），才能快速计算。

### 2.4 屏幕驱动 (User/oled.c)
封装了 `U8G2` 库。U8G2 是单片机界非常通用的黑白屏驱动库。
*   `u8g2_SendBuffer(&u8g2)`: 这一步非常关键。U8G2 通常使用“显存缓冲”模式。所有的画点、画线操作只是在内存里修改数据，只有执行了这句话，才会把整个画面一次性发送给 OLED 屏幕显示出来。

## 3. 高低频逻辑总结
*   **数组索引 = 频率**: `FFT_output` 数组的下标从小到大，对应频率从低到高。
*   **Visualizer (屏幕显示方向)**: 
    *   **左侧 (Left)** (下标 0-40)：**低音区**（低沉的声音，如鼓点、贝斯）。
    *   **中间 (Center)** (下标 40-80)：**中音区**（人声、吉他、钢琴中段）。
    *   **右侧 (Right)** (下标 80-127)：**高音区**（尖锐的声音，如镲片、三角铁）。
    *   **结论**: 这是一个符合直觉的“左低右高”设计，就像钢琴键盘一样，左手是低音，右手是高音。

## 4. 频率范围详解 (补充)
基于代码配置推算：
*   **采样率 (Fs)**: 48 kHz (见 `i2s.c` -> `I2S_AUDIOFREQ_48K`)。
    *   这意味着系统每秒采集/播放 48000 个音频点。
*   **FFT 点数 (N)**: 512 (见 `FFT.h`)。
*   **频率分辨率**: $ \Delta f = \frac{Fs}{N} = \frac{48000}{512} \approx 93.75 \text{ Hz} $。
    *   每个柱子代表约 93.75 Hz 的频段宽度。
*   **总显示带宽**:
    *   代码可见 `u8g2_DrawHVLine` 循环 `x` 从 0 到 127。
    *   显示范围 = $ 128 \times 93.75 \text{ Hz} = 12000 \text{ Hz} $ (12 kHz)。
    *   **结论**: 屏幕上覆盖了 0 Hz 到 12 kHz 的声音范围。这涵盖了人耳能听到的绝大部分音乐能量（人耳极限 20Hz-20kHz，但音乐高频能量通常较低，12k 以上往往只是泛音和空气感）。


