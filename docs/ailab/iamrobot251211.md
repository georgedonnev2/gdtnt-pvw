---
title: 机械臂开发示例-251211
layout: default
parent: AI实验课
nav_order: 20
---

# 机械臂开发示例-251211
{: .no_toc }

<details open markdown="block">
  <summary>
    目录
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>


## 录音和播放

```bash
sudo apt-get install libportaudio2 portaudio19-dev python3-dev
pip3 install sounddevice numpy scipy
```

```bash
# 列出录音设备
arecord -l
# 列出播放设备
aplay -l
```

可以和自己偏好的大模型（比如 DeepSeek 等）对话，获取如何用 USB 麦克风/喇叭 将语音录制为文件。

本次实验使用 得普声 Q5+ USB 麦克风/喇叭。

以下是样例程序，仅供参考。

```python
import sounddevice as sd
import numpy as np
import soundfile as sf # 导入soundfile库
# 关键：从scipy.io.wavfile中导入write函数
from scipy.io.wavfile import write
import subprocess
import os

# 使用PulseAudio作为音频后端
input_device_index = 'pulse'  # 或使用索引 32
output_device_index = 'pulse' # 播放也用pulse

# 录音参数（根据pulse设备信息，它支持多种采样率）
fs = 44100  # 与你的USB设备默认采样率一致
duration = 15
channels = 1  # 单声道，兼容性最佳

print("开始录音...")
recording = sd.rec(int(duration * fs),
                   samplerate=fs,
                   channels=channels,
                   dtype='int16', # soundfile会自动处理int16到float32的转换
                   device=input_device_index)
sd.wait()
print("录音结束。")

# 在 sd.wait() 之后，sf.write() 之前，添加：
print("录制数据的类型:", recording.dtype)
print("录制数据的最小值:", recording.min())
print("录制数据的最大值:", recording.max())
print("录制数据的形状:", recording.shape)


# 保存为WAV文件
file_wav = "output.wav" 
write(file_wav, fs, recording)
print(f"音频文件已保存为: {file_wav}")

# print(f"保存前的形状: {recording.shape}") # 应该是 (220500, 1)
# # 关键修复：移除多余的维度
# recording = recording.squeeze() # 现在形状变为 (220500,)
# print(f"保存前的形状: {recording.shape}")

# # 保存为FLAC文件
# file_flac = "output.flac" # 文件扩展名改为 .flac
# sf.write(file_flac, recording, fs, subtype='PCM_16') # 指定16位PCM格式
# print(f"音频文件已保存为: {file_flac}")

# 2. 使用ffmpeg将WAV转换为FLAC
file_flac = "Recording.flac"
# 静默模式运行，-y表示覆盖输出文件
subprocess.run(['ffmpeg', '-i', file_wav, '-c:a', 'flac', '-y', file_flac], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
print(f"FLAC文件已保存: {file_flac}")

# 尝试播放刚录制的音频
print("正在播放录音...")
sd.play(recording, fs, device=output_device_index)
sd.wait()
print("播放结束。")
```
