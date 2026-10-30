# Lab 09: ROS Actions & Universal Robots UR3e Control
**Course**: MXET 400 – Mechatronics II: Industrial Robotic Systems  
**Department**: Multidisciplinary Engineering Technology, Texas A&M University  

---

## 1. Overview & Objectives

In this lab, you transition from simple topic publishers/subscribers to industrial robot manipulation using **ROS 2 Actions** and **MoveIt 2**:
1. **Understand ROS 2 Actions**: Learn how long-running, goal-oriented tasks (such as robotic trajectory execution) provide feedback and goal preemption via `control_msgs/action/FollowJointTrajectory`.
2. **Industrial Robot Architecture**: Interface with an industrial Universal Robots UR3e manipulator using `ur_robot_driver` and `ur_client_library`.
3. **MoveIt 2 & Inverse Kinematics**: Use the MoveIt 2 motion planning framework to solve 3D Cartesian Inverse Kinematics and generate smooth, collision-free joint trajectories via OMPL.
4. **Trajectory Execution**: Execute motion planning goals on the physical UR3e robot arm in the robotics lab, or in an offline simulated environment at home using **Mock Hardware**.

---

## 2. Package Structure

```text
lab09/
├── README.md                          # This lab guide & command reference
└── mxet_ur_launch/                    # Custom TAMU launch & calibration package
    ├── package.xml
    ├── CMakeLists.txt
    ├── config/
    │   ├── ex-ur3e-1_calibration.yaml # Calibration parameters for Robot Station 1
    │   ├── ex-ur3e-2_calibration.yaml # Calibration parameters for Robot Station 2
    │   ├── ex-ur3e-3_calibration.yaml # Calibration parameters for Robot Station 3
    │   └── test_goal_publishers_config.yaml
    └── launch/
        ├── ex-ur3e-1.launch.py        # Driver launch file for Robot Station 1
        ├── ex-ur3e-2.launch.py        # Driver launch file for Robot Station 2
        ├── ex-ur3e-3.launch.py        # Driver launch file for Robot Station 3
        ├── moveit-ur3-1.launch.py     # MoveIt 2 launch file for Robot Station 1
        ├── moveit-ur3-2.launch.py     # MoveIt 2 launch file for Robot Station 2
        └── moveit-ur3-3.launch.py     # MoveIt 2 launch file for Robot Station 3
```

---

## 3. Getting Started & Workspace Build

First, install the UR driver prerequisites (if not already installed):

```bash
sudo apt update
sudo apt install -y ros-jazzy-ur-client-library ros-jazzy-ur-robot-driver
```

Build the package from your workspace root:

```bash
# 1. Navigate to workspace root
cd ~/ros2_ws

# 2. Build mxet_ur_launch
colcon build --symlink-install --packages-select mxet_ur_launch

# 3. Source the workspace
source install/setup.bash
```

---

## 4. Execution Workflow

### Workflow A: Physical UR3e Robot in Lab

#### Step 1: Network Configuration
1. Connect your workstation/laptop to the robot control box via a physical Ethernet cable.
2. If connected to campus Wi-Fi at the same time, ensure ROS 2 discovery is set to localhost in `~/.bashrc`:
   ```bash
   export ROS_AUTOMATIC_DISCOVERY_RANGE=LOCALHOST
   ```
3. Ensure your laptop's Ethernet adapter is configured statically on the `192.168.1.0/24` subnet:
   * **IP Address**: `192.168.1.5` (or `192.168.1.<laptop_number>`)
   * **Subnet Mask**: `255.255.255.0`
   * **Gateway**: `192.168.1.1` (or leave blank)
4. Disable Linux firewall (mandatory to allow incoming reverse connection on port 50002):
   ```bash
   sudo ufw disable
   ```

#### Step 2: Teach Pendant (PolyScope) Setup
1. On the UR3e Teach Pendant, navigate to: **Installation > URCaps > External Control**.
2. Verify "Remote host IP" matches your laptop's Ethernet IP (e.g. `192.168.1.5`) and custom port is `50002`.
3. Open or create a program named `Laptop Connection` containing only the `External Control` program node.
4. **Do NOT press Play yet.**

