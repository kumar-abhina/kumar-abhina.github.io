---
layout: page
title: Autonomous navigation and SLAM with turtlebot3
description: a project with autonmous navigation of slam and apriltag detction for hence making a resuce robot for recon missions in a simulated diaster zone
img: assets/Projects/Turtlebot_slam/T10.jpg
tags: formatting math
importance: 2
category: work

---

Introduction
In recent years, mobile robots have proven invaluable for **disaster respons** by enabling safe and efficient reconnaissance of dangerous environments. This project implements a 0**TurtleBot3** with **SLAM** and **AprilTag** detection in ROS to identify and localize “victims” (AprilTags) in an unknown environment.

## Problem Statement
Emergency scenarios like collapsed buildings or hazardous areas often require robots to:
1. **Map** the environment (2D occupancy grid).
2. **Identify** and **localize** points of interest (here, AprilTags) in the map.

Our approach simultaneously builds a SLAM map while detecting AprilTags using a camera, ensuring both tasks reinforce each other.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="/assets/Projects/Turtlebot_slam/T1.jpg" title="example image" class="img-fluid rounded z-depth-1" %}czxvbNM<>
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="/assets/Projects/Turtlebot_slam/T2.jpg" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="/assets/Projects/Turtlebot_slam/T11.jpg" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>


## Proposed Solution
We use a **TurtleBot3** equipped with:
- **LiDAR**: For environmental mapping.
- **Camera**: For detecting AprilTags at various angles/heights.
- **ROS** (Robot Operating System) Noetic, running on a Raspberry Pi.

