---
layout: page
title: Manipulation Stack for the Unitree G1 Robot &Robot learning
description: Built a custom Python control stack for the Unitree G1 Robot for Inverse kinematics solver, Controller and Motion Planner 
img: assets/Projects/Unitree_G1/thumbnail.gif
tags: formatting math
importance: 5
category: work
---
## Unitree G1 Humanoid Arm — Building a Full-Stack Manipulation Pipeline in Python

After validating ideas on the Panda arm, I shifted focus to the **Unitree G1**, a 7-DOF humanoid manipulator.  
My goal: write everything—from inverse kinematics to joint-level control—in pure Python on top of Unitree’s DDS-based SDK, then push cycle‐times low enough for reactive tasks like placing Lego bricks.

### 1. Inverse Kinematics (7-DOF) with Pinocchio + CasADi

The G1’s extra elbow joint gives useful redundancy but complicates IK. I:

1. **Modelled the arm in Pinocchio** to obtain fast analytical Jacobians and geometry trees.  
2. Formulated IK as a **non-linear least-squares** problem in CasADi:  

   \[
   \min_{\boldsymbol{q}} \; \|\, f(\boldsymbol{q}) - \mathbf{x}_{\text{goal}} \|_2^2
   \]
   subject to joint limits, self-collision inequality constraints, and a **null-space regularizer** that biases solutions toward comfort poses.

3. Solved with **Levenberg–Marquardt** (damped Newton) plus an adaptive λ that shrinks when the solver is near convergence.  
4. Achieved **< 2 ms** median runtime for typical end-effector targets, fast enough to call IK inside the planner loop.

### 2. Sampling-Based Motion Planner

I implemented an **RRT-Connect-style planner** that calls the IK solver at each expansion:

* **Tree node:** Cartesian pose → joint solution (from IK) → collision check (`pinocchio.geometry`).  
* **Extension metric:** combined translational, rotational, and joint-space distance with tunable weights.  
* **Early shortcutting:** a first-pass smoothing that removes way-points already linearly reachable in collision-free space.  
* **Planning rate:** ≈ 60 Hz on a single core, yielding 100–300 way-points for a 0.5 m reach maneuver.

### 3. Trajectory Generation & Optimization

Raw way-points are fed into a two-step pipeline:

1. **Cubic-Spline Interpolator** — produces joint trajectories with continuous velocity/acceleration.  
2. **Time-Optimal Parameterization (TOPP)** — scales the spline to respect joint velocity \( \dot q_{\max} \) and acceleration \( \ddot q_{\max} \) limits.  
   * Added an **energy-minimizing mode** that lengthens segments with low torques to reduce heat during long horizon tasks.

Result: trajectories that track within 0.1 rad/s and 0.3 rad/s² of hardware limits while staying smooth.

### 4. Real-Time Controller

* **Joint-space PID at 1 kHz**, gravity terms computed from the Pinocchio model and refreshed every control tick.  
* **Trajectory blending:** a quintic blend window that lets new splines splice into the active motion without abrupt torque jumps.  
* **Safety layer:** soft-torque limit, velocity clamping, and an *instant hold* mode triggered by a watchdog if no new command is heard within 50 ms.

### Dual-Arm Policy Training with LeRobot

I’m also training an action policy for the dual-arm Unitree G1 using the LeRobot pipeline: synchronized camera streams feed a vision encoder, joint states and gripper signals form the proprioceptive branch, and the policy is optimized with behavior cloning plus action smoothing to keep impedance-friendly torques. I gathered demonstrations with a teleoperation pipeline and trained both ACT and diffusion policies for everyday tasks—toast bread, block stacking, snack picking, and similar manipulation—while using domain randomization on lighting and hand-held objects so the network transfers from staged demos to real assembly motions without brittle heuristics.

<div class="row justify-content-center">
  <div class="col-sm mt-3 mt-md-0">
      {% include video.liquid path="assets/Projects/Unitree_G1/Motion Planner.mp4" class="img-fluid rounded z-depth-1" controls=true autoplay=true %}
  </div>
</div>
### 5. System Integration

| Aspect | Value |
|--------|-------|
| Programming language | Python 3.12 |
| SDK transport | Unitree DDS (ROS-agnostic) |
| End-to-end latency | **≈ 4 ms** (planner→controller→actuator→feedback) |
| Simulation twin | MuJoCo 3 + identical Python stack |

<div class="row">
  <div class="col-sm mt-3 mt-md-0">
    {% include figure.liquid path="assets/Projects/Unitree_G1/G1_lidar_odom.PNG" title="IK Convergence" class="img-fluid rounded z-depth-1" %}
  </div>
  <div class="col-sm mt-3 mt-md-0">
    {% include video.liquid path="assets/Projects/Unitree_G1/V2.mp4" title="Planner Roll-out" class="img-fluid rounded z-depth-1" %}
  </div>
</div>

<div class="row justify-content-center">
  <div class="col-sm-5 mt-3 mt-md-0">
      {% include video.liquid path="assets/Projects/Unitree_G1/clideo_editor_0306915380f6438888ba426299c62219.mp4" title="Multi-step Pick" class="img-fluid rounded z-depth-1" controls=true autoplay=true %}
  </div>
  <div class="col-sm-5 mt-3 mt-md-0">
      {% include video.liquid path="assets/Projects/Unitree_G1/copy_88FE5E70-92FF-40A2-883C-2CC1693A1EA9.MOV" title="Fine Alignment Trial" class="img-fluid rounded z-depth-1" controls=true autoplay=true %}
  </div>
</div>


### Current Achievements

* **Sub-millimetre accuracy** in peg-in-hole and pick-and-place trials.  
* **35 % faster cycle-time** than Unitree’s stock planner by avoiding blocking IK calls.  
* Modular architecture—IK, planner, generator, and controller can be swapped or tuned independently.

### Next Milestone — Autonomous Lego Stacking

> **Vision:** use the stack to pick identical Lego bricks and place them precisely atop one another until a tower forms.

Planned steps:

1. **RGB-D Perception:** point-cloud segmentation to localize the top brick surface.  
2. **Grasp Planning:** sampling antipodal grips with friction cones checked in real time.  
3. **Visual Servoing:** fine motion within 2 cm using end-effector camera feedback.  
4. **Adaptive Compliance:** impedance control overlay to absorb tiny alignment errors while stacking.

Completion of this task will demonstrate dexterous, repeatable assembly using a fully Pythonic control stack on a commercial humanoid platform.

---
