# 机械臂开发示例-251120

## 0、相关说明

### 0.1 组装机械臂

除了以下几个注意点，和 NLP 实验箱类似：

- 机械臂上的绑着的摄像头的 USB 线。要插到开发板的 USB 口。
- 机械臂上的 USB 线（位于机械臂和开发板挨着的地方）。也要插到开发板的 USB 口。
- 屏幕的电源线。到实验室后面柜子中领取 USB 插头，电源线插到 USB 插头上，然后 USB 插头插到桌子下的插座上。（和 NLP 实验箱略有区别。NLP实验箱的屏幕的电源线，是插到开发板的 USB 口上。） 
- 机械臂底座。放置在图纸的长方形框内。

### 0.2 账号密码，IP地址

1. 账号密码

- jetson / yahboom
- root / yahboom

2. IP地址

执行 `ifconfig | grep 172`，屏幕输出 `172.18.xx.xx` 就是本开发板的 IP 地址。

如果有多个 172 开头的地址，请检查并关闭开发板的 wifi。wifi 打开并连接，还插了网线，会导致无法被 `ping` 通等故障。

### 0.3 尝试样例是否运行正常

1. 样例的目录

开机后，找到样例所在的目录 `elephant-ai`。可能在当前目录下，即 `/home/jetson/elephant-ai`，也可能在根目录下，即 `/elephant-ai`。

几个相关命令：
- `pwd`。确定当前目录是哪里。
- `ls -l`。列出当前目录下的子目录和文件。
- `cd`。切换到用户的 HOME 目录。每个用户都有 HOME 目录，默认是 `/home/用户名`。比如对于 `jetson` 用户，默认的 HOME 目录是 `/home/jetson`。
- `cd 目标目录名`。切换到目标目录。比如切换到根目录，可执行 `cd /`。

2. 在样例的目录中，修改 `agent.py`

将 `agent.py` 中的 `RequestLLM()` 的 `base_url` 和 `model_name`，修改为如下值。
```python  
if __name__ == "__main__":
    # 1. 实例化agent
    llm = RequestLLM(base_url="https://api.deepseek.com/v1/", model_name="deepseek-chat")
```

3. 运行样例

在样例目录中，运行 `sudo python3 agent.py`。稍等几秒后，命令行界面出现提示符 `<USER>:`。

然后输入 `grap blue cube and move to -80, 200, 110` ，按回车。随后观察机械臂是否能抓取蓝色积木，并放到右边区域。

4. 退出样例

同时按下 `ctrl` 和 `c`，可退出样例。

### 0.4 笔记本和开发板连接

1. 可在笔记本上通过 vscode 打开开发板上的代码

可在开发板上安装 vscode，然后在开发板上用 vscode 编辑代码。