Four main components:
1. **SLAM** (via [cartographer](http://wiki.ros.org/cartographer) or [gmapping](http://wiki.ros.org/gmapping))  
2. **Frontier-Based Exploration** ([explore_lite](http://wiki.ros.org/explore_lite))  
3. **AprilTag Detection** ([apriltag_ros](http://wiki.ros.org/apriltag_ros))  
4. **Camera–LiDAR Alignment** (either manual offset measurement or external calibration tools)

By filtering the LiDAR data to match (or approximate) the camera’s field of view, we compel the robot to explore areas that the camera can actually see, helping it find more tags.

---

## System Specification

### Robot Operating System (ROS)
[ROS](https://www.ros.org/) abstracts hardware drivers, provides message-passing tools, and unifies the software stack across nodes. We used **ROS Noetic** on **Ubuntu 20.04**.

### TurtleBot3 (Burger)
Key components:
- **Differential Drive**: Two wheels (radius ~0.033 m), track width ~0.16 m.
- **OpenCR** microcontroller.
- **Raspberry Pi 3** single-board computer (Ubuntu + ROS Noetic).
- **HLS-LFCD LiDAR** (or similar).
- **Pi Camera** module.

### Network and Master PC
A laptop (Ubuntu 20.04, ROS Noetic) acts as the **ROS Master**. The TurtleBot3 runs **ROS nodes** and connects via the same Wi-Fi (e.g., phone hotspot). Both devices require correct ROS IP configurations.

---

## ROS Packages Utilized

1. **cartographer** *(or gmapping)*  
   SLAM algorithm producing real-time 2D occupancy grid maps from LiDAR scans.
2. **explore_lite**  
   Frontier-based exploration that sends navigation goals to move_base until no frontiers remain.
3. **move_base**  
   A global/local planner that executes velocity commands to reach exploration or user-defined goals.
4. **apriltag_ros**  
   Detects AprilTags and broadcasts each tag’s pose and ID in a ROS topic (and TF transform).
5. **camera_calibration**  
   Calibrates the intrinsic camera parameters (distortion coefficients, camera matrix).
6. **lidar_filter** *(optional)*  
   Crops or filters LiDAR data to match the camera’s field of view, forcing the robot’s navigation to orient toward uncharted camera-visible areas.

---

## Camera Calibration

### Intrinsic Calibration
We used `camera_calibration` with a 9×7 checkerboard, gathering ~50 images from the Pi camera. This yields:
- **Distortion coefficients** (`D`)
- **Camera matrix** (`K`)
- **Rotation matrix** (`R`)
- **Projection matrix** (`P`)

### Extrinsic Calibration
Ideally, use a dedicated **LiDAR–camera extrinsic calibration** tool for accurate alignment. If unavailable or if hardware fails, manual measurement of camera’s relative position to the base_link (or LiDAR) can be used, updating the robot’s URDF accordingly.

### Motion Model
Given differential drive kinematics, the robot’s position in the world frame evolves as:
\[
\begin{bmatrix}
\dot{x} \\
\dot{y} \\
\dot{\theta}
\end{bmatrix}
=
\begin{bmatrix}
\cos(\theta) & 0 \\
\sin(\theta) & 0 \\
0 & 1
\end{bmatrix}
\begin{bmatrix}
\frac{r}{2}(\dot{\phi}_r + \dot{\phi}_l) \\
\frac{r}{w}(\dot{\phi}_r - \dot{\phi}_l)
\end{bmatrix}
\]
where \(r\) = wheel radius, \(w\) = track width, \(\dot{\phi}_{r,l}\) = right/left wheel angular velocities.

---

## Hardware Challenges

1. **LIDAR Motor Failure**: The scanning motor on the LiDAR occasionally stopped spinning, limiting scan readings.
2. **USB Port Mismatches**: The LiDAR would sometimes appear on `/dev/ttyUSB1` while the TurtleBot launch files expected `/dev/ttyAMC0`.
3. **System Reinstallation**: Conflicts in package versions (cartographer on Ubuntu 20.04) forced repeated reinstallation.

### Solution
- **Simulation**: Used Gazebo to verify code while the physical LiDAR was offline.  
- **Replace LiDAR**: Ultimately swapped in a functional LiDAR for real-world mapping.

---

## Results

### Simulation
We created a Gazebo world (e.g., `turtlebot3_house`) with 10 AprilTags randomly placed. Using **cartographer** + **explore_lite** + **apriltag_ros**:
- Detected **9 out of 10** tags (the final one was in an inaccessible corner).
- Verified the map’s accuracy via RViz.

### Environment Setup (Real Robot)
We placed 7 tags (IDs 0–6) around the test area at varying heights/orientations. The robot began at the entrance and navigated automatically (via **explore_lite**). Obstructions simulated debris or partial walls.

### Implementation

#### SLAM Implementation
- Attempted **cartographer**, faced install issues on Ubuntu 20.04 + ROS Noetic.
- Switched to **gmapping**, remapping LiDAR data through `scan_filtered` if needed.

#### AprilTag Pose Estimation
- Used **apriltag_ros** to detect tags in camera frames.
- Combined transforms from base_link → map (SLAM) and camera → base_link (manual or extrinsic calibration) to compute final AprilTag poses in the map frame.

#### Baseline Implementation
- **gmapping** + **explore_lite** + **unfiltered LiDAR** mapped everything quickly but only **4 of the 7** tags were detected (due to poor camera orientation on some tags).

#### Exploration Implementation
- **lidar_filter** restricted LiDAR scans to the camera’s approximate FOV.
- The robot reoriented toward camera-visible areas, finding **all 7 tags** with correct map positions.

---

## Conclusion
The project **successfully** demonstrates:
1. **SLAM-based mapping** of unknown, obstacle-filled environments.
2. **AprilTag detection** for “victim” identification, reporting each tag’s pose relative to the map.
3. **Frontier-based exploration** to autonomously traverse large or cluttered spaces.

**Key Trade-off**: Narrowing the LiDAR to match the camera’s FOV can degrade localization, but it boosts tag detection coverage. Incorporating IMU data or using a parallel full-range map could improve robustness.

---

## Future Work
1. **IMU Fusion**: Integrate IMU-based odometry (e.g., Extended Kalman Filter) to enhance localization when LiDAR scans are limited.  
2. **Parallel SLAM**: Run two SLAM pipelines—one with full LiDAR for localization and one filtered for exploration.  
3. **Accurate Extrinsic Calibration**: Use a calibration toolkit to fine-tune the camera–LiDAR transform for better tag pose accuracy.  
4. **Multi-Robot Exploration**: Deploy multiple TurtleBot3s for faster coverage of large disaster areas.

---

## References
1. Siciliano, B., & Khatib, O. (Eds.). (2016). *Springer Handbook of Robotics*. Springer International Publishing.  
2. [ROS Documentation](https://wiki.ros.org/) (Accessed 2023)  
3. [Cartographer ROS](http://wiki.ros.org/cartographer)  
4. [Apriltag ROS](http://wiki.ros.org/apriltag_ros)  
5. [Explore Lite](http://wiki.ros.org/explore_lite)  
6. [Move Base](http://wiki.ros.org/move_base)  

---

**Project Repository**: If you have a GitHub or GitLab link, you can place it here with a `[Project Repo](<URL>)` link.

> **Note**: All images and videos related to robot hardware, Rviz screenshots, or simulation should be placed in your `assets` folder (or a similar structure), then referenced here with standard Markdown syntax:
>
> ```md
> ![Robot in Action](/assets/img/robot_action.png)
> ```
>
> or
>
> ```md

> ```
>
> depending on your website’s setup.