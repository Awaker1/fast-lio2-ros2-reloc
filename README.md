# WMJ_localization

本仓库保存WMJ战队2023赛季哨兵机器人所使用的建图定位算法，该算法主要基于激光惯性里程计**FAST-LIO**与**ROS2 Humble**软件框架。算法主要功能包括：建立场地激光点云地图功能，基于先验点云地图与初始位姿的场地定位功能。经过实机测试算法具有较强鲁棒性和准确性，能在较剧烈运动与干扰下实现场地定位。

该方法旨在解决以下几个问题：1. 在赛场上仅使用激光里程计时，容易受到动态物体干扰，同时每次里程计起始点手工确定，多次摆放一致性问题难以解决，使用地图先验进行激光匹配有效保证了地图一致性。2. 仅基于Lidar匹配的定位算法在遮挡较多的角落，被包围的情况下容易飘飞，经验证在几乎丢失全部点云的情况下，算法可以保持定位在原地一段时间，并在点云信息恢复后恢复定位精度。

## Related Works

1. [ikd-Tree](https://github.com/hku-mars/ikd-Tree): A state-of-art dynamic KD-Tree for 3D kNN search.
2. [IKFOM](https://github.com/hku-mars/IKFoM): A Toolbox for fast and high-precision on-manifold Kalman filter.
3. [FAST-LIO-ROS2 version](https://github.com/Ericsii/FAST_LIO)

## FAST-LIO

**FAST-LIO** (Fast LiDAR-Inertial Odometry) is a computationally efficient and robust LiDAR-inertial odometry package. It fuses LiDAR feature points with IMU data using a tightly-coupled iterated extended Kalman filter to allow robust navigation in fast-motion, noisy or cluttered environments where degeneration occurs. Our package address many key issues:

1. Fast iterated Kalman filter for odometry optimization;
2. Automaticaly initialized at most steady environments;
3. Parallel KD-Tree Search to decrease the computation;

## 1. Prerequisites

### 1.1 **Ubuntu** and **ROS**

目前算法仅在ROS2Humble环境下进行测试运行，ROS2早期版本兼容性未知，推荐使用ROS2 Humble (LTS)。

**Ubuntu == 22.04**

**ROS2 == Humble**

### 1.2. **PCL && Eigen**

PCL与Eigen版本均使用Ubuntu22.04，ROS2 Humble apt源安装，安装命令如下：

`sudo apt install ros-humble-pcl-ros `

`sudo apt install libeigen3-dev`

### 1.3. **livox_ros_driver**2

目前算法主要适配 **Livox-MID360** 激光雷达，其官方ROS driver地址:https://github.com/Livox-SDK/livox_ros_driver2 ,由于其适配ROS版本方法太ex，编译过程中会出现诸多问题，遂推荐使用云龙大佬修改的ros驱动:https://github.com/Ericsii/livox_ros_driver2/tree/feature/merge-ros ,该版本驱动可使用ROS2 colcon工具正常编译。

## 2. Build

克隆本仓库和**livox_ros_driver2**仓库，使用colcon工具进行编译（注：使用官方ros驱动请先按官方仓库指引编译好驱动source后再编译本仓库）

```shell
cd {your_workspace}/src
git clone $this_repo_url$
git clone $livox_ros_driver2_repo_url$
cd ..
colcon build --symlink-install
source install/local_setup.sh
```

## 3. Directly run

对于**Livox-MID360** 激光雷达

```
ros2 launch livox_ros_driver2 msg_MID360_launch.py
ros2 launch fast_lio mapping.launch.py
```

## Config params

### mid360.yaml

path: ./config/mid360.yaml

**修改参数说明**：

- map_file_path：点云地图保存/加载路径
- reloc_en：是否加载先验地图进行定位
- initialPose：机器人在地图坐标系下的启动位置（可由建图过程中输出的里程信息获得），使用先验地图定位功能时需确定机器人在地图中的初始位置。
- pcd_save_en：是否保存PCD点云地图，建图时开启，同时需保证map_en置true。

## Best Practice

首先，找到一段使用Livox-MID360录制的rosbag文件，或直接连接激光雷达进行测试。

**建图功能测试：**

修改config文件中：

```yaml
reloc_en: false
...
initialPose:
    use: false
...
map_en: true
...
pcd_save:
    pcd_save_en: true
...
```

发布雷达数据，启动建图节点：

```
ros2 launch fast_lio mapping.launch.py
```

可以打开rviz查看定位情况，建图完成后ctrl+c关闭即可保存地图，地图保存在指定的文件夹下，可使用`pcl_viewer`工具查看。注意，目前地图零点为程序起始位置，也可以通过设置initial pose改变起始点在地图中的位置。建议使用rosbag录制数据后回放数据建图，一方面防止异常终止，另一方面可以使用该数据验证后续地图定位功能。

**地图定位功能测试：**

修改config文件中：

```yaml
reloc_en: true
...

initialPose:
    use: true
...

pcd_save:
    pcd_save_en: false
...
```

建图生成的点云文件位置不变，根据当前机器人初始位置在地图坐标系下的位置修改initial pose，再次运行程序，打开rviz2查看位置是否收敛，稳定。

## Known Issues

- 激光雷达倒置时算法工作不稳定，极易出现飘飞情况，倾斜放置没有问题。

- 初始位姿设定与真实情况差距较大时（尤其是旋转差异较大时）容易飘飞。

## RoadMap

- [x] ROS2建图功能
- [x] 基于已有地图进行匹配定位
- [x] 异地初始化定位
- [ ] 优化代码结构，解决已知Bug
- [ ] 通过位姿状态量赋值实现定位初始化或全局重定位位姿调整
- [ ] 提高位置输出帧率
- [ ] 增加后端回环提高建图精度

## Developer

 王云飞 - QQ：2570506298

## Acknowledgments

Thanks for [Ericsii](https://github.com/Ericsii)[YLFeng](https://github.com/Ericsii)，https://github.com/Ericsii/FAST_LIO