# FreeFleet with Zenoh-DDS Bridge (Detailed Setup in Virtual Environment)

[![ROS2](https://img.shields.io/badge/ROS2-Humble%20-green.svg)](https://docs.ros.org/en/rolling/)
[![Open-RMF](https://img.shields.io/badge/Open--RMF-Compatible-blue.svg)](https://open-rmf.org/)
[![Zenoh](https://img.shields.io/badge/Zenoh-1.0+-purple.svg)](https://zenoh.io/)
[![Python](https://img.shields.io/badge/Python-3.8%20%7C%203.9%20%7C%203.10-blue.svg)](https://www.python.org/)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)

This repository provides a **complete, production-ready deployment** of **FreeFleet** (Open-RMF fleet management) using the **Zenoh ROS2 DDS Bridge** for scalable, low-latency multi-robot communication.

**Key Requirements:**
- All Python dependencies installed inside a dedicated **virtual environment** named `ff_env`
- **NumPy version strictly < 2.0** (e.g., 1.26.4) to ensure compatibility with ROS2 dependencies (especially `numpy` in `cv_bridge`, `nav2`, and scientific packages)
- Tested on Ubuntu 22.04 (ROS2 Humble) 

This setup is ideal for managing 10–100+ autonomous mobile robots (AMRs) in warehouses, hospitals, or factories with reliable fleet monitoring and task dispatching.

## Why This Setup?

- **Zenoh Bridge** eliminates DDS discovery bottlenecks → perfect for Wi-Fi-heavy fleets
- **Virtual environment** (`ff_env`) isolates dependencies → no pollution of system Python
- Clone only with vcs import **rmf-humble-latest.repos**
- **NumPy < 2.0** avoids ABI breaks in ROS2 packages (common issue in 2025 with NumPy 2.0+)

## Prerequisites

- Ubuntu 22.04 
- ROS2 Humble or Jazzy installed system-wide
- Python 3.8–3.10
- Git

## Step-by-Step Installation (with Virtual Environment)

## 1. Create & Activate Virtual Environment

```bash
# Create a dedicated virtual environment
python3 -m venv ~/ff_env

# Activate it (do this every time you work on the project)
source ~/ff_env/bin/activate

# Upgrade pip inside venv
pip install --upgrade pip setuptools wheel

#install cycloneddds for fleet
sudo apt update && sudo apt install python3-pip ros-humble-rmw-cyclonedds-cpp

```

### 1.1 To install Zenoh features 
```bash

# Change preferred zenoh version here
export ZENOH_VERSION=1.5.0

# Download and extract zenoh-bridge-ros2dds release
wget -O zenoh-plugin-ros2dds.zip https://github.com/eclipse-zenoh/zenoh-plugin-ros2dds/releases/download/$ZENOH_VERSION/zenoh-plugin-ros2dds-$ZENOH_VERSION-x86_64-unknown-linux-gnu-standalone.zip
unzip zenoh-plugin-ros2dds.zip

# If using released standalone binaries of zenoh router, download and extract the release
# wget -O zenoh.zip https://github.com/eclipse-zenoh/zenoh/releases/download/$ZENOH_VERSION/zenoh-$ZENOH_VERSION-x86_64-unknown-linux-gnu-standalone.zip
# unzip zenoh.zip

```

## 2. CycloneDDS Configuration (Best Practice for Zenoh Bridge)

To get **maximum performance and zero Wi-Fi spam** with large fleets, configure CycloneDDS to use only loopback (localhost). Zenoh handles all inter-robot communication – no direct DDS traffic between robots.

### 2.1. Set Environment Variables (Add to Your Daily Startup)

Add these lines to your `~/start_fleet.sh` script (or ~/.bashrc):

```bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
export CYCLONEDDS_URI=file://$HOME/.ros/cyclonedds.xml
export ROS_DOMAIN_ID=0          # Recommended: Use Domain ID 0 (default)
# (Your previous example used 1 – change to 0 unless you have a reason)
```
## 3. RMF packages
- Clone the respective repositaries for the final implementation of fleet using **rmf-humble-latest.repos** given in the package. Eventually create a separate workspace for it.
 ``` bash
cd rmf_ws/src
vcs import . < rmf-humble-latest.repos
cd ..
colcon build --symlink-install
source install/setup.bash 
rosdep install --from-paths src --ignore-src --rosdistro humble -yr
```
- These cloning process should not be inside  virtual environment
- Then follow up the setup guide for openrmf using the documentation https://osrf.github.io/ros2multirobotbook/

## 4. Multi-Robot Environemnt

- Create a multirobot environment by namespacing URDF or namespacing TF alone which will create multiple robot instances and frames even tf trees.
- After namespacing , launch the following files
### 4.1 Gazebo launch

 ``` bash
ros2 launch amr_bot gazebo.launch.py use_sim_time:=true
 ``` 
### 4.2 Localization launch 
 ``` bash
ros2 launch amr_bot multi_robot_localization.launch.py use_sime_time:=true map:=~/(your_workspace_name)/src/amr_bot/maps/new_map.yaml
```
### 4.3 Rviz
 ``` bash
 rviz2 -d ~/workspace_name/src/bcr_bot/rviz/multibot.rviz
  ``` 
### 4.4 Navigation Launch
 ``` bash
 ros2 launch amr_bot multi_robot_navigation.launch.py use_sim_time:=true
  ``` 
### 4.5 Collision Monitor Launch 
In order to make the robot move initialize collision monitor node
``` bash
ros2 launch amr_bot collision_monitor_node.launch.py use_sim_time:=true
``` 


## 5. Implementation of Fleet

Examples for running a single robot or multiple robots in simulation has been up in amr_bot package, along with example configuration files for zenoh as well as fleet configuration files for free_fleet_adapter.
### 5.1 Initialize Zenoh Router
- copy the command in the separate terminal 
``` bash
zenohd
```

### 5.2 Running Zenoh bridge with DDSof ROS2 
- copy the command in the separate terminal

 ``` bash
cd PATH_TO_EXTRACTED_ZENOH_BRIDGE
./zenoh-bridge-ros2dds -c ~/amr_ws/src/amr_bot/config/zenoh/nav2_unique_multi_amrbot_zenoh_bridge_ros2dds_client_config.json5
 ```
### 5.3 Launch the Common launch file
- This launch file contains nodes of rmf_traffic , rmf_schedule, building_map_server, door_supervisor, lift_supervisor and building.yaml which built from traffic editor to create lanes and nodes for traffic management etc.

 ``` bash
ros2 launch amr_bot warehouse_world_rmf_common.launch.xml
 ``` 

### 5.4  Launch the fleet adapter 
- This launch file contains navigation graphs for the robot to travel in a specified path, and then configuration file for the each and individual robots.

- Make sure that reference coordinates are very very important with respect to building map_server which is in rmf_ws. So that, robots can able to move with respect to nav2_map.

 ``` bash
ros2 launch amr_bot multiple_bots_fleets_adapter.launch.xml
 ``` 

 NOTE: When launching the two fleet launch files make sure it is inside virtual environment so that python packages may not affect

 - https://docs.ros.org/en/humble/Installation/RMW-Implementations/Non-DDS-Implementations/Working-with-Zenoh.html take it as a reference while installing Zenoh.

## Conclusion

This project delivers a **fully integrated, production-ready multi-robot fleet management system** built on **Open-RMF FreeFleet** and **ROS 2 (Humble)**, enhanced with the **Eclipse Zenoh-DDS bridge** for exceptional scalability and reliability in real-world environments.

By combining:
- Precise multi-robot simulation and deployment in Gazebo
- Robust localization and Navigation2 stacks with namespaced TF trees
- Full RMF interoperability (traffic scheduling, doors, lifts, building maps)
- Efficient communication via Zenoh over CycloneDDS (eliminating Wi-Fi discovery bottlenecks)
- Clean dependency isolation through a dedicated virtual environment with NumPy < 2.0

the system achieves safe, structured, and coordinated navigation for multiple AMRs—even in complex, dynamic indoor settings like warehouses or hospitals.

The use of **rmf-humble-latest.repos** with `vcs import` ensures reproducible, up-to-date access to the complete Open-RMF ecosystem, while the Zenoh bridge enables seamless scaling from a few robots to large fleets without network congestion or complex namespacing.

This setup not only demonstrates best practices for modern ROS 2 multi-robot systems in 2025 but also provides a solid, extensible foundation for real-world deployments requiring high reliability, traffic-aware planning, and centralized fleet control.

Whether for simulation testing or physical robot fleets, this configuration empowers developers and operators to deploy autonomous mobile robots with confidence, efficiency, and future-proof scalability.

**Your fleet is now ready—safe, coordinated, and truly multi-robot capable.**



