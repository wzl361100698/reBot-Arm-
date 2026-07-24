开启虚拟机后先远程连接rdk
连接时命令有：

先ping rdk ip地址	检查是否能连通
ssh sunrise@rdk ip地址       远程连接上虚拟机

# 安装pinocchio(两端都要安装)
sudo apt update
sudo apt install -y \
  ros-humble-pinocchio \
  python3-numpy \
  python3-pip
# 安装meshcat（都要安装）
/usr/bin/python3 -m pip install --user meshcat

# 安装 can-utils
sudo apt update
sudo apt install -y can-utils


## RDK端

ls -l /dev/ttyACM*
# 清理之前可能残留的 SLCAN 进程
pkill -f slcand
# 删除旧的 slcan0；若提示不存在可忽略
ip link delete slcan0
# 将 CANable 转为 SocketCAN，-s8 为 1 Mbps
slcand -o -c -s8 /dev/ttyACM0 slcan0
# 启用接口
ip link set slcan0 up
# 确认状态
ip -details link show slcan0


# =======================================串口权限加启用接口
sudo slcand -o -c -s8 /dev/ttyACM0 slcan0
sudo ip link set slcan0 up
ip -details link show slcan0
=======================================
# 启动
cd ~/rebotarm_ros2
source /opt/ros/humble/setup.bash
source install/setup.bash
export ROS_DOMAIN_ID=10
export ROS_LOCALHOST_ONLY=0
ros2 launch rebotarm_bringup driver.launch.py \
  model:=dm channel:=slcan0

# 扫描七个电机
motorbridge-cli scan \
  --vendor damiao \
  --transport auto \
  --channel slcan0 \
  --start-id 1 \
  --end-id 7

## 在 RDK X5 新开终端，确认状态正常：
source /opt/ros/humble/setup.bash
source ~/rebotarm_ros2/install/setup.bash
export ROS_DOMAIN_ID=10
export ROS_LOCALHOST_ONLY=0
ros2 topic echo --once /rebotarm/joint_states
ros2 topic echo --once /rebotarm/arm_status

# 确认机械臂周围无人、无障碍物后，使能：
ros2 service call /rebotarm/enable std_srvs/srv/Trigger
# 电机通常会从红灯变为绿灯，机械臂会变得有保持力；这是正常现象。
---------------------------------------------------------------------------------------------------
## Ubuntu端
# 在 Ubuntu 电脑上打开新终端，连接同一个 ROS 域：
cd ~/rebotarm_ros2
source /opt/ros/humble/setup.bash
source install/setup.bash

export ROS_DOMAIN_ID=10
export ROS_LOCALHOST_ONLY=0

ros2 node list

# 在 Ubuntu 上启动真实硬件版 MoveIt：
ros2 launch rebotarm_moveit_config hardware.launch.py

### RViz 打开后： 
 先只点击 Plan；
 检查规划轨迹是否安全；
 确认无误后才点击 Execute；
 暂时不要点击 Plan & Execute。
###
-----------------------------------------------------------------------------------------
# 归位（需要启动ros2 launch rebotarm_bringup driver.launch.py \model:=dm channel:=slcan0才有效）
ros2 service call /rebotarm/safe_home std_srvs/srv/Trigger
-------------------------------------------------------------------------------------------------------
# 环境检测
cd ~/rebotarm_ros2/third_party/reBotArm_control_py
source /opt/ros/humble/setup.bash
export PYTHONPATH="$(pwd):${PYTHONPATH}"
python3 -c "import pinocchio; import reBotArm_control_py; print('环境正常')"

# 快速运动的逆运动学（不要启动ros2 launch rebotarm_bringup driver.launch.py \model:=dm channel:=slcan0）
python3 example/7_arm_ik_control.py
# 例：
#用法A
> 0.3 0.0 0.4 # 仅控制位置（姿态默认为0），让机械臂末端走到前方 0.3 米，上方 0.4 米的位置。
#用法B
> 0.3 0.0 0.4 0.0 0.0 0.5 #同时控制位置和姿态：走到指定位置，同时手腕偏航角旋转 0.5 弧度。
> ctrl + c # 退出系统
----------------------------------------------------------------------------------------------------------------

