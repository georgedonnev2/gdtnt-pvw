---
title: 机械臂开发示例-251211
layout: default
parent: AI实验课
nav_order: 20
---

# 机械臂开发示例-251211
{: .no_toc }

在现有机械臂样例程序基础上，增加语音控制，而不是通过界面输入 `grab green cube and move to -80,200`。

期望的实验结果。对连接了机械臂（开发板）的麦克风说话，比如：`抓取绿色方块，并移动到 -80 200`，然后机械臂能按照语音指令，去转去绿色方块，并移动到指定坐标位置。

<details open markdown="block">
  <summary>
    目录
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

## 参考方案

`agent.py` 中，会读取配置文件 `config.json` 中的 `voice` 的取值。如果是  `voice: True`，就会根据录音文件 `Recording.flac` 做相关处理，替代在界面上输入指令。

```python
from react_agent.LLM import RequestLLM
from react_agent.agent import ReactAgent
from react_agent.tools import tools_registry
import time
import json
import os
import voice
import tools

def wait_for_file_update(file_path, last_mtime):
    """ 持续等待文件更新 """
    while True:
        try:
            current_mtime = os.path.getmtime(file_path)
            if current_mtime != last_mtime:
                return current_mtime
        except FileNotFoundError:
            pass
        time.sleep(1)  # 等待 1 秒后再检查

if __name__ == "__main__":
    # 1. 实例化agent
    llm = RequestLLM(base_url="https://api.deepseek.com/v1/", model_name="deepseek-chat")
    agent = ReactAgent(llm)
    # 2. 注册工具
    for name, cls in tools_registry.items():
        agent.register_tool(name, cls)
    # 3. 更新系统prompt
    agent.update_system_message()

    with open("config.json", "r") as config_file:
        config_data = json.load(config_file)

    use_voice = config_data.get("voice", False)
    recording_file_path = "Recording.flac"
    last_mtime = None

    while True:
        time.sleep(5)
        if use_voice:
            if last_mtime is None:
                # 第一次直接识别
                user_input = voice.record_auto()
                last_mtime = os.path.getmtime(recording_file_path)
            else:
                # 等待文件更新
                last_mtime = wait_for_file_update(recording_file_path, last_mtime)
                user_input = voice.record_auto()
        else:
            user_input = input("<USER>:")
        agent.chat(user_input)
```


`agent2.py` 和 `agent.py` 类似。可以读配置文件 `config.json` 中 `voice` 的配置，或者启动时带命令行参数 `sudo python3 agent2.py -v` 。
```python
from react_agent.LLM import RequestLLM
from react_agent.agent import ReactAgent
from react_agent.tools import tools_registry
import time
import json
import os
import voice
import tools
import sys
import argparse

def exit_function():
    """在程序退出时执行的清理函数"""
    print("\n程序即将退出，正在执行清理操作（如机械臂归位）...")
    # 在这里添加机械臂归位的代码
    print("清理完成，程序退出。")

def wait_for_file_update(file_path, last_mtime):
    """ 持续等待文件更新 """
    while True:
        try:
            current_mtime = os.path.getmtime(file_path)
            if current_mtime != last_mtime:
                return current_mtime
        except FileNotFoundError:
            pass
        time.sleep(1)  # 等待 1 秒后再检查

def parse_arguments():
    """解析命令行参数"""
    parser = argparse.ArgumentParser(description='React Agent with command-line input')
    parser.add_argument('user_input', nargs='*', 
                       help='用户输入内容，多个词会被合并成一句话')
    parser.add_argument('--interactive', '-i', action='store_true',
                       help='交互模式，忽略命令行输入，等待用户输入')
    parser.add_argument('--voice', '-v', action='store_true',
                       help='强制使用语音模式，覆盖配置文件设置')
    return parser.parse_args()

if __name__ == "__main__":
    # 解析命令行参数
    args = parse_arguments()
    
    # 1. 实例化agent
    llm = RequestLLM(base_url="https://api.deepseek.com/v1", model_name="deepseek-chat")
    agent = ReactAgent(llm)
    
    # 2. 注册工具
    for name, cls in tools_registry.items():
        agent.register_tool(name, cls)
    
    # 3. 更新系统prompt
    agent.update_system_message()
    
    with open("config.json", "r") as config_file:
        config_data = json.load(config_file)
    
    # 语音设置：命令行参数优先，否则使用配置文件
    use_voice = args.voice or config_data.get("voice", False)
    recording_file_path = "Recording.flac"
    last_mtime = None
    
    try:
        # 如果提供了命令行输入且不是交互模式
        if args.user_input and not args.interactive:
            # 将命令行参数合并成一句话
            user_input = ' '.join(args.user_input)
            print(f"<USER>: {user_input}")
            agent.chat(user_input)
        else:
            # 交互模式或没有提供命令行输入时的原有逻辑
            print("进入交互模式...")
            while True:
                time.sleep(5)
                if use_voice:
                    if last_mtime is None:
                        # 第一次直接识别
                        user_input = voice.record_auto()
                        last_mtime = os.path.getmtime(recording_file_path)
                    else:
                        # 等待文件更新
                        last_mtime = wait_for_file_update(recording_file_path, last_mtime)
                        user_input = voice.record_auto()
                else:
                    user_input = input("<USER>:")
                agent.chat(user_input)
    finally:
        exit_function()
```

因此，参考方案如下：
 
- （和下面的方法，二选一即可）[方法1] 在开发板上启动一个 `终端(terminal)`，执行 `sudo python3 agent2.py -v`。（命令行参数 `-v`  是告知开发板处理语音文件的意思。）
- （和上面的方法，二选一即可）[方法2] 或者，在开发板上启动一个 `终端(terminal)`，修改 `config.json` 中 `voice` 配置为 `voice: true`，然后执行 `sudo python3 agent.py`。

- 在开发板上，再启动一个 `终端(terminal)`，运行 `q5.py`（待编写），将通过和开发板连接的麦克风说的话，生成 `Recording.flac` 语音文件，供机械臂识别语音并执行相关动作。

<hr>

## 录音（和播放）

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