也可以在笔记本上的 vscode，直接打开开发板上的代码。如何配置操作，可参考 [vscode通过ssh连接服务器实现免密登录+删除（吐血总结）](https://blog.csdn.net/Oxford1151/article/details/137228119)。

2. Windows 笔记本 SSH 登录开发板

如需要在 Windows 笔记本通过 SSH 登录开发板，可在 `cmd` 或 `powershell` 窗口中，执行 `ssh 用户名@开发板的IP地址` 登录开发板。比如，`ssh jetson@172.18.139.108`。

还可以通过 [MobaXterm](./nlp251113.md#04-本地笔记本装-mobaxterm) 登录。

<!--  -->
## 基本原理

机械臂本质原理上：通过大模型返回指定格式的、编排好的工具使用顺序，调用对应工具。

## 工具实现

所有的工具实现均在 `tools.py` 内，通过装饰器 `register_tool` 注册到工具列表；需要继承 `BaseTool` 类完成工具功能描述 `description` 和参数需求 `parameters`；通过实现 `call` 函数完成具体动作；`MyCobot` 是与机械臂进行通讯的库，通过 `send_coords`、`send_angles` 等库函数移动机械臂。

## 参考文档

机械臂的具体操作文档请参考（基于python开发使用）：[MyCobot 280 JN Python开发指南](https://docs.elephantrobotics.com/docs/mycobot_280_jn_cn/3-FunctionsAndApplications/6.developmentGuide/python/)

## 现有基础工具

- `move_to` - 移动工具
- `grab_object` - 抓取工具
- `show_object` - 展示工具

## 开发新动作示例

接下来以开发新的动作为例：将指定垃圾（方块上的图案）放入垃圾桶（放置区域内的垃圾桶照片示例）。

### 动作组成

新动作的基本组成可以分为：
1. 抓取区域拍照
2. 抓取
3. 放置区域拍照
4. 移动放置

### 开发步骤

#### 步骤1: 独立出拍照工具

此工具需要要求大模型必须在抓取动作之前调用；放置动作之前根据情况调用。

**关键设计**：通过 `enum` 限制 `area_description` 参数只能是 `grabbing_area`（抓取区域）或 `placement_area`（放置区域）两种选择，这样可以：
- 确保大模型只能传入正确的参数值
- 根据不同区域切换到对应的拍照位置和参数
- 避免参数错误导致的异常行为

```python
@register_tool('take_photo')
class TakePhoto(BaseTool):
    description = 'This function takes a photo of the scene to capture the current state. Must be called before grab_object to detect objects in the grabbing area. Can be called before move_to if you need to analyze the placement area.'
    parameters = [
        {
            'name': 'area_description',
            'type': 'string',
            'enum': ['grabbing_area', 'placement_area'],
            'description': 'Area to photograph. Must be either "grabbing_area" (for detecting objects to grab) or "placement_area" (for detecting target placement location).',
            'required': True
        }
    ]

    def call(self, area_description, **kwargs):
        # 验证参数
        if area_description not in ['grabbing_area', 'placement_area']:
            return f"Invalid area_description: {area_description}. Must be 'grabbing_area' or 'placement_area'"
        
        # 根据不同区域移动到对应的拍照位置
        if area_description == 'grabbing_area':
            # 移动到抓取区域的拍照位置
            # ！！！不是真实的数字，只是样例！！！
            mc.send_angles([-65.39, 14.76, -65.83, -42.36, 2.72, 70.3], 10)
        else:  # placement_area
            # 移动到放置区域的拍照位置，需要确认具体放置区域坐标
            # ！！！不是真实的数字，只是样例！！！
            mc.send_angles([-65.39, 14.76, -65.83, -42.36, 2.72, 70.3], 10)
        time.sleep(3)

        # 打开摄像头并拍照
        cap = cv2.VideoCapture(0)
        ret, frame = cap.read()
        time.sleep(2)

        if not ret:
            print("Failed to capture image")
            return "Failed to capture image"

        # 伽马校正
        gamma_corrected_frame = adjust_gamma(frame, gamma=2.1)

        # 保存图片
        image_path = "captured_image.jpg"
        cv2.imwrite(image_path, gamma_corrected_frame)
        print(f"Image saved as {image_path} for {area_description}")
        
        cap.release()
        cv2.destroyAllWindows()
        
        return f"Photo taken successfully for {area_description}"
```

#### 步骤2: 修改 grab_object 工具

将拍照工具从 `grab_object` 工具内删除。`grab_object` 工具现在将直接读取拍照获得的图片并分析指定物体的坐标，然后抓取。

```python
@register_tool('grab_object')
class GrabObject(BaseTool):
    description = 'This function detects the position of a specified object from the previously captured image and performs a grabbing action. Must call take_photo before using this function.'
    parameters = [
        {
            'name': 'object_name',
            'type': 'string',
            'description': 'The name of the object to be detected and grabbed.',
            'required': True
        }
    ]

    def call(self, object_name, **kwargs):
        
        # 初始化机械臂
        init.BotInit(mc)

        # 读取已拍摄的图片（不再拍照）
        width, height = Image.open('captured_image.jpg').size
        
        # 调用视觉API分析物体位置
        positions = api.QwenVLRequest("一个" + object_name, "captured_image.jpg").get("coordinates", [])
        print(f"检测到的位置: {positions}")

        # 读取配置偏移量
        with open("config.json", "r") as config_file:
            config_data = json.load(config_file)

        x_offset = config_data.get("x", 0)
        y_offset = config_data.get("y", 0)
        z_offset = config_data.get("z", 0)

        # 只处理第一个检测到的物体
        if positions:
            position = positions[0]
            # 计算中心点坐标
            center_x = (position['x1'] + position['x2']) / 2
            center_y = (position['y1'] + position['y2']) / 2
            target_coord = (center_x / 1000 * width, center_y / 1000 * height)
            
            # 像素坐标转换为机械臂坐标
            robot_coord = eyeonhand.pixel_to_arm(target_coord)
            print(f"像素坐标 {target_coord} 对应的机械臂坐标为: {robot_coord}")
            
            # 应用偏移量
            robot_coord[0] = robot_coord[0] + x_offset
            robot_coord[1] = robot_coord[1] + y_offset
            if robot_coord[0] > 210:
                robot_coord[0] = robot_coord[0] - 5
            z = 120 + z_offset

            # 执行抓取动作
            init.open_gripper()
            mc.send_coords([robot_coord[0], robot_coord[1], 200, -173, 0, -45], 40)
            time.sleep(5)
            mc.send_coords([robot_coord[0], robot_coord[1], z, -173, 0, -45], 40)
            time.sleep(5)
            init.close_gripper()

            mc.send_coords([robot_coord[0], robot_coord[1], 200, -173, 0, -45], 20)
            time.sleep(3)
            mc.send_angles([0, 0, 0, 0, 0, -45], 40)
            
            return f"Successfully grabbed {object_name} at {robot_coord}"
        else:
            return f"No {object_name} found in the image"
```

#### 步骤3: 修改 move_to 工具

修改 `move_to` 工具接受的参数。现在移动的目标坐标有两种传入方式，让大模型自己决定如何选择：
- 如果直接传入合理的坐标值，则不需要分析图片
- 如果传入物体名称字符串，则此时需要在 `move_to` 工具之前调用拍照工具拍摄放置区照片，然后 `move_to` 工具自动调用多模态大模型找到指定物体的坐标

```python
@register_tool('move_to')
class MoveTo(BaseTool):
    description = 'This function moves the grabbed object to a target location. The target can be specified either as coordinates or as an object name (e.g., "trash bin"). If using an object name, must call take_photo before this function to capture the placement area.'
    parameters = [
        {
            'name': 'target',
            'type': 'string or list',
            'example': '[100, 200] or "trash bin"',
            'description': 'Target location. Can be either: 1) A coordinate list [x, y] where x ranges from -70 to 140, y ranges from 150 to 280, or 2) An object name string to detect from the captured image.',
            'required': True
        },
        {
            'name': 'target_height',
            'type': 'int',
            'example': '110',
            'description': 'Height at which to release the object. Typically 110. For stacking, increase by 20 for each layer (110, 130, 150, etc.).',
            'required': True
        }
    ]

    def call(self, target, target_height, **kwargs):
        
        # 判断 target 是坐标还是物体名称
        if isinstance(target, list):
            # 直接使用提供的坐标
            target_robot_coord = target
            print(f"使用直接坐标: {target_robot_coord}")

        elif isinstance(target, str):
            # 从图片中识别目标物体的位置
            width, height = Image.open("captured_image.jpg").size
            
            # 调用视觉API分析目标物体位置
            positions = api.QwenVLRequest("一个" + target, "captured_image.jpg").get("coordinates", [])
            
            if not positions:
                return f"Cannot find {target} in the captured image"
            
            # 计算目标物体的中心坐标
            position = positions[0]
            center_x = (position['x1'] + position['x2']) / 2
            center_y = (position['y1'] + position['y2']) / 2
            target_coord = (center_x / 1000 * width, center_y / 1000 * height)
            
            # 转换为机械臂坐标
            target_robot_coord = eyeonhand.pixel_to_arm(target_coord)
            print(f"识别到 {target} 的坐标: {target_robot_coord}")
        else:
            return "Invalid target parameter type"

        print(f"目标坐标: {target_robot_coord}")

        # 执行移动和放置动作
        time.sleep(3)

        # 移动到目标位置上方
        mc.send_coords([target_robot_coord[0], target_robot_coord[1], 180, -175, 0, -45], 40)
        time.sleep(3)

        # 下降到指定高度
        mc.send_coords([target_robot_coord[0], target_robot_coord[1], target_height, -175, 0, -45], 40)
        time.sleep(3)
        
        # 释放物体
        init.open_gripper()
        time.sleep(1)

        # 抬起并返回初始位置
        mc.send_coords([target_robot_coord[0], target_robot_coord[1], 200, -175, 0, -45], 40)
        time.sleep(3)
        mc.send_angles([0, 0, 0, 0, 0, -45], 40)
        time.sleep(1)

        return f"Object moved successfully to {target_robot_coord}"
```

#### 步骤4：测试
多次运行相关动作，验证大模型是否可以完成动作编排。若编排有问题，则可以调整react_agent/agent.py里的总提示词进行调整。

### 完整的工作流程示例

将指定垃圾放入垃圾桶的完整调用流程：

```python
# 大模型会生成如下的工具调用序列：

# 1. 拍摄抓取区域照片
take_photo(area_description="grabbing_area")

# 2. 抓取指定的垃圾（例如：有苹果图案的方块）
grab_object(object_name="apple pattern block")

# 3. 拍摄放置区域照片
take_photo(area_description="placement_area")

# 4. 将物体移动到垃圾桶
move_to(target="trash bin", target_height=110)
```



