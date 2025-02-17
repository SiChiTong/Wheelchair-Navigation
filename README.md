# Wheelchair-Navigation

A smart wheelchair improves the quality of life for older adults by supporting their mobility independence. Some
critical maneuvering tasks, like table docking and doorway passage, can be challenging for older adults in wheelchairs,
especially those with additional impairment of cognition, perception or fine motor skills. Supporting such functions in
a shared manner with robot control seems to be an ideal solution. Considering this, we propose to augment smart
wheelchair perception with the capability to identify potential docking locations in indoor scenes.

[ApproachFinder-CV](https://github.com/ShivamThukral/ApproachFinder-CV) is a computer vision pipeline that detects safe docking poses and estimates their desirability weight based on
hand-selected geometric relationships and visibility. Although robust, this pipeline is computationally intensive. We
leverage this vision pipeline to generate ground truth labels used to train an end-to-end differentiable neural net that
is 15x faster.

[ApproachFinder-NN](https://github.com/ShivamThukral/ApproachFinder-NN) is a point-based method that draws motivation from Hough voting and uses deep point
cloud features to vote for potential docking locations. Both approaches rely on just geometric information, making them
invariant to image distortions. A large-scale indoor object detection dataset, SUN RGB-D, is used to design, train and
evaluate the two pipelines.


Potential docking locations are encoded as a 3D temporal desirability cost map that can be integrated into any real-time
path planner. As a proof of concept, we use a model predictive controller that consumes this 3D costmap with efficiently
designed task-driven cost functions to share human intent. This [wheelchair navigation](https://github.com/ShivamThukral/Wheelchair-Navigation) controller outputs a nominal path that is safe,
goal-oriented and jerk-free for wheelchair navigation.

# Brief Overview

We used a MPC based controller that works by interleaving optimization and execution. This is a sampling based, derivative
free approach which was applied to aggressive autonomous driving. At each time step, controller estimates the optimal control
sequence from the previous time step and uses importance sampling to generate thousands of new sequences of control
inputs. These sequences are propagated forward in time using system dynamics, and each trajectory is evaluated according
to a set cost functions. The estimate of the optimal control sequence is then updated with a cost-weighted average over
the sampled trajectories.

Our work uses this Autorally platform with handcrafted cost functions that are meticulously designed to navigate in a
room with obstacle avoidance and proceeding according to the user’s intent. Specifically, we fuse desirability cost map,
obstacle map and user joystick commands to produce safe and jerk free paths.

# Installation Instructions:

1. Installation can be found [here](https://github.com/AutoRally/autorally#1-install-prerequisites). Please follow steps
   all the steps to compile this code.
2. Clone or Fork this Repository in a [catkin_workspace](http://wiki.ros.org/catkin/workspaces)
3. Install AutoRally ROS Dependencies:<br>
   Within the catkin workspace folder, run this command to install the packages this project depends on.
   `rosdep install --from-path src --ignore-src -y`
4. Compilation:
    1. Check Eigen version: `pkg-config --modversion eigen3`. If version < 3.3.5, upgrade Eigen by following "Method 2"
       within the included [INSTALL](https://github.com/eigenteam/eigen-git-mirror/blob/master/INSTALL) file.
    2. Run `catkin_make` from the workspace folder. Due to the additional requirement of ROS's distributed launch
       system, you must run
       `source src/autorally/autorally_util/setupEnvLocal.sh`



# Simulation Demo:

1. Launch the simulation environment: For in-depth details about simulation environment, please
   refer [here](https://github.com/ShivamThukral/ApproachFinder-CV#demo-in-simulation).
    1. You need to run the following commands from ApproachFinder-CV package: <br>
       `roslaunch my_planning simulation.launch`
2. Launch the nodes to communicate between MPPI controller and simulation enviromment.
    1. This involves: user joystick talker, robot ground state publisher, robot command publisher and a localisation
       node with real-time obstacle marking and clearing. For details about these nodes please refer
       the `shared_control.launch`<br>
       `roslaunch my_planning shared_control.launch`
3. Launch the MPPI controller node <br>
   `roslaunch autorally_control standalone_path_integral_bf.launch`
4. Inputs: 
   1. Use R2 shoulder button to enable user joystick commands and left axis stick to provide user intent.
   2. Use R1 to enable robot. This is pass the MPPI controller to robots drifferential drive motor.
   

If everything loads successfully, then you should be able to see:

- A Gazebo office environment with a robot spawned at (4,4)
- In RVIZ you can see votenet detected object, docking locations and desirability costmaps w.r.t to each heading bin.
- An occupancy grid map where the robot is localised and it updates its obstacle believe when the environment changes.
- You can also see nominal path generated by the controller. Once you enable the robot, it will start moving according
  to the given inputs.

Note: This demo requires 32GB of RAM for run successfully.

# Results
![](results/video1.gif)

![](results/video2.gif)

# How to run shared-controller with your environment?

Supply floor plan to the planner:
Autorally expects an initial obstalce map which it uses to initialise the controller parameters. A value of 0 defines
the center of the track, and a value of 1 indicates off the track. The cost map is a simple text file with the following
format:

- first entry: minimum x value
- second entry: maximum x value
- third entry: minimum y value
- fourth entry: maximum y value
- fifth entry: resolution of the grid.

The total width of the grid is (maximum x - minimum x)*resolution and similarly for the height. all remaining entries:
the grid values in row-major order. Some sample examples are kept in `path_integral/params/maps`. The first task is to
create such obstacle map for your environemnt and supply to the controller.

Following steps outline this process:

- For this work, we have used laser-based SLAM to build the 2d occupancy grid map. For this, we
  use [slam_gmapping](http://wiki.ros.org/gmapping).
- Launch the slam_gmapping and build a static map using this node.
    - `roslaunch my_planning slam_gampping` (you might have to play around with launch file params).
- Once the map is build, store it as a .txt file further processing.
    - `roslaunch my_planning convert_costmap_node` (please change the filename according to your convenience).
- Convert this into autorally readable format using this command:
    - `python track_converter.py -i <map_file.txt> -d map_image.jpg -o map.npz`
- Finally, copy the generated map.npz file to the  '/autorally_control/src/path_integral/params/maps/'
    - Replace the `map_path` param in `standalone_path_integral_bf.launch` accordingly.

Remaining steps are same is followed in Demo Simulation.