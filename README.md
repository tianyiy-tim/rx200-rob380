# Trossen RX200 for ROB380
This will teach you how to set up and use the Trossen Robotics / Interbotix ReactorX 200 (RX200) robotic arm using ROS 2 Humble. This will include installing the Interbotix packages, connect and move the arm, and usage of the provided URDF atached in this repository.


## Contents
 
1. [What's in this repo](#whats-in-this-repo)
2. [Requirements](#requirements)
3. [Install the Interbotix packages](#install-the-interbotix-packages)
4. [Try it without the arm](#try-it-without-the-arm)
5. [Connect the arm](#connect-the-arm)
6. [Run the arm](#run-the-arm)
7. [Control the arm from Python](#control-the-arm-from-python)
8. [Using the URDF](#using-the-urdf)
9. [Troubleshooting](#troubleshooting)
10. [Reference](#reference)
---
 
## What's in this repo
 
| Path | What it is |
|---|---|
| `urdf/rx200_interbotix.urdf` | Standalone RX200 URDF, derived from Interbotix `interbotix_xsarm_descriptions` |
| `urdf/rx200_interbotix_with_meshes.zip` | Same URDF bundled with its STL meshes, for machines without the Interbotix packages |
| `README.md` | This guide |


## Requirements
### Hardware
- RX200 arm with its 12V power supply cable
- micro-USB cable (to tether the robot to your computer)
- A computer running **Linux** (see below for more information)

### Software
- Ubuntu 22.04 with ROS 2 Humble. The install script below installs ROS 2 for you
- Native Ubuntu is recommended but WSL2 on windows also works but will need an extra step to pass the USB device through
- MacOS cannot run this natively, so a virtual machine such as VirtualBox is recommended for use

## Install the Interbotix packages
 
Run Trossen's install script. It installs ROS 2 Humble (if missing), the Interbotix packages, and the USB rules for the arm.
 
```bash
sudo apt install curl
curl 'https://raw.githubusercontent.com/Interbotix/interbotix_ros_manipulators/main/interbotix_ros_xsarms/install/amd64/xsarm_amd64_install.sh' > xsarm_amd64_install.sh
chmod +x xsarm_amd64_install.sh
./xsarm_amd64_install.sh -d humble
```
 
When it asks for a robot model, enter `rx200`. It takes a while.
 
When it finishes, open a **new terminal** (or `source ~/.bashrc`) and check the install:
 
```bash
ros2 pkg list | grep interbotix
```
 
You should see packages such as `interbotix_xsarm_control` and `interbotix_xsarm_descriptions`.
 
> Every new terminal needs ROS sourced. The install script adds this to `~/.bashrc`, so normally nothing extra is needed. If `ros2` is "command not found", run `source /opt/ros/humble/setup.bash` and `source ~/interbotix_ws/install/setup.bash`.
 
## Try it without the arm
 
You can check that everything works before plugging in hardware.
 
**View the model with sliders:**
```bash
ros2 launch interbotix_xsarm_descriptions xsarm_description.launch.py robot_model:=rx200 use_joint_pub_gui:=true
```
RViz opens with the arm, plus a window of sliders. Moving a slider moves that joint.
 
**Simulated arm:** the control launch file can run without hardware, so you can test code with no arm attached:
```bash
ros2 launch interbotix_xsarm_control xsarm_control.launch.py robot_model:=rx200 use_sim:=true
```
The Python examples below work against this simulated arm too.
 
## Connect the arm
 
1. Clear the area around the arm.
2. Plug the **12 V power supply** into the arm.
3. Connect the **U2D2** to your computer with the USB cable.
4. Check that the computer sees it:
```bash
   ls /dev/ttyDXL
```
   If that file exists, you're ready. If not, see [Troubleshooting](#troubleshooting).
 
### WSL2 on Windows
 
WSL can't see USB devices until you pass them through with [usbipd-win](https://github.com/dorssel/usbipd-win).
 
1. Install usbipd-win on Windows (`winget install usbipd`).
2. In **PowerShell as Administrator**, find the U2D2 (it shows up as a USB serial device) and attach it:
```powershell
   usbipd list
   usbipd bind --busid <BUSID>
   usbipd attach --wsl --busid <BUSID>
```
3. In Ubuntu, check that it appeared with `ls /dev/ttyUSB*`. If `/dev/ttyDXL` is missing, run:
```bash
   sudo service udev restart
   sudo udevadm trigger
```
 
You need to run `usbipd attach` again each time you replug the arm or restart WSL.
 
## Run the arm
 
**1. Start the driver.** This connects to the motors and opens RViz:
```bash
ros2 launch interbotix_xsarm_control xsarm_control.launch.py robot_model:=rx200
```
The model in RViz should match the real arm's pose. Leave this terminal running.
 
**2. Read the joint angles.** In a second terminal:
```bash
ros2 topic echo /rx200/joint_states
```
Angles are in radians, in the order `waist, shoulder, elbow, wrist_angle, wrist_rotate, gripper, left_finger, right_finger`.
 
**3. Move it by hand (torque off).** **Hold the arm first.**
```bash
ros2 service call /rx200/torque_enable interbotix_xs_msgs/srv/TorqueEnable "{cmd_type: 'group', name: 'all', enable: false}"
```
Move the arm and watch RViz and the joint angles follow. Turn torque back on with `enable: true`.
 
**4. Shut down.** Put the arm in its sleep pose (resting on the cradle) before stopping the driver or cutting power, so it doesn't drop.
 
### Useful commands
 
| Command | What it does |
|---|---|
| `ros2 topic list` | List all topics |
| `ros2 service list` | List all services |
| `ros2 run tf2_ros tf2_echo rx200/base_link rx200/ee_gripper_link` | Print the gripper's live position relative to the base |
| `ros2 run tf2_tools view_frames` | Save a PDF of the full frame tree |
 
## Control the arm from Python
 
Interbotix provides a Python API, `InterbotixManipulatorXS`. With the driver from step 1 running, save this as `demo.py` and run `python3 demo.py`:
 
```python
from interbotix_xs_modules.xs_robot.arm import InterbotixManipulatorXS
from interbotix_common_modules.common_robot.robot import robot_startup, robot_shutdown
 
bot = InterbotixManipulatorXS(robot_model='rx200', group_name='arm', gripper_name='gripper')
robot_startup()
 
bot.arm.go_to_home_pose()
bot.arm.set_ee_pose_components(x=0.25, z=0.2)   # move the gripper to a point (meters)
bot.gripper.release()                           # open
bot.gripper.grasp()                             # close
bot.arm.set_single_joint_position('waist', 0.5) # rotate one joint (radians)
bot.arm.go_to_home_pose()
bot.arm.go_to_sleep_pose()
 
robot_shutdown()
```
 
Always end in the sleep pose. For more examples, see Trossen's [Python demos](https://docs.trossenrobotics.com/interbotix_xsarms_docs/python_ros_interface.html).
 
## Using the URDF 
### Where this URDF comes from
 
Interbotix describes all of its arms with shared **xacro** files: XML templates with variables and macros that expand into a URDF. `urdf/rx200_interbotix.urdf` is the RX200's xacro expanded into one plain file, so it can be used without the Interbotix tooling. It is the same model the driver uses.
 
To regenerate it (for example after an Interbotix update):
```bash
xacro $(ros2 pkg prefix interbotix_xsarm_descriptions)/share/interbotix_xsarm_descriptions/urdf/rx200.urdf.xacro robot_name:=rx200 > rx200_interbotix.urdf
```
 
### Structure
 
```
rx200/base_link
 └─ waist (revolute, z)        → shoulder_link
     └─ shoulder (revolute, y) → upper_arm_link
         └─ elbow (revolute, y)       → forearm_link
             └─ wrist_angle (revolute, y)  → wrist_link
                 └─ wrist_rotate (revolute, x) → gripper_link
                     └─ … gripper, left_finger / right_finger (prismatic) → ee_gripper_link
```
 
| Joint | Limits (deg) |
|---|---|
| waist | −180 to 180 |
| shoulder | −108 to 113 |
| elbow | −108 to 93 |
| wrist_angle | −100 to 123 |
| wrist_rotate | −180 to 180 |
 
All link names start with `rx200/` (for example `rx200/base_link`), matching the frames the driver publishes.
 
### Check and view it
 
```bash
sudo apt install liburdfdom-tools ros-humble-joint-state-publisher-gui
check_urdf urdf/rx200_interbotix.urdf
```
`check_urdf` prints the link tree if the file is valid.
 
To view it in RViz with sliders, use three terminals:
```bash
ros2 run robot_state_publisher robot_state_publisher --ros-args -p robot_description:="$(cat urdf/rx200_interbotix.urdf)"
ros2 run joint_state_publisher_gui joint_state_publisher_gui
rviz2
```
In RViz, set **Fixed Frame** to `rx200/base_link`, then **Add → RobotModel** and set its **Description Topic** to `/robot_description`.
 
### Meshes
 
The URDF draws each part from an STL mesh, referenced like `package://interbotix_xsarm_descriptions/meshes/rx200_meshes/base.stl`. These paths work on any machine with the Interbotix packages installed. Without them, use `rx200_interbotix_with_meshes.zip`, which contains the meshes in the same folder layout.
 
## Troubleshooting
 
| Problem | Try |
|---|---|
| `ros2: command not found` | Source ROS: `source /opt/ros/humble/setup.bash` and `source ~/interbotix_ws/install/setup.bash` |
| `/dev/ttyDXL` missing | Check power and the USB cable; replug; run `sudo udevadm trigger`. On WSL, re-run `usbipd attach` |
| Driver can't find motors | Arm not powered, or another program is using the port. Close other terminals running the driver |
| Permission denied on the port | Add yourself to the `dialout` group: `sudo usermod -aG dialout $USER`, then log out and back in |
| RViz model doesn't match the real arm | Check the joint angles with `ros2 topic echo /rx200/joint_states`. Known issue: wrist rotation may appear 90° off in RViz (under investigation) |
| A motor blinks red or stops responding | Overload or overheating. Power off, let it cool, power on again |
 
## Reference
 
- [Interbotix X-Series Arm docs (Trossen Robotics)](https://docs.trossenrobotics.com/interbotix_xsarms_docs/)
- [ROS 2 Software Setup](https://docs.trossenrobotics.com/interbotix_xsarms_docs/ros_interface/ros2/software_setup.html)
- [interbotix_ros_manipulators on GitHub](https://github.com/Interbotix/interbotix_ros_manipulators)
- [ROS 2 Humble docs](https://docs.ros.org/en/humble/)
- [URDF tutorials](https://docs.ros.org/en/humble/Tutorials/Intermediate/URDF/URDF-Main.html)
