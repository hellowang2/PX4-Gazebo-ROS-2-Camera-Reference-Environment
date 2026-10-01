# PX4-Gazebo-ROS-2-Camera-Reference-Environment
PX4 + Gazebo + ROS 2 Camera Reference Environment
這是一個已經驗證過的 PX4 SITL + Gazebo Harmonic + ROS 2 Jazzy + Micro XRCE-DDS + X500 mono camera Docker 環境，提供專題成員作為模擬與 ROS 2 相機串接的參考。
> 這不是完整的人員追蹤系統。  
> 本環境目前只驗證到 **Gazebo 相機影像能進入 ROS 2**。YOLO / ByteTrack / PX4 tracking controller 請自行整合。
已驗證功能
PX4 SITL
Gazebo Harmonic
ROS 2 Jazzy
Micro XRCE-DDS Agent
X500 mono camera
`ros_gz_image`
Gazebo camera image → ROS 2
ROS 2 camera topic 可正常發布
測試時相機資訊：
```text
resolution: 1280 x 960
encoding: rgb8
rate: 約 14–17 FPS
```
實際 FPS 會依電腦效能而不同。
---
1. 需求
建議環境：
Windows 11
WSL2
Ubuntu 24.04
Docker Desktop
Docker Desktop 已啟用 WSL Integration
WSLg 可正常顯示 Linux GUI
---
2. 載入 Docker Image
取得：
```text
uav-px4-ros2-camera-ok.tar.gz
```
後，在 WSL 中進到檔案所在位置並執行：
```bash
docker load < uav-px4-ros2-camera-ok.tar.gz
```
確認 image：
```bash
docker image ls uav-px4-ros2:camera-ok
```
應可看到：
```text
uav-px4-ros2   camera-ok
```
---
3. 啟動 Container
【Terminal A：WSL】
```bash
docker run --rm -it --name px4-gazebo \
  --mount type=bind,src=/mnt/wslg/.X11-unix,dst=/tmp/.X11-unix,readonly \
  -e DISPLAY=:0 \
  -e QT_QPA_PLATFORM=xcb \
  -e LIBGL_ALWAYS_SOFTWARE=1 \
  -e ROS_DOMAIN_ID=83 \
  -e PX4_SIM_MODEL=gz_x500_mono_cam \
  uav-px4-ros2:camera-ok
```
進入 container shell 後啟動 Micro XRCE-DDS Agent：
```bash
MicroXRCEAgent udp4 -p 8888
```
這個 Terminal 保持開啟。
---
4. 啟動 PX4 SITL + Gazebo
【Terminal B：另外開一個 WSL】
```bash
docker exec -it px4-gazebo /usr/local/bin/ros2-entrypoint.sh px4-gazebo
```
正常情況下會啟動：
PX4 SITL
Gazebo Harmonic GUI
X500 mono camera model
PX4 ↔ Micro XRCE-DDS ↔ ROS 2
Gazebo 中應能看到 X500 無人機。
---
5. 將 Gazebo Camera Bridge 到 ROS 2
【Terminal C：另外開一個 WSL】
先進 container：
```bash
docker exec -it px4-gazebo /usr/local/bin/ros2-entrypoint.sh bash
```
啟動 image bridge：
```bash
ros2 run ros_gz_image image_bridge \
/world/default/model/x500_mono_cam_0/link/camera_link/sensor/camera/image
```
此 Terminal 保持開啟。
ROS 2 camera topic：
```text
/world/default/model/x500_mono_cam_0/link/camera_link/sensor/camera/image
```
---
6. 驗證 ROS 2 是否收到 Camera Image
【Terminal D：另外開一個 WSL】
```bash
docker exec -it px4-gazebo /usr/local/bin/ros2-entrypoint.sh bash
```
查看 image topic：
```bash
ros2 topic list | grep image
```
查看發布頻率：
```bash
ros2 topic hz \
/world/default/model/x500_mono_cam_0/link/camera_link/sensor/camera/image
```
查看影像格式：
```bash
ros2 topic echo \
/world/default/model/x500_mono_cam_0/link/camera_link/sensor/camera/image \
--once | head -n 12
```
已驗證的輸出包含：
```text
frame_id: camera_link
height: 960
width: 1280
encoding: rgb8
step: 3840
```
---
7. 顯示 Camera 畫面
在 Terminal D 中：
```bash
ros2 run rqt_image_view rqt_image_view
```
在 `rqt_image_view` 上方選擇：
```text
/world/default/model/x500_mono_cam_0/link/camera_link/sensor/camera/image
```
預設 Gazebo world 幾乎只有灰色天空與灰色地面，因此沒有加入人物或其他物件時，camera 畫面看起來可能只是一條地平線，這是正常現象。
---
8. 目前做到哪裡
目前已完成：
```text
PX4 SITL
    ↕
Micro XRCE-DDS
    ↕
ROS 2
    ↑
Gazebo Harmonic
    ↓
X500 mono camera
    ↓
ros_gz_image
    ↓
ROS 2 sensor_msgs/Image
```
下一階段請自行整合，例如：
```text
ROS 2 camera image
        ↓
YOLO person detection
        ↓
ByteTrack
        ↓
target bbox / track ID
        ↓
tracking controller
        ↓
PX4 Offboard control
```
本 Docker image 沒有預先完成上述追蹤與控制整合。
---
9. 停止
各 Terminal 可使用：
```text
Ctrl + C
```
停止對應程式。
若要直接停止 container：
```bash
docker stop px4-gazebo
```
由於啟動時使用 `--rm`，container 停止後會被刪除，但：
```text
uav-px4-ros2:camera-ok
```
這個 Docker image 不會因此消失。
---
說明
此環境用途是提供一個「已確認 Camera → ROS 2 可運作」的模擬 baseline，方便後續自行進行 perception、tracking 與 PX4 controller 整合。