# 平滑运动的逆运动学（不要启动ros2 launch rebotarm_bringup driver.launch.py \model:=dm channel:=slcan0）
uv run python example/8_arm_traj_control.py
# 例：
#用法A
> 0.3 0.0 0.4 #仅指定位置，姿态默认为 0，移动时间默认为 2.0 秒
#用法B
> 0.3 0.0 0.4 0.0 0.0 0.5 #同时控制位置和姿态：走到指定位置，同时手腕偏航角旋转 0.5 弧度，移动时间默认为 2.0 秒
#用法C
> 0.3 0.0 0.4 0.0 0.0 0.0 5.0 #让机械臂走到特定位置，并指定用 5.0 秒 的时间慢慢挪过去。(注意：如果要输时间，前方的姿态参数 0 0 0 不能省略)
> ctrl + c # 退出系统
------------------------------------------------------------------------------------------------------------
# 重力补偿测试（不要启动ros2 launch rebotarm_bringup driver.launch.py \model:=dm channel:=slcan0）

# 预期行为
# 机械臂可以在任意姿态下"漂浮"
# 松开后不会因自重坠落
# 可以手动掰动到任意位置

uv run python example/9_gravity_compensation.py
# 停止脚本（Ctrl+C）时，程序会直接失能所有电机，机械臂不会自动回到零点。请在退出前用手扶住机械臂或先将其移动到安全/归零姿态，避免关节突然下落造成碰撞或损伤。
-------------------------------------------------------------------------------------------------------------------------------------------------
# reBotArm 的 Pinocchio/MeshCat 可视化

# 下载meshcat-python
conda deactivate
source /opt/ros/humble/setup.bash
/usr/bin/python3 -m pip install --user meshcat
# 验证：
/usr/bin/python3 -c "import meshcat; print('MeshCat 安装成功:', meshcat.__file__)"
# 如果提示没有 pip：
sudo apt update
sudo apt install -y python3-pip

## 先在 Ubuntu 虚拟机终端执行：
conda deactivate
cd ~/rebotarm_ros2/third_party/reBotArm_control_py
ls example/sim/fk_sim.py
# 如果能看到文件，准备环境：（如果报错，则是未安装pinocchio或者meshcat）
source /opt/ros/humble/setup.bash
export PYTHONPATH="$(pwd):${PYTHONPATH}"

/usr/bin/python3 -c "import numpy, pinocchio, meshcat; print('环境正常')"
# 如果缺少 MeshCat：
/usr/bin/python3 -m pip install --user meshcat
# 然后启动：
/usr/bin/python3 example/sim/fk_sim.py
# 或者
/usr/bin/python3 example/sim/ik_sim.py
# 启动后终端一般会打印类似：
http://127.0.0.1:7000/static/

# 输入格式：
# 仅位置：x y z（米）
# 位置+姿态：x y z roll pitch yaw（弧度）
# 示例：

> 0.25 0.0 0.25              # 仅位置
> 0.25 0.0 0.25 0 0 0        # 位置+姿态
---------------------------------------------------------------------------------------------
# 正运动学仿真 (sim/fk_sim.py)实体机械臂不会动
cd ~/rebot_grasp/sdk/reBotArm_control_py
# （进入你的sim/fk_sim.py文件所在文件夹）

source /opt/ros/humble/setup.bash
export PYTHONPATH="$(pwd):${PYTHONPATH}"

.venv/bin/python example/sim/fk_sim.py
# 输入q退出
-----------------------------------------------------------------------------------------
# 逆运动学仿真 (sim/ik_sim.py)实体机械臂不会动
cd ~/rebot_grasp/sdk/reBotArm_control_py
# （进入你的sim/ik_sim.py文件所在文件夹）

source /opt/ros/humble/setup.bash
export PYTHONPATH="$(pwd):${PYTHONPATH}"
.venv/bin/python example/sim/ik_sim.py
# 输入q退出
-------------------------------------------------------------------------------------------
# 轨迹仿真学（同上）
uv run python example/sim/traj_sim.py
-----------------------------------------------------------------------------------------
# 先进入 SDK 根目录：
cd ~/rebot_grasp/sdk/reBotArm_control_py
source /opt/ros/humble/setup.bash
export PYTHONPATH="$(pwd):${PYTHONPATH}"

# 如果还没有解决 NumPy 冲突，先执行：
uv pip install --python .venv/bin/python "numpy==1.26.4"
# 启动 Python：
.venv/bin/python
# 看到 >>> 后逐行输入：
import numpy as np
from example.sim.visualizer import Visualizer

