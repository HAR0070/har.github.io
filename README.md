# har.github.io
Showcasing my builds
    ------------------------------------- V2 ---------------------------------

    Modeling experiments and results - example
    - Sensor responses - IMU frequency response - show the IMU vibration
      what all does the sensor do - in terms of fusion - RTK gps,  - acceleration has lot of vibration but velocity is clean and good. Comparison with vehicle odometry value
    - Vehicle modeling (accel and brake)- in torque mode --  time delay, ramp rates -- show how well the model predicts the trajectory -- create a sequence of input command - check what vehicle does and what does the model do --

    - Steering modeling - PID response for step input while loaded -   sequence of input -- what steering does and what model does

    Software stack
        - What are requirements
        - architecture diagram and Algorithms
        - The why
        - The how
        - ability to expand
        - Webots Demo, real Demo

How to do end-to-end autonomy
    -

To do
- Steering motor PID response graph
- IMU noise graph
- modelling part and software part

Objectives / goal
The vehicle we have in iit madras - autonomous systems lab is a normal electric vehicle platform

The vehicle is powered by a low voltage battery Lithium ion battery pack, and has 3phase induction motor
We have access to motor controller configuration. The motor controller uses FOC control algorithm.
This algorithm is briefly mentioned mainly to highlight parameter choices and their impacts - and vehicle limitations.  

( all the edge cases - and limits for the  -  torque control)
First we will delve into motor torque rpm curve


The vehicle is made fully drive by wire using an additional actuator to control the steering and can communication with the motor controller using CANOpen protocol.


The can receiver for CubeMars, vehicle and Radar is a CANFD device, were buffer is stored and feedback messages can be requested. Device driver is written in cpp.
There are higher level device specific driver codes, and ros packages to communicate steering, acceleration and regenerative braking requests.

  Cubemars parameters - baud rate, feedback freq , current limits, rpm limits, can timeout ()   -ch




The IMU used is fixposition visual inertial navigation unit with RTK corrections from Survey of India CORS portal- with fusion working at 200Hz, with drift of angular drift of 0.1 deg / hour. Velocities used are bias corrected and in ENU frame,
The IMU has a internal fusion engine running,  it starts with only RTK corrections are provided, the IMU considers  --- elements -- for fusion

The final data quality is ensured:
  - Ensured that while not in motion IMU doesnt have non-explanable frequency response
  - While moving -- the frequency response is evaluated and found to be well with  requirements ( how u define it? )
  - Effect of vehicle model on IMU stats
  - Wheel odometry stats

The raw acceleration inputs still have high vibration - hence acceleration is derived from fused velocity with additional jerk filter


Vehicle modelling and testing

The Velocities obtained from the IMU are more reliable than the vehicle odometry -- a comparison is provided here
  -- image -- and evaluation metric to be used --

The testing methods used for data collection for modelling vehicle is described here
  First the vehicle is made fully joystick controllable
  IMU is intialized
  Vehicle feedback is obtained at 20Hz - by reading TPDO feedback from controller
  For longitudanal testing scenario steering control is disabled and handled by driver

Testing scenarios are because -
  - Its observed that the controller delay varies with the current RPM of the vehicle --
    image -- plotting the delay to rpm

  - How is the delay measured --
  The delay for the model involves time from issueing the command - in our case ( time or publishing the command as rostopics)
  to the time where the vehicle response is observed - IMU showing corresponding increase in velocity/acceleration or change in vehicle odometry reading

  The accuracy will be upto the time period of sampling


What all to record and why

Scenario 1
  Vehicle is driven in straight road - full throttle command until max velocity is achived
  Then full brake command

  To understand the change in response delay during motion ( which will be different from while starting stationary)
Scenario 2
  Vehicle is driven to 20Kmph - then allowed to free wheel, accelerated again, repeat 2 -3 times. Do 3 runs

  To understand regenerative braking strength
Scenario 3
  Vehicle is driven to 20Kmph - then allowed to free wheel, accelerated again, repeat 2 -3 times. Do 3 runs
  This time note that depending on your rpm range - the regenerative braking will change (based on the mapping) - hence its recomended to keep regen constant till nominal rpm of the vehicle - and in each iteration change it and test again -- make sure while accelerating not to cross the nominal rpm.
  Also record the deceleration rates during regenerative braking

Model fitting
  There are multiple ways to do this, learning approaches and classical methods, i would only describe 1 example for each
  For classical methods - start with first order model and then if better performance is required increase the model order
  This would mean - identifying time delay, time constant and gain constant
  Also have to identify the rate of decay -- comes from rolling resistance and other drag forces on the vehicle

  If the vehicles performance is limited by ramp rates of acceleration / brake - its better to use the discrete form equation with ramp rate limitation directly
  This would limit your ability to complex theoritically analysis of the model.

  For learning models - the choice of independent and depended variables important, cleaning the data and collection good data

Steering modelling and testing
  Data collection
  Clear patterns
  Related work -- power steering / steering assist  - also the observer based model
  Machine learning model - bias varience / BO optimization /  feature importances -> explanation / validation set data results / real time video and anotation.
  Edge cases - low speed is modelled differently -- by taking measurements from - and fitting wrt steering angle
  User feedback

Software stack, development and deployment
  ROS2 Jazzy, ubuntu 24.04 , C++ 17, Python 3.12, Webots
