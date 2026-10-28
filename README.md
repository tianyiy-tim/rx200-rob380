# Trossen RX200 for ROB380
This will teach you how to set up and use the Trossen Robotics / Interbotix ReactorX 200 (RX200) robotic arm using ROS 2 Humble. This will include installing the Interbotix packages, connect and move the arm, and usage of the provided URDF atached in this repository.


## Contents


## Requirements
### Hardware
- RX200 arm with its 12V power supply cable
- micro-USB cable (to tether the robot to your computer)
- A computer running **Linux** (see below for more information)

### Software
- Ubuntu 22.04 with ROS 2 Humble. The install script below installs ROS 2 for you
- Native Ubuntu is recommended but WSL2 on windows also works but will need an extra step to pass the USB device through
- macOS unfortunately is not supported for running the arm directly so users are recommended to use a CAEN computer with WSL2

## Install the Interbotix Packages
