+++
title = 'Software'
date = 2026-02-16T11:27:34+05:30
draft = true
+++

I will spent time to describe the architecture and gist of all the elements, then move towards algorithms

Software stack
- What are requirements
- Architecture diagram and Algorithms
- The why
- The how
- ability to expand
- Webots Demo, real Demo


Software stack, development and deployment
  ROS2 Jazzy, ubuntu 24.04 , C++ 17, Python 3.12, Webots


Major requirements of the software stack --
- Modular and templated
- minimal middle ware dependence
- Compliance to AUTOSAR C++ standards
- Easily configurable
- Interpretable
- Good visualization and debugging

Architecture diagram


The code is made independent of the ros platform, or the simulation platform. The requirements from the simulator or the vehicle are - it's present location, and objects around it which can be obtained by gps module, or odometry or other fusion algorithms
For comparison Autoware software architecture is build on ROS2 backbone, all the data structures in use are ROS2 based. ROS2 is also build with focus on deployability unlike ROS1 which was only used for testing and development in industry, ROS2 is actually being deployed.

<Bit about ROS1 ROS2> time , timer_callbacks, why no master node,
    in ros2 why is the publisher and subscriber shared_ptr and not unique_ptr
    How does the timer_callback work

<Bit about Autoware>  

The core phylosophy is to have very minimal dependencies, thus the development is not dependent on external factors,
??
If internal modules are designed and maintained - testing and validation of the code is only incremental, otherwise whenever a new external package is updated it goes

unneccessory code and packages can be avoided - and have a stricter version control.
<Why another stack>  If we have to deploy only 1 module and with very less compute / This stack can serve as backbone for ADAS development and experimentation.

Classes - and who all uses these classes -- class hirarchy diagram, with general concepts of tracking the vehicle's position and sampling the map for A*.
Data Structure

All the algorithms implementation is based on classes and structured in a way that it can be reused.

## The Map
  The map can contain a lot of information.

  Picture is taken from Autoware website - You can see your vehicle, obstacles which is result of localization and perception and you see road, junction, predicted path of other vehicles these come from Map (predicted path is basically time based occupancy of positions in map) additionally you will have speed limits, sign interpretations (hump, pothole, construction etc...)


## Main data structure
  robot state and state_l
  The reason to build data structures can be 2 fold, simplify code by having reusable components inside data structure and reducing dependencies. Especially on ros.

  The main data that is going to be used extensively in multiple parts of the code is
  * current state of car (position, velocity , acceleration)
  * map (global path, Navigation)
  * planned locations for next n timesteps
  * actuation commands
  * obstacles around the car (bounding box, position, velocity, variance in measurements)

  Commonly for all the messages that are shared over various module its good to track timestamp. There are implicit buffer inside all the ros publishers and subscribers, or if your custom stack has message buffer, both way there is possibility you are reading from uncleared buffer - hence

  To check before actuation wether its updated or did we miss a message.

  Since this information is required in multiple places, it will be required to transform the message into other formats / class types its better to keep this functionality in the data class struct itself because of re-usability. Now these conversions can be overloaded to accommodate the variations in data types and message types since same data could be coming from various sources.
  Also a careful usage of unique_ptr can help control accidental overwriting and avoid unwanted conversions.

## A-star Local goal generation
  What is the type of algorithm that we need -- purpose - and why does A* fit in?
  The purpose of having a sampling based algorithm is obstacle avoidance. Input is the map and obstacle locations -

  If map is large, for each A* iteration there is only requirement to load the way points incrementally that is discard the previous point and add next few points (can be again decided based on varying distance according to planners last steps velocity).

  A* implementation sudo code - heuristic, cost function and elimination. Complexity analysis
  Input - these are local goals obtained from sampling the map - each point is located based on distance gap? We sample for 6 timesteps and MPC runs with 4 step receding horizon. control module (MPC) takes these local goals as reference and generates the control command

  Astar is sampling based algorithm, hence how  efficiently we sample the states determine majority of the code.
  Key things in Astar planner are how to expand the state space, cost function, definition of a node -

  While running sampling based algorithms choice of coordinates make a huge difference. For a car the widely used is one is 2D x-y coordinates but there is another choice of making the roads center-line as one axis and perpendicular to it as another axis. Now we should focus on major computations that we do while sampling, which are tracking the distance along the center line, cross track error, measure distance to goal, measure progress towards local goal, predicting next state etc... As you might have noticed many of these values are directly present in the second choice of coordinates. This significantly improves the local goal planner computation. You should also keep track of 2Dx-y coordinate for easily conversion conversion and use the better of both. Also think of scenario in curved roads where triangle based cross track error from straight line assumption might be wrong.

### Definition of node
  * node id
  * parent nodes id
  * goal state information
  * list of child nodes