viz = Visualizer()
viz.neutral()
# 终端会显示 MeshCat 地址，例如：
# http://127.0.0.1:7000/static/
# 在 Ubuntu 浏览器中打开它。
# 更新机械臂姿态：
q = np.radians([10, -20, -30, 0, 0, 0])
viz.update(q)
# 恢复模型中位姿态：
viz.neutral()
# 绘制一条红色三维路径：
points = [
    [0.20, 0.00, 0.20],
    [0.25, 0.00, 0.25],
    [0.30, 0.05, 0.30],
]

viz.draw_path(points, "test_path", color=0xFF0000)
# 清除路径：
viz.clear_paths()
# 退出输入：
exit()
# 关节角传给 viz.update() 时必须使用弧度，所以示例用 np.radians() 将角度转换为弧度。这一过程是纯仿真，不会连接或移动真实机械臂。

-----------------------------------------------------------------------------------------------------------------------------
## 视觉demo

# 部署方式
Ubuntu 虚拟机
├─ RGB-D 相机
├─ YOLO / 视觉识别
├─ 深度计算和抓取点生成
└─ MoveIt / 运动规划
          │ ROS 2 网络
          ▼
RDK X5 4GB
├─ reBotArmController
├─ slcan0 / CANable
└─ 达妙机械臂驱动

# Step 1. 克隆仓库
# 优先使用 Seeed-Projects 官方仓库：
git clone https://github.com/Seeed-Projects/reBot-DevArm-Grasp.git rebot_grasp
cd rebot_grasp

# 也可以使用当前开发仓库：
git clone https://github.com/EclipseaHime017/reBot-DevArm-Grasp.git rebot_grasp
cd rebot_grasp
# Step 2. 创建并配置 conda 环境
conda env create -f environment.yml
conda activate rebotarm
# Step 3. 安装机械臂控制库
git clone https://github.com/vectorBH6/reBotArm_control_py.git sdk/reBotArm_control_py
cd sdk/reBotArm_control_py
pip install -e .
cd ../..
# Step 4. 安装深度相机 SDK
cd ~/rebot_grasp
conda activate rebotarm

python -m pip install --upgrade pyorbbecsdk2 \
  -i https://mirrors.aliyun.com/pypi/simple/ \
  --timeout 120 --retries 10

# 注意：
关闭 Ubuntu：
sudo poweroff
在 VMware 主界面选择该虚拟机，点击：
编辑虚拟机设置
→ USB 控制器
→ USB 兼容性
→ USB 3.1（没有则选 USB 3.0）
→ 确定

# 验证相机是否连接成功
lsusb
lsusb -t

