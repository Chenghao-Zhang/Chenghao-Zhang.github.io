---
permalink: /
title: "Chenghao Zhang"
excerpt: ""
author_profile: true
redirect_from: 
  - /blog/
  - /blog.html
---

This blog briefly recalls my experience leading the battle strategy development for RoboX:

## Table of contents

1. [Preface](#Preface)
2. [Our Scheme with RL](#Our Scheme with RL)
3. [Competition](#Competition)
4. [Conclusion](#Conclusion)

## Preface

At the beginning, after analyzing and discussing the official framework, we divided the tasks into three areas: SLAM localization, automatic aiming, and battle strategy. Shiao's group handled SLAM, while my group took charge of battle strategy, in addition to the pre-existing automatic aiming technology. The SLAM part was already well-developed within the official framework, so our goal was mainly to understand its principles to help troubleshoot if localization issues arose later. I had worked on the automatic aiming system last year, using traditional vision algorithms and stereo depth estimation. This year, we aimed for more precise targeting, and the control team assisted in that aspect. For battle strategy, we wanted to use reinforcement learning, which is cutting-edge research. My group had solid programming skills, and we focused on building a battlefield simulator for reinforcement learning training. We held weekly meetings to track progress, troubleshoot issues, and propose solutions when needed. By the winter break, most of the work was completed.



Before the new semester started, the robots hadn’t arrived at school yet, so we spent time improving the simulator. The initial simulator only implemented basic functions like movement and shooting. During this period, we added features like path planning, ammunition replenishment, and Buff zones to closely simulate the real competition environment. Once the semester began and the official robots arrived, we split into two teams. My group continued refining the simulator and developing reinforcement learning, while the remaining team worked on integrating the official framework, SLAM localization, and automatic aiming with the real robots. I also helped them resolve various issues they encountered. One memorable bug occurred when we realized the robot's speed caused localization errors. After checking all the parameters, we discovered the problem was the LiDAR's insufficient rotation speed, leading to distorted scan data at high speeds, which disrupted localization. The solution was simple: we replaced the LiDAR with a better one. Other issues, like power supply problems, setting up the LiDAR’s TF tree, and installing PyTorch and TensorFlow on the onboard computers, were eventually resolved.



During the first half of the semester, most functionalities were implemented on the real robot, although some small issues remained. SLAM localization stabilized after solving the LiDAR issues. The automatic aiming system, though switched to a stereo camera, wasn’t very reliable, working fine at night but failing during the day. As for battle strategy, reinforcement learning enabled us to achieve enemy pursuit, but when the enemy wasn't visible, the model often converged to a stationary state rather than patrolling as we had hoped. Although there was significant improvement from last year, we felt the current system wasn’t ready for competition and decided to revise some of the modules.



The most significant change was to the automatic aiming module, which wasn’t reliable enough for real competition. On the suggestion of one team member, we switched to deep learning, using Tiny YOLO for object detection. Testing on the real robot showed much better stability, with speeds reaching over 10fps, which was sufficient. Ultimately, we adopted this approach.



## Our Scheme with RL

Regarding the battle strategy I was responsible for, here's a more detailed breakdown. From the official platform, it’s clear that teams are expected to focus on strategy. Our team used a combination of traditional state machines and reinforcement learning. The state machine controlled global strategies, such as when to start the match, reload, or use reinforcement learning, as these actions followed fixed patterns. Reinforcement learning was used for local strategies, and this season, we only implemented the pursuit feature. Pursuit required quick reactions to enemy movements, so reinforcement learning provided a flexible approach. Here's how we implemented pursuit using reinforcement learning:



**Simulation Environment**  

Before deploying the reinforcement learning algorithm, we needed to design a realistic simulation environment for model training. We developed it based on OpenAI Gym's Top-down car environment. The Top-down car simulates a vehicle driving on a road, and we used its vehicle movement controls, while adding the real ICRA competition's field boundaries, obstacles, bullets, supply zones, and Buff zones. We mainly referred to the box2d tutorial and its Python interface. To facilitate reinforcement learning training, we simplified the real field:

\- Ignored vehicle acceleration and controlled movement by directly setting speed, with no friction.

\- Ignored armor plates; bullets cause damage upon hitting the vehicle, and vehicle collisions also cause damage.

\- Set the time limit to 2 minutes instead of 7 to prevent meaningless standoffs that slow down training.

\- Set the initial ammo to 200 so the model can focus on offense.



**Reinforcement Learning Algorithm**  

In the initial technical proposal, we planned to use the global map as the state, global target points as the actions, health advantage as the reward, and DQN as the core model. However, we encountered the following issues during deployment:

\- We couldn’t fully migrate the path planning from the official RoboRTS framework to the simulation environment. The paths in the simulation often differed from those in reality.

\- If the enemy wasn’t detected, the map would only show our own location, leading the model to get stuck in a Markov chain loop, causing it to spin in place or remain stationary.

\- Even when the enemy was detected, the model struggled to learn pursuit strategies, as pursuit and damage were not directly related; shooting bullets acted as an intermediary.



Later, we tried Policy Gradient and finally adopted Actor-Critic. We also abandoned the idea of using reinforcement learning for everything, instead reusing the existing handcrafted strategies from the official framework. Only the pursuit behavior, which was hard to code manually, was implemented using reinforcement learning. Additionally, we added extra safeguards to limit the pursuit area to avoid wall collisions. This hybrid battle strategy worked well on the real robot. The handcrafted global strategy controlled patrolling, pursuing, retreating, reloading, and triggering buffs, while reinforcement learning handled the pursuit. We used LiDAR scan signals as the state, movement as the action, and health advantage as the reward, successfully implementing the pursuit function. Next, we discuss techniques for speeding up convergence.



**Input State**  

The idea to use LiDAR scan signals as the state was inspired by ViZDoom, a competition where reinforcement learning is used in FPS games. Since reinforcement learning can be achieved using just a first-person view, the one-dimensional LiDAR signal contains enough information. The advantages of using LiDAR signals as the state are:

\- Reduced input size, as LiDAR signals are one-dimensional, while global maps are two-dimensional.

\- LiDAR signals are directly available, whereas global maps require SLAM mapping.

\- LiDAR signals are more direct, with each signal value indicating the distance to the nearest obstacle at a specific angle.



Additionally, to include information about the enemy, we added another array of the same size, representing the type of obstacle at each angle. If the obstacle is an enemy vehicle, it is marked as 1; otherwise, it is 0. Thus, the input state consists of two arrays: one for the LiDAR distance signals, and another for the type of obstacle at each corresponding angle.



**Actor-Critic**  

The core neural network model is quite simple, as shown in the figure:

The input is first expanded to 1024 dimensions, then reduced to a 256-dimensional vector, which serves as the high-level feature representation. From this vector, three dimensions are output for translational movement (left, forward, and right), three dimensions for rotational movement (left turn, no turn, and right turn), and the Critic estimates the value. The Actor and Critic share the first few layers to accelerate convergence by sharing information.



**Incorporating Priors**  

Although the model can converge, it does so slowly. Adding priors can help it converge faster.



First, consider the turning actions. Turning doesn’t change position, so there’s no need for obstacle avoidance, but it should aim towards the enemy because the auto-aiming has a limited field of view. The relevant input state is the second array, representing the type of obstacles. If an enemy is on the left, it should turn left; if on the right, it should turn right; and if in front, it should not turn. The prior is:



$$

\mathrm{p}_{\mathrm{r}}=\text {Softmax} \left(\left[\sum \text {enemy}_{\text{left}}, \sum \text {enemy}_{\text{front}}, \sum \text {enemy}_{\text{right}}\right]\right)

$$



where $enemy_{\text{direction}} = \left[t_1, t_2, \cdots, t_n\right], t_i \in \{0, 1\}, i \in direction$



The final action is: $a_{\mathrm{r}} = \mathrm{O}_{\mathrm{T}} + \mathrm{p}_{\mathrm{r}}$, where $\mathrm{O}_{\mathrm{T}}$ is the Actor’s rotational output. This way, the Actor only needs to learn the residual from the prior action, speeding up convergence.



Similarly, for movement, distance signals must be considered to avoid walls, moving in the direction of the greatest distance. The prior is:



$$

\begin{aligned}

\mathrm{p}_{\mathrm{m}}=\operatorname{Softmax}\left(\left[\sum \operatorname{distance_{\text{left}}}, \sum \operatorname{distance}_{\text{front}}, \sum \operatorname{distance}_{\text{right}}\right]\right)

\end{aligned}

$$



where $distance_{\text{direction}} = \left[d_1, d_2, \cdots, d_n\right], d_i \in R^+, i \in direction$



The final action is: $a_{\mathrm{m}} = O_{\mathrm{m}} + \mathrm{p}_{\mathrm{m}}$, where $O_{\mathrm{m}}$ is the Actor’s movement output. Again, the Actor only needs to learn the residual, leading to faster convergence.



**Pros and Cons Analysis**  

Compared to our initial proposal, this model is much simpler but effectively implements the pursuit function. The advantages are:

\- Simple model, easy to understand, fast training, quick response during real-robot testing.

\- Incorporating priors accelerates convergence.

\- Once the enemy is detected, it can flexibly execute the pursuit function.



The disadvantages are:

\- Priors may introduce bias, preventing convergence to a globally optimal strategy.

\- Limited functionality.

\- No memory capability.

\- Without constraints, it may crash into walls.

\- If no enemy is detected, it tends to spin in place, unable to patrol autonomously.



**Real-Robot Testing**  

Although the pursuit function was implemented in the simulation environment, there’s still a gap in deploying it on the real robot. The main challenges are:

\- How to integrate it with the official RoboRTS framework.

\- How to handle obstacle avoidance.



The official RoboRTS framework is based on ROS, so we wrote a Python ROS node to communicate LiDAR and aiming data to the model, and send actions back to the global decision model. The global strategy decides when to call this model based on its good pursuit performance. However, since the model’s obstacle avoidance is weak, we also wrote a protection mechanism using the raw LiDAR signals: when the robot gets too close to an obstacle in its movement direction, the movement command is blocked.



## Competition  

In our first official match, we unfortunately lost. Both of the opponent's robot couldn't move, and we still had one car working normally and causing damage. However, due to the software sync issue, the heat limit of the muzzle was not changed to the new rule value, resulting in more deduction of our own health. The second game was indeed not as strong as the opponent and we lost the game.



Despite the losses, we remained optimistic. It was our first competition, and completing the event safely was an achievement in itself. We also had valuable technical exchanges with teams from Jilin University, Zhejiang University, and UC Berkeley, discussing reinforcement learning approaches.



## Conclusion

In conclusion, reinforcement learning is powerful, but guiding the model’s convergence is a challenge, especially in real-world applications. By combining handcrafted strategies with reinforcement learning, we were able to create a hybrid battle strategy that performed well in real matches. Though our results weren’t perfect, participating in the RoboMaster competition was a unique experience, and we’re eager to improve for future challenges.