# gazebo_lidar_imu_sim

A ROS 2 package that simulates a differential-drive robot equipped with a
**3D spinning lidar** (16-channel, Velodyne-VLP16-style) and an **IMU** in
**Gazebo** (new Gazebo / `gz sim`, via `ros_gz`).

## Stack

- ROS 2 (developed against **Jazzy**; should also work on Humble with
  `gz-fortress`/`ros_gz` — see [Notes](#notes) below)
- Gazebo **Harmonic** (`gz sim`)
- `ros_gz_sim` / `ros_gz_bridge` for the ROS 2 <-> Gazebo bridge

## What's in the robot

- Simple box chassis on two differential-drive wheels + a passive caster
- **3D lidar** (`gpu_lidar` sensor): 360° horizontal x 16 vertical channels
  over a ±15° vertical FOV, published as:
  - `/lidar/points` (`sensor_msgs/PointCloud2`) — full 3D point cloud
  - `/scan` (`sensor_msgs/LaserScan`) — single mid-plane ring
- **IMU** (`imu` sensor): published as `/imu` (`sensor_msgs/Imu`)
- Odometry from the diff-drive plugin on `/odom`, TF on `/tf`

The world (`worlds/lidar_imu_world.sdf`) is a flat ground plane with a box,
a rotated box, and a cylinder scattered around so the lidar has something to
return.

## Install

Prerequisites: ROS 2 Jazzy + Gazebo Harmonic + `ros_gz`, e.g. on Ubuntu 24.04:

```bash
sudo apt install ros-jazzy-desktop ros-jazzy-ros-gz \
    ros-jazzy-robot-state-publisher ros-jazzy-joint-state-publisher \
    ros-jazzy-joint-state-publisher-gui ros-jazzy-xacro ros-jazzy-rviz2
```

Clone into a colcon workspace and build:

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src
git clone https://github.com/tk685-cpu/gazebo_lidar_imu_sim.git
cd ~/ros2_ws
colcon build --symlink-install
source install/setup.bash
```

## Run

```bash
ros2 launch gazebo_lidar_imu_sim sim.launch.py
```

This starts Gazebo with the world, spawns the robot, starts the ROS 2 <->
Gazebo bridge (lidar, IMU, odom, TF, joint states, cmd_vel, clock), and
opens RViz2 pre-configured to show the point cloud, IMU, TF, and robot
model. Pass `use_rviz:=false` to skip RViz.

Drive the robot around to see the lidar sweep obstacles and the IMU react
to motion:

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard --ros-args -r /cmd_vel:=/cmd_vel
```

Inspect topics:

```bash
ros2 topic hz /lidar/points
ros2 topic echo /imu --once
```

## Package layout

```
gazebo_lidar_imu_sim/
├── urdf/
│   ├── robot.urdf.xacro   # chassis, wheels, caster, diff-drive plugin
│   ├── lidar.xacro        # 3D gpu_lidar sensor macro
│   └── imu.xacro          # imu sensor macro
├── worlds/
│   └── lidar_imu_world.sdf
├── launch/
│   └── sim.launch.py
├── config/
│   └── ros_gz_bridge.yaml # ROS <-> Gazebo topic bridge map
└── rviz/
    └── lidar_imu.rviz
```

## Notes

- This was authored and structured for correctness but **not run/tested on
  this machine** (no ROS 2 / Gazebo installed here) — if a topic name or
  plugin filename needs a tweak for your exact distro combination, check
  `ros2 topic list` / `gz topic -l` after `sim.launch.py` starts, and adjust
  `config/ros_gz_bridge.yaml` accordingly.
- To target ROS 2 Humble + Gazebo Fortress instead, swap the plugin
  filenames in `urdf/robot.urdf.xacro` / `worlds/lidar_imu_world.sdf` from
  `gz-sim-*-system` to `ignition-gazebo-*-system`, and install
  `ros-humble-ros-gz` instead.

## License

MIT — see [LICENSE](LICENSE).
