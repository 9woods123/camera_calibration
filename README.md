
# **相机与IMU标定方法**

## **1. 准备工作**

### 前置条件：

* 安装 **ROS**，建议使用 **Ubuntu 20.04** 和 **ROS Noetic**。
* 安装一些必要的软件包和依赖项。

#### 安装依赖：

* **对于 Ubuntu 20.04**，安装以下软件包：

```bash
sudo apt-get install python3-catkin-tools python3-osrf-pycommon
python3 -m pip install empy
python3 -m pip install catkin-pkg
```

#### 安装其他常用依赖：

```bash
sudo apt-get install -y git wget autoconf automake nano \
libeigen3-dev libboost-all-dev libsuitesparse-dev \
doxygen libopencv-dev libpoco-dev libtbb-dev \
libblas-dev liblapack-dev libv4l-dev libceres-dev \
libdw-dev
```

对于不同版本的 Ubuntu，请根据需要安装 Python 和相关依赖：

* **Ubuntu 20.04** 安装：

  ```bash
  sudo apt-get install -y python3-dev python3-pip python3-scipy \
      python3-matplotlib ipython3 python3-wxgtk4.0 python3-tk \
      python3-igraph python3-pyx
  ```

### 安装并配置 **Kalibr** 工具包：

在开始标定之前，确保你已经克隆并编译好 **Kalibr** 工具包。

1. 进入 **Kalibr** 目录并编译：

```bash
mkdir -p calibration_ws/src
cd calibration_ws/src
git clone https://github.com/9woods123/camera_calibration.git
cd ..
catkin_make
```

2. 配置环境：

```bash
cd calibration_ws && source devel/setup.bash
```

---

## **2. 生成标定目标二维码**

使用 **Kalibr** 工具生成标定目标二维码。二维码可以打印在不同纸张（如 A4 和 A3）上。以下是生成标定二维码的命令：

### 生成 A4 纸的二维码：

```bash
rosrun kalibr kalibr_create_target_pdf --type apriltag --nx 5 --ny 5 --tsize 0.0336 --tspace 0.2  ## for A4
```

### 生成 A3 纸的二维码：

```bash
rosrun kalibr kalibr_create_target_pdf --type apriltag --nx 5 --ny 5 --tsize 0.0475 --tspace 0.2  ## for A3
```

打印出来后，将二维码纸平整贴在墙上，确保没有折叠或变形，避免影响图像提取。

---

## **3. 录制数据**

使用 **rosbag** 工具录制相机图像数据和IMU数据。以下是录制命令：

```bash
rosbag record /camera/infra1/image_raw /camera/infra2/image_raw /imu/data -O camera_calibration.bag
```

此命令会录制相机的 **`/camera/infra1/image_raw`** 和 **`/camera/infra2/image_raw`** 图像话题，以及 **IMU** 的 **`/imu/data`**，并保存为 `camera_calibration.bag` 文件。

---

## **4. 相机标定**

录制完成后，使用 **Kalibr** 工具进行相机标定。执行以下命令进行标定：

```bash
cd calibration_ws && source devel/setup.bash

rosrun kalibr kalibr_calibrate_cameras \
    --target /camera_calibration/yaml_files/apil.yaml \
    --bag /camera_calibration/rosbags/camera_calibration.bag \
    --models pinhole-radtan pinhole-radtan \
    --topics /camera/infra1/image_rect_raw /camera/infra2/image_rect_raw \
    --show-extraction
```

### 参数说明：

* **--target**：标定目标的配置文件（通常为 `apil.yaml`）。
* **--bag**：录制的ROS bag文件，包含图像和IMU数据（即 `camera_calibration.bag`）。
* **--models**：相机模型类型，`pinhole-radtan` 表示透视模型加畸变参数。
* **--topics**：相机的图像话题。
* **--show-extraction**：显示提取过程，用于检查图像是否正确提取。

---

## **5. IMU和相机联合标定**

如果你需要同时标定 **IMU** 和 **相机** 的关系，可以使用以下命令进行联合标定：

```bash
rosrun kalibr kalibr_calibrate_imu_camera \
    --target /camera_calibration/yaml_files/apil.yaml \
    --bag /camera_calibration/rosbags/camera_calibration.bag \
    --cam /camera_calibration/camera_calibration-camchain.yaml \
    --imu /camera_calibration/yaml_files/imu.yaml \
    --show-extraction
```


```bash
<!-- rosrun kalibr kalibr_calibrate_imu_camera \
    --target /home/easy/easy_ws/camera_calibration/src/camera_calibration/yaml_files/apil.yaml \
    --bag /home/easy/easy_ws/camera_calibration/src/camera_calibration/bags/camera_record_2025-11-20-17-21-02.bag.active.bag \
    --cam /home/easy/easy_ws/camera_calibration/src/camera_calibration/camera_calibration-camchain.yaml \
    --imu /home/easy/easy_ws/camera_calibration/src/camera_calibration/yaml_files/imu.yaml \
    --show-extraction -->

```
### 参数说明：

* **--cam**：相机链配置文件，通常由 **kalibr_calibrate_cameras** 生成，格式为 `camchain.yaml`。
* **--imu**：IMU配置文件，包含IMU话题和统计信息，通常为 `imu.yaml`。
* **--show-extraction**：显示提取过程，用于检查图像和IMU数据是否正确提取。

---

## **6. 参数说明**

在进行标定时，需要提供以下输入：

* **--bag filename.bag**：包含图像和IMU数据的 **rosbag** 文件。
* **--cam camchain.yaml**：相机链的配置文件，包含内参和外参。
* **--imu imu.yaml**：IMU的配置文件，包含IMU话题和统计信息。
* **--target target.yaml**：标定目标的配置文件，描述了标定目标二维码的位置、大小等信息。

---

## **7. 注意事项**

* **标定目标放置**：
  确保标定目标（二维码）平整贴在墙上，避免折叠、变形等情况，以免影响图像提取。

* **稳定运动**：
  在录制数据时，相机和IMU应保持稳定运动，避免震动或不规则运动轨迹。

* **光照条件**：
  确保标定目标有足够的光照，避免过暗或强光照射，影响标定精度。

* **检查结果**：
  标定完成后，查看输出的日志文件和优化结果，检查标定误差。如果误差较大，可以考虑重新调整参数或重新录制数据。

---


### **附录：常见错误排查**

1. **"Could NOT find PY_em" 错误**：

   * 如果遇到 `empy` 模块找不到的问题，可以通过以下命令安装：

     ```bash
     python3 -m pip install empy
     ```

2. **"No module named 'catkin_pkg'" 错误**：

   * 如果遇到 `catkin_pkg` 模块找不到的问题，可以通过以下命令安装：

     ```bash
     python3 -m pip install catkin-pkg
     ```

---

