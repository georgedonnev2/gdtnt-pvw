---
title: AI教具使用简要说明
layout: default
parent: AI Lab
nav_order: 2
---

# AI教具使用简要说明
{: .no_toc }

<details open markdown="block">
  <summary>
    目录
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

## 视觉实验箱

登录后，执行 `cd /elephant-ai` 切换到目录 `/elephant-ai`：
```bash
# cd 
jetson@jetson-Yahboom:~$ cd /elephant-ai

# pwd 确认是在 /elephant-ai 目录
jetson@jetson-Yahboom:/elephant-ai$ pwd
/elephant-ai
```

<br>

执行 `sudo python3 agent.py`，启动视觉实验箱样例程序： 
```bash
jetson@jetson-Yahboom:/elephant-ai$ sudo python3 agent.py
WARNING: Carrier board is not from a Jetson Developer Kit.
WARNNIG: Jetson.GPIO library has not been verified with this carrier board,
WARNING: and in fact is unlikely to work correctly.
<USER>:
```

<br>

放置一些积木，在桌上图纸的带 + 号的方框内。积木上的蓝色、红色、绿色等朝上。

在 `<USER>:` 提示符后面，输入 `grab green cube and move -80,200`

```bash
<USER>:grab green cube and move -80,200
<LLM>:我需要先抓取绿色方块，然后将其移动到指定位置。根据指令，我需要先使用grab_object，然后使用move_to。

✿FUNCTION✿: grab_object
✿ARGS✿: {"object_name": "green cube"}

✿FUNCTION✿: move_to
✿ARGS✿: {"target_coord": [-80,200], "target_height": 110}
functions_and_args: [('grab_object', {'object_name': 'green cube'}), ('move_to', {'target_coord': [-80, 200], 'target_height': 110})]
#################### <函数执行> ####################
Image saved as captured_image.jpg
[{'x1': 0, 'x2': 179, 'y1': 332, 'y2': 584}]
像素坐标 (57.28, 219.84) 对应的机械臂坐标为: [212.8  68.1]
#################### <函数执行> #################### 

#################### <函数执行> ####################
*************
[-80, 200]
Objects arranged successfully
#################### <函数执行> #################### 

<USER>:
```

{: .note}
move -80,200。分别是 x、y、z坐标。z可以省略，默认是110。x和y是什么方向，可参考图纸上的标识。z的正方向是桌面向上。

同时按下 `ctrl`和`c`，可退出。
```bash
<USER>:^CTraceback (most recent call last):
  File "agent.py", line 50, in <module>
    user_input = input("<USER>:")
KeyboardInterrupt

jetson@jetson-Yahboom:/elephant-ai$ 
```

### 抓不准该如何调整

- 机械臂底座，是否在图纸的矩形中。

- 如果还抓不准，可以调整 `/elephant-ai/config.json` 中的 x、y、z 的偏移量。
```json
{
    "points_pixel": [
        [320,220],
        [590,430],
        [72,31],
        [590,26]
    ],
    "points_arm": [
        [210,-30],
        [140, -110],
        [280,70],
        [280,-110]
    ],
    "x": 10,
    "y": -7,
    "z": -5,
    "voice":false,
    "threshold": 110
}
```

如何修改 `config.json`：
- 执行 `sudo vim config.json` 打开文件
- 删除。光标移动到要删除字符的位置，按 `Esc` 键，然后再按 `x`，可删除字符。
- 插入。光标移动到要插入字符的位置，按 `Esc` 键，然后再按 `i`，然后输入新的字符。
- 保存退出。按 `Esc` 键，然后输入 `:wq`，再按`回车`键。

执行 `cat config.json`，确认确实修改了。
```bash
jetson@jetson-Yahboom:/elephant-ai$ cat config.json
```

<hr>

## 华为昇腾开发板

**0、账号密码**
- 账号：`HwHiAiUser` / 密码：`Mind@123`
- 账号：`root` / 密码：`Mind@123`

**1、登录后，执行 `su - root` 切换到 `root` 用户**

```bash
(base) HwHiAiUser@davinci-mini:~$ su - root
Password: 
(base) root@davinci-mini:~# 
```

{. :note}
输密码时，屏幕不会显示。输完密码后按回车即可。
{. :note}
切换到 root 用户，是为了体验摄像头识别物体。

**2、执行 `cd` 切换到相关目录中**

```bash
(base) root@davinci-mini:~# cd /home/HwHiAiUser/samples/notebooks
(base) root@davinci-mini:/home/HwHiAiUser/samples/notebooks# 
```

**3、执行 `jupyter lab --allow-root` 启动后台**