#### Step 3: Launch the UR3e Driver
In **Terminal 1**, run the command corresponding to your assigned physical robot station:

```bash
# Robot Station 1
ros2 launch mxet_ur_launch ex-ur3e-1.launch.py robot_ip:=192.168.1.31 ur_type:="ur3e" launch_rviz:=false

# Robot Station 2
ros2 launch mxet_ur_launch ex-ur3e-2.launch.py robot_ip:=192.168.1.32 ur_type:="ur3e" launch_rviz:=false

# Robot Station 3
ros2 launch mxet_ur_launch ex-ur3e-3.launch.py robot_ip:=192.168.1.33 ur_type:="ur3e" launch_rviz:=false
```

Once the terminal prints:
```text
[ros2_control_node-1] [INFO] [UR_Client_Library:]: Robot connected to reverse interface. Ready to receive control commands.
[spawner-6] [INFO] [spawner_scaled_joint_trajectory_controller]: Configured and activated scaled_joint_trajectory_controller
```
press **PLAY** on the Teach Pendant. The terminal will report successful RTDE synchronization and PolyScope's play icon will stay active (green).

#### Step 4: Launch MoveIt 2 & RViz
In **Terminal 2**:

```bash
# Robot Station 1
ros2 launch mxet_ur_launch moveit-ur3-1.launch.py ur_type:="ur3e"

# Robot Station 2
ros2 launch mxet_ur_launch moveit-ur3-2.launch.py ur_type:="ur3e"

# Robot Station 3
ros2 launch mxet_ur_launch moveit-ur3-3.launch.py ur_type:="ur3e"
```
*RViz will open displaying the full 3D UR3e model and the MotionPlanning interactive marker.*

---

### Workflow B: Offline / At-Home Simulation (No Robot Required)

You can develop, test, and verify motion planning without being in the robotics lab using **Mock Hardware**:

#### Terminal 1 (Start Mock Hardware Driver):
```bash
source ~/ros2_ws/install/setup.bash
ros2 launch ur_robot_driver ur_control.launch.py ur_type:=ur3e robot_ip:=127.0.0.1 use_mock_hardware:=true launch_rviz:=false
```

#### Terminal 2 (Start MoveIt 2 & RViz):
```bash
source ~/ros2_ws/install/setup.bash
ros2 launch ur_moveit_config ur_moveit.launch.py ur_type:=ur3e launch_rviz:=true
```

---

## 5. Motion Planning in RViz

1. **Interact with the End-Effector**:
   * Drag the colored arrows (Cartesian translation) or circular rings (rotation) at the robot's tool center point (TCP).
   * Notice that the orange ghost robot updates its joint angles automatically using **Inverse Kinematics (IK)**.
2. **Execute a Trajectory**:
   * In the bottom-left panel, select the **MotionPlanning** display.
   * Under the **Planning** tab, verify **Planning Group** is set to `ur_manipulator`.
   * Click **Plan**: MoveIt will compute a collision-free path and display a preview animation.
   * Click **Execute**: The trajectory action client will send the goal to the robot driver, and the physical (or simulated) robot arm will move smoothly to the goal.

---

## 6. Inspecting Joint States & Actions

In **Terminal 3**:
```bash
# View the current joint angles, velocities, and efforts
ros2 topic echo /joint_states --once

# List all active action servers (notice /scaled_joint_trajectory_controller/follow_joint_trajectory)
ros2 action list

# Inspect the action definition
ros2 interface show control_msgs/action/FollowJointTrajectory
```

---

## 7. Checkpoint Checklist

Before completing your lab report, verify:
* [ ] Workstation successfully connects to the UR3e controller over Ethernet.
* [ ] MoveIt 2 launches and displays the 3D model with interactive markers in RViz.
* [ ] At least 2 Cartesian waypoints planned and executed smoothly on the robot arm.
* [ ] Captured `/joint_states` output for your lab report.
* [ ] Answered all Lab Questions in your report (IP addressing, message types, and IK vs FK).