### Planner class
  Class objects are
  * node list
  * hash map to store visited nodes
  * multimap - fringe  and closed list
  * goal lane marker

  heuristic - while choosing heuristic you have to keep in mind estimate shouldn't be more than the actual cost.
  * Distance to the global goal (Euclidean / manhattan)
  * curve synthesis based path length
  Unless you search space is very broad or has some special feature cost (which might serve as a good heuristic) simple distance based heuristics work well.

  State expansion requires you to choose appropriate vehicle model - this can be ackerman steering based, bazier curves, dubin curve or reed-sheep - it should be compactible with the dynamics of the car. If using steering models like ackerman then there will be parameter dependence based on which type of vehicle are you deployed model. While curve generation methods can be used for wider range of vehicle parameters a accurate model would provide better results as higher search space is explored. In current case this is of not much significant difference since this is a reference to MPC controller generating control commands using higher fidelity model.

  Bezier curve - 4 point - tangential to the start and end -- to maintain continuity

  Heuristic is distance to the final goal.
  Cost function is a significant factor determining the quality of the A* output path. And there is no standards for the same and based on it's trial and error. popular choices of the cost are
  * cross track error from central lane marker
  * control effort error (change in acceleration, steering)
  * vehicle state based cost (accelerations, steering value, heading, )
  * potential function based error
  * uncertainty based cost

  If the lower level controller is pure pursuit or stanly controller then it is important that the planner should consider the actuator limits and corresponding cost function, in which case state expansion should be a higher fidelity model.

  If you only consider Cross track error, then in case ob obstacle avoidance, taking a turn you might have very high lateral acceleration and very high steering effort requirement. So you can add these into cost function or define bounds for these parameters, but it might happen that because of this bound in some edge cases algorithm will fail to find solution.

  While using multiple of them with weights these weight become tuning parameters to achive the desired results. And if desired results are not well defined you are prone to failure in some edge case which couldn't be imagined while making the algorithm. So depending on the situation and use cases choice of error function must be wisely chosen.

  In our implementation its only cross track error, primarily because maximum velocity is low, based on the testing results we might change it later.

  Obstacle avoidance - The obstacles are represented as boxes in 2D plane and any node passing through box is invalidated.


## MPC - reference states, actuator limits

  What should be the prediction horizon -- how good is the vehicle model
  - Prediction horizon is generally considered in time
  - A good starting point is the amount of time the car takes to reach full stop at current speed.
  - Prediction horizon in terms of number of steps can be fixed, or based on delta time ie. 5 steps per seconds or 5 steps for full horizon.
  - In both case MPC model's linearization should hold good. Slip caused by lateral acceleration shouldn't be more than 3-5 deg and steering rate should be less than 50deg/s.


  Time step you should use for MPC -- does it need to be exactly same as computation cycle? Faster/slower
    - The output of the MPC is supposed to go to the actuator directly
    - Ramp rate constraints are limits on delta between consecutive states (now this is dependent on time delta)
    -
    What is the advantage - lower processing
    - Actuator output needs to be post processed to interpolate between current control command and next step - to get the intermediate step (since MPC)
    - We would violate the ramp rate constrains at post processing

  Bicycle model - state space variables - constraints -
    - Good article to understand bicycle model - ([LINK](https://thomasfermi.github.io/Algorithms-for-Automated-Driving/Control/BicycleModel.html))
    -
  OSQP -- error diagnosis -- primal dual --


## velocity profile

  For generating velocity profile - there needs to be a optimization, now this can have multiple objectives - most likely one is to minimize the jerk or lateral acceleration or maximize user comfort (which is basically limits on accelerations), or you could have a profile with continuous snap profile or multiple of these objectives. These can be complete numerical optimization (LQR or MPC) or polynomial based, or sampling based like A*.

  If you think about real world scenario, velocity profile gets affected by lot of external parameters which are not related to the host vehicle. For example traffic light, crossings, speed limits, obstacle, terrain etc... Now how do we incorporate all these into the velocity profile generation?

  Now when i think of it, it's not velocity profile generators job - all these external parameters contribute to the constrains on the optimizer, and all of these shall be treated as external factors ie. either from Map or perception module there should be a record of what is the speed limits for positions in planning horizon - it can be 0 when there is traffic light, map can update it when light goes green, same goes for speed limits etc...

  So to generate a velocity profile most of the work is done by map generation / perception module.

  In my case I used trajectory generation module from TUM -- 


How is logging and documentation done. Google logger and Doxygen with architecture diagrams.

Any perception module can be inserted - infact the code can be split into 2 major parts -- perception and motion planning and control -- we havent implemented a full perception module yet,
Idea is to have many control / motion planning algorithms - and this code serve as a template to implement

Decomposition - Re-usability and manageable / common functionality (map points - lane marker)

In this article we will talk about the software part
For autonomous driving there are major 3 modules - perception, localization, Motion planning and controls??
For initial integration the scope is to run the vehicle in closed look control with accurate localization without perception. We will be doing localization based on IMU and GPS data, the GPS has RTK corrections and can give positional accuracy of +-1.5cm. Hence this would be good test case to perfect the motion planning and control module before integration other modules.

Its always a good strategy to make the software development modular and test and integrate each modules independently. This modularity should be choosen such a way that the inter dependence is minimal and there are reasonable ways to test and validate each module. Its always good practice to develop test cases along the code writing. But generally the author of the code is too focused on algorithm and complexity management and developing the hardware interface test requires separate effort. Nevertheless thinking about test cases will promt to add correct debugging messages and interpretable logs.

I will first discuss the architecture of the Motion planning and control module.

Arch diagram --