```bash
(base) root@davinci-mini:/home/HwHiAiUser/samples/notebooks# jupyter lab --allow-root
[I 2025-12-08 13:55:37.302 ServerApp] Package jupyterlab took 0.0001s to import
[I 2025-12-08 13:55:37.418 ServerApp] Package jupyter_lsp took 0.1141s to import
[W 2025-12-08 13:55:37.418 ServerApp] A `_jupyter_server_extension_points` function was not found in jupyter_lsp. Instead, a `_jupyter_server_extension_paths` function was found and will be used for now. This function name will be deprecated in future releases of Jupyter Server.
[I 2025-12-08 13:55:37.470 ServerApp] Package jupyter_server_terminals took 0.0502s to import
[I 2025-12-08 13:55:37.471 ServerApp] Package notebook_shim took 0.0001s to import
[W 2025-12-08 13:55:37.472 ServerApp] A `_jupyter_server_extension_points` function was not found in notebook_shim. Instead, a `_jupyter_server_extension_paths` function was found and will be used for now. This function name will be deprecated in future releases of Jupyter Server.
[I 2025-12-08 13:55:37.477 ServerApp] jupyter_lsp | extension was successfully linked.
[I 2025-12-08 13:55:37.494 ServerApp] jupyter_server_terminals | extension was successfully linked.
[I 2025-12-08 13:55:37.514 ServerApp] jupyterlab | extension was successfully linked.
[I 2025-12-08 13:55:39.552 ServerApp] notebook_shim | extension was successfully linked.
[I 2025-12-08 13:55:39.683 ServerApp] notebook_shim | extension was successfully loaded.
[I 2025-12-08 13:55:39.693 ServerApp] jupyter_lsp | extension was successfully loaded.
[I 2025-12-08 13:55:39.697 ServerApp] jupyter_server_terminals | extension was successfully loaded.
[I 2025-12-08 13:55:39.699 LabApp] JupyterLab extension loaded from /usr/local/miniconda3/lib/python3.9/site-packages/jupyterlab
[I 2025-12-08 13:55:39.699 LabApp] JupyterLab application directory is /usr/local/miniconda3/share/jupyter/lab
[I 2025-12-08 13:55:39.707 LabApp] Extension Manager is 'pypi'.
[I 2025-12-08 13:55:39.719 ServerApp] jupyterlab | extension was successfully loaded.
[I 2025-12-08 13:55:39.720 ServerApp] Serving notebooks from local directory: /home/HwHiAiUser/samples/notebooks
[I 2025-12-08 13:55:39.721 ServerApp] Jupyter Server 2.5.0 is running at:
[I 2025-12-08 13:55:39.721 ServerApp] http://localhost:8888/lab?token=f20c1335ecc7d52f63a372cbed8a12fccbd336bf590e1a96
[I 2025-12-08 13:55:39.721 ServerApp]     http://127.0.0.1:8888/lab?token=f20c1335ecc7d52f63a372cbed8a12fccbd336bf590e1a96
[I 2025-12-08 13:55:39.721 ServerApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
[W 2025-12-08 13:55:39.750 ServerApp] No web browser found: Error('could not locate runnable browser').
[C 2025-12-08 13:55:39.750 ServerApp] 
    
    To access the server, open this file in a browser:
        file:///root/.local/share/jupyter/runtime/jpserver-9111-open.html
    Or copy and paste one of these URLs:
        http://localhost:8888/lab?token=f20c1335ecc7d52f63a372cbed8a12fccbd336bf590e1a96
        http://127.0.0.1:8888/lab?token=f20c1335ecc7d52f63a372cbed8a12fccbd336bf590e1a96
```

**后续步骤，请参考如下链接**
- [登录 jupyter lab](https://www.hiascend.com/document/detail/zh/Atlas200IDKA2DeveloperKit/23.0.RC2/Getting%20Started%20with%20Application%20Development/paqe/paqe_0002.html)
- [目标检测样例](https://www.hiascend.com/document/detail/zh/Atlas200IDKA2DeveloperKit/23.0.RC2/Getting%20Started%20with%20Application%20Development/paqe/paqe_0003.html)

<hr>

## 语音实验箱

访问：[https://172.18.144.18/plugin/frontend/#/appview/AJ62kgyj?i=1](https://172.18.144.18/plugin/frontend/#/appview/AJ62kgyj?i=1)

也可在 Pad 上部署一系列软件后，在 Pad 启动语音服务。

在 Pad 部署，可参考手册：[语音对话实验指导手册](./ailabkit.assets/语音对话实验指导手册.pdf)


## 自然语言实验箱

相关手册：[自然语言处理实验箱V1](./ailabkit.assets/自然语言处理实验箱V1.pdf)