应该出现类似：
Bus 004 Device 002: ID 2bc5:0803 Orbbec Gemini 336
并在 lsusb -t 中位于 xhci_hcd 下，速度为 5000M 或更高。
---------------------------------------------------------------------------------------------
# 验证安装环境
python -c "import pyorbbecsdk; print('pyorbbecsdk OK')"
-------------------------------------------------------------------------------------------
# 先临时开放 USB 权限：
sudo chmod a+rw /dev/bus/usb/*/*

# 为避免每次重新插相机都要授权，添加永久规则：
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="2bc5", MODE="0666"' \
  | sudo tee /etc/udev/rules.d/99-orbbec.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
# 然后拔插相机，并在VMware中再次选择“连接到虚拟机”。
# 重新插拔相机，


# 然后执行：
python -c "from pyorbbecsdk import Context; d=Context().query_devices(); print('检测到摄像头数量:', d.get_count())"
# 必须输出：
检测到摄像头数量: 1

# 配置相机和轻量模型
cd ~/rebot_grasp
nano config/default.yaml
# 确认相机配置：
camera:
  type: orbbec_gemini2
  serial: null
  color_width: 1280
  color_height: 720
  fps: 30
# 建议把 YOLO 改成较小模型：
yolo:
  model_name: "yoloe-26s-seg.pt"
  device: "cpu"
  use_world: true

  保存退出：
Ctrl+O
回车
Ctrl+X

# 整体检查系统和环境配置
echo "===== 系统 ====="
lsb_release -a
uname -m
echo "===== 基础软件 ====="
command -v conda || true
conda --version 2>/dev/null || true
python3 --version
git --version
echo "===== Conda 环境 ====="
conda env list 2>/dev/null || true
echo "===== USB 设备 ====="
lsusb
lsusb -t
echo "===== 机械臂串口 ====="
ls -l /dev/ttyACM* /dev/ttyUSB* 2>/dev/null || true
echo "===== 是否存在旧控制进程 ====="
ps -ef | grep -E '[s]lcand|[r]ebotarm|[r]os2'
echo "===== 已有项目 ====="
find ~ -maxdepth 3 -type d \( -name "rebot_grasp" -o -name "reBotArm_control_py" \) 2>/dev/null
echo "===== GPU（没有也没关系） ====="
nvidia-smi 2>/dev/null || echo "虚拟机内没有可用 NVIDIA GPU"
# 如果无法识别到摄像头，可以检查一下是不是USB的协议太低导致的，关闭虚拟机后更改USB协议至3.0

----------------------------------------------------------------------------------
# 测试视觉


# 先安装 CPU 版 PyTorch：
python -m pip install \
  --index-url https://download.pytorch.org/whl/cpu \
  torch torchvision torchaudio
# 然后安装项目指定版本。国内网络可以使用清华源：
python -m pip install \
  -i https://pypi.tuna.tsinghua.edu.cn/simple \
  "numpy<2.0" \
  "ultralytics==8.4.35" \
  "opencv-python==4.7.0.72" \
  "opencv-contrib-python==4.7.0.72"
# 验证：
python -c "import numpy, torch, cv2, ultralytics; print('NumPy:',numpy.__version__)

# Gemini 336 不支持项目当前设置的硬件深度对齐，需要改成软件对齐。
cd ~/rebot_grasp
cp drivers/camera/orbbec_gemini2.py \
   drivers/camera/orbbec_gemini2.py.bak
sed -i 's/OBAlignMode\.HW_MODE/OBAlignMode.SW_MODE/' \
  drivers/camera/orbbec_gemini2.py
grep -n "set_align_mode" drivers/camera/orbbec_gemini2.py




# 运行
conda activate rebotarm
python scripts/object_detection.py

------------------------------------------------------------------------
# 检查串口
lsusb
ls -l /dev/ttyACM*
# 检查相机：
cd ~/rebot_grasp
conda activate rebotarm

python -c "
from pyorbbecsdk import Context
d = Context().query_devices()
print('相机数量:', d.get_count())
"

# 必须显示：
# 相机数量: 1

# 启动 CAN 接口
sudo pkill -f slcand
sudo slcand -o -c -s8 /dev/ttyACM0 slcan0
sudo ip link set slcan0 up
ip -details link show slcan0

# 然后扫描电机：
motorbridge-cli scan \
  --vendor damiao \
  --transport auto \
  --channel slcan0 \
  --start-id 1 \
  --end-id 7

 #6 1. 修改机械臂SDK配置
cd ~/rebot_grasp

cp sdk/reBotArm_control_py/config/rebotarm_dm.yaml \
   sdk/reBotArm_control_py/config/rebotarm_dm.yaml.bak

sed -i 's|^channel:.*|channel: slcan0|' \
  sdk/reBotArm_control_py/config/rebotarm_dm.yaml

grep -n '^channel:' \
  sdk/reBotArm_control_py/config/rebotarm_dm.yaml

  ------------------------------------------------------
# 检测环境是否正常
cd ~/rebot_grasp
conda activate rebotarm

export PYTHONPATH="$PWD/sdk/reBotArm_control_py${PYTHONPATH:+:$PYTHONPATH}"

python -c "from reBotArm_control_py.actuator import RebotArm; import motorbridge; print('SDK 和 motorbridge 都正常')"

# 手眼标定
python scripts/collect_handeye_eih.py

# 视觉识别定位（G执行，Q退出，R恢复实时）
python scripts/main.py --dry-run



#  执行夹取命令
  python scripts/main.py

###
初始化 RGB-D 相机，确认图像流可用
机械臂与夹爪使能，移动到预备高位
实时相机预览 + YOLO 目标检测与实例分割
OBB 短轴估计夹爪朝向，深度分位数估计抓取高度
按 G 冻结帧，经手眼变换计算机械臂目标位姿
机械臂移动到预抓取点 → 下降 → 夹爪闭合 → 提升 → 回预备位
###


# 抓取与放置程序
  scripts/set.py --dry-run
###
相机与机械臂初始化，移动到预备点位
实时相机预览 + YOLO 目标检测与实例分割
按 G 冻结帧，经手眼变换计算机械臂目标位姿
机械臂移动抓取香蕉并抬高
机械臂将香蕉放置在盒子内，并回归初始姿态
按 Q 退出系统，机械臂回归零点
###
