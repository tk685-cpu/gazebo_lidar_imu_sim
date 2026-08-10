# gazebo_lidar_imu_sim

A ROS 2 package that simulates a differential-drive robot equipped with a
**3D spinning lidar** (16-channel, Velodyne-VLP16-style) and an **IMU** in
**Gazebo** (via `ros_gz`).

## Stack

Verified against:

- ROS 2 **Humble**
- Gazebo **Fortress** (`ignition-gazebo` 6 / `ign gazebo`)
- `ros_gz_sim` / `ros_gz_bridge` 0.244.x for the ROS 2 <-> Gazebo bridge

installed via a [RoboStack](https://robostack.github.io/) conda environment
(the standard way to get ROS 2 + Gazebo on macOS, since there are no native
macOS binaries). On Linux the same package works with a regular
`apt install ros-humble-desktop ros-humble-ros-gz` setup.

To target ROS 2 Jazzy + Gazebo Harmonic (`gz sim`) instead, swap the plugin
`filename` attributes in `urdf/robot.urdf.xacro` and
`worlds/lidar_imu_world.sdf` from `ignition-gazebo-*-system` back to
`gz-sim-*-system`.

## What's in the robot

- Simple box chassis on two differential-drive wheels + a passive caster
- **3D lidar** (`gpu_lidar` sensor): 360° horizontal x 16 vertical channels
  over a ±15° vertical FOV, published as:
  - `/lidar/points` (`sensor_msgs/PointCloud2`) — full 3D point cloud
  - `/lidar` (`sensor_msgs/LaserScan`) — single mid-plane ring
- **IMU** (`imu` sensor): published as `/imu` (`sensor_msgs/Imu`)
- Odometry from the diff-drive plugin on `/odom`, TF on `/tf`,
  `/joint_states` for the wheel joints

The world (`worlds/lidar_imu_world.sdf`) is a flat ground plane with a box,
a rotated box, and a cylinder scattered around so the lidar has something to
return.

## Install

### macOS (RoboStack / conda)

```bash
# if you don't already have a ROS 2 Humble + ros_gz conda env:
conda create -n ros2_humble -c robostack-staging -c conda-forge \
    ros-humble-desktop ros-humble-ros-gz ros-humble-robot-state-publisher \
    ros-humble-joint-state-publisher ros-humble-xacro ros-humble-rviz2
conda activate ros2_humble

mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src
git clone https://github.com/tk685-cpu/gazebo_lidar_imu_sim.git
cd ~/ros2_ws
colcon build --symlink-install
source install/setup.bash
```

### Linux (Ubuntu 22.04 + Humble)

```bash
sudo apt install ros-humble-desktop ros-humble-ros-gz \
    ros-humble-robot-state-publisher ros-humble-joint-state-publisher \
    ros-humble-joint-state-publisher-gui ros-humble-xacro ros-humble-rviz2 \
    ros-humble-rviz-imu-plugin

mkdir -p ~/ros2_ws/src && cd ~/ros2_ws/src
git clone https://github.com/tk685-cpu/gazebo_lidar_imu_sim.git
cd ~/ros2_ws && colcon build --symlink-install && source install/setup.bash
```

## Run

```bash
ros2 launch gazebo_lidar_imu_sim sim.launch.py
```

This starts Gazebo (server-only, `-s`, by default — see the macOS note
below), spawns the robot, starts the ROS 2 <-> Gazebo bridge (lidar, IMU,
odom, TF, joint states, cmd_vel, clock), and opens RViz2 pre-configured to
show the point cloud, IMU, TF, and robot model. Pass `use_rviz:=false` to
skip RViz, or `extra_gz_args:=''` on Linux for the full Gazebo GUI window
instead of `-s`.

Drive the robot around to see the lidar sweep obstacles and the IMU react
to motion:

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

Inspect topics:

```bash
ros2 topic hz /lidar/points
ros2 topic echo /imu --once
```

## Verified on this repo's dev machine (macOS, RoboStack Humble + Fortress)

- ✅ Package builds (`colcon build`), xacro compiles, and the generated URDF
  converts to valid SDF (`ign sdf -p` / `ign sdf -k`).
- ✅ Full launch brings up Gazebo, spawns the robot, and the bridge connects
  all topics (`/clock`, `/cmd_vel`, `/odom`, `/tf`, `/joint_states`, `/imu`,
  `/lidar`, `/lidar/points`).
- ✅ `/imu` publishes real data (gravity read back as ~9.79 m/s² on Z).
- ✅ Publishing to `/cmd_vel` drives the robot and updates `/odom` correctly.
- ⚠️ **The lidar sensor itself crashes Gazebo on this specific macOS setup.**
  Gazebo Fortress's `sensors-system` plugin needs an Ogre2 rendering window
  to compute lidar (and camera) returns — even in server-only (`-s`) mode —
  and creating that window off the main thread crashes with
  `NSWindow should only be instantiated on the main thread!` on macOS. This
  is an upstream limitation of this Gazebo version's renderer on macOS, not
  a bug in this package: the same URDF/SDF sensor definitions are standard
  and match Gazebo's own bundled example
  (`ignition-gazebo6/worlds/gpu_lidar_sensor.sdf`). Everything else in the
  package (chassis, wheels, diff-drive, IMU, bridge, spawn, RViz config) was
  exercised end-to-end and works.
  - **Workaround**: run this package on native Linux, or inside a Linux
    Docker container/VM on this Mac — lidar/camera rendering works normally
    there. Search `gazebosim/gz-sim` GitHub issues for "macOS" + "rendering"
    if you want to track upstream fixes.

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
└── rviz/
    └── lidar_imu.rviz
```

## License

MIT — see [LICENSE](LICENSE).
