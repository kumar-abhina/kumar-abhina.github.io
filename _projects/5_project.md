---
layout: page
title: Panda Arm Control with RL Algorithms
description: Deep Reinforcement Learning Tasks on Panda Arm.
img: assets/Projects/Panda_arm_RL/DDPG_episodes/5.gif
tags: formatting math
importance: 5
category: work
---
<a href="https://github.com/yashmewada9618/ROS-navigation-stack-using-IMU-and-RTK-GPS">GitHub1</a>

In this project, we explore cutting‐edge reinforcement learning (RL) algorithms to control the Panda Arm, a highly dexterous robotic manipulator. Our aim is to leverage RL techniques to enable adaptive and precise control in complex manipulation tasks. We have implemented two state‐of‐the‐art algorithms: Deep Deterministic Policy Gradient (DDPG) and Proximal Policy Optimization (PPO).

## Reinforcement Learning Overview for Panda Arm

The control of a robotic arm like the Panda involves high-dimensional state and action spaces. RL algorithms can learn policies directly from interaction data, which is especially useful when the dynamics are complex or partially unknown. In our experiments, we trained the Panda Arm in simulation and later transferred insights to the real system.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/Projects/Panda_arm_RL/reach.gif" title="Panda Arm Simulation" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    The Panda Arm simulated in a virtual environment, setting the stage for advanced RL training.
</div>

## DDPG: Deep Deterministic Policy Gradient

DDPG is an actor-critic, model-free algorithm tailored for continuous action spaces. It utilizes a deterministic policy, making it suitable for robotics applications. Here’s a brief overview of its mathematical formulation:

- **Actor-Critic Architecture:**  
  The actor network approximates a deterministic policy \( \mu(s|\theta^\mu) \) that maps states to actions. The critic network estimates the action-value function \( Q(s, a|\theta^Q) \).

- **Critic Update:**  
  The loss function for the critic is defined as:
  
  $$
  L(\theta^Q) = \frac{1}{N} \sum_{i=1}^N \left( Q(s_i, a_i|\theta^Q) - y_i \right)^2,
  $$
  
  where the target \( y_i \) is computed by:
  
  $$
  y_i = r_i + \gamma \, Q'\left(s_{i+1}, \mu'\left(s_{i+1}|\theta^{\mu'}\right)\Big|\theta^{Q'}\right).
  $$

- **Actor Update:**  
  The policy parameters are updated using the gradient:
  
  $$
  \nabla_{\theta^\mu} J \approx \frac{1}{N} \sum_{i=1}^N \nabla_a Q(s, a|\theta^Q)\Big|_{a=\mu(s_i)} \nabla_{\theta^\mu}\mu(s_i|\theta^\mu).
  $$

This framework allows the Panda Arm to learn smooth, continuous control strategies by directly mapping sensory inputs to motor commands.

<style>
.carousel-item img {
  max-height: 400px; /* Adjust this value as needed */
  object-fit: cover;
  width: 100%;
}
</style>

<div id="carouselExampleIndicators" class="carousel slide" data-ride="carousel">
  <ol class="carousel-indicators">
    <li data-target="#carouselExampleIndicators" data-slide-to="0" class="active"></li>
    <li data-target="#carouselExampleIndicators" data-slide-to="1"></li>
    <li data-target="#carouselExampleIndicators" data-slide-to="2"></li>
  </ol>
  <div class="carousel-inner">
    <div class="carousel-item active">
      {% include figure.liquid path="assets/Projects/Panda_arm_RL/slide.png" title="Image 1" class="d-block w-100" %}
    </div>
    <div class="carousel-item">
      {% include figure.liquid path="assets/Projects/Panda_arm_RL/stack.png" title="Image 2" class="d-block w-100" %}
    </div>
    <div class="carousel-item">
      {% include figure.liquid path="assets/Projects/Panda_arm_RL/reach.gif" title="Image 3" class="d-block w-100" %}
    </div>
  </div>
  <a class="carousel-control-prev" href="#carouselExampleIndicators" role="button" data-slide="prev">
    <span class="carousel-control-prev-icon" aria-hidden="true"></span>
    <span class="sr-only">Previous</span>
  </a>
  <a class="carousel-control-next" href="#carouselExampleIndicators" role="button" data-slide="next">
    <span class="carousel-control-next-icon" aria-hidden="true"></span>
    <span class="sr-only">Next</span>
  </a>
</div>


## PPO: Proximal Policy Optimization

PPO is known for its reliability and ease of implementation. It improves upon previous policy gradient methods by using a clipped surrogate objective, which ensures that policy updates do not deviate excessively from the previous policy. The key idea is to maintain stability during training. The PPO objective is formulated as:

$$
L^{CLIP}(\theta) = \hat{\mathbb{E}}_t \left[ \min\Big( r_t(\theta)\hat{A}_t, \; \text{clip}\Big(r_t(\theta), 1-\epsilon, 1+\epsilon\Big)\hat{A}_t \Big) \right]
$$

where:
$$
( r_t(\theta) = \frac{\pi_\theta(a_t \mid s_t)}{\pi_{\theta_{\text{old}}}(a_t \mid s_t)} \) is the probability ratio between the new and old policies.
\( \hat{A}_t \) is the estimated advantage at time \( t \).
\( \epsilon \) is a hyperparameter that controls the clip range.
$$


By constraining the update, PPO ensures a stable and monotonic improvement in the policy, making it an attractive option for real-world robotic control scenarios such as the Panda Arm.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid path="assets/Projects/Panda_arm_RL/DDPG_episodes/5.gif" title="disparity image" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid path="assets/Projects/Panda_arm_RL/DDPG_episodes/50.gif" title="disparity image" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid path="assets/Projects/Panda_arm_RL/DDPG_episodes/200.gif" title="disparity image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

## Visual Showcase

Every project has a beautiful feature showcase page. It’s easy to include images in a flexible 3-column grid format. Make your photos 1/3, 2/3, or full width to display the experimental setup, simulation environments, and results.

To give your project a background in the portfolio page, just add the `img` tag to the front matter like so:
