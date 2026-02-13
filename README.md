# har.github.io
Showcasing my builds
    ------------------------------------- V2 ---------------------------------



How to do end-to-end autonomy
    -

To do
- Steering motor PID response graph - real time video and anotation - User feedback
- Steering ML model -- interpretation of the parameters
- IMU noise graph
- vehicle feedback quality
- Modeling part and software part
- vehicle modelling -  show how well the model predicts the trajectory -- create a sequence of input command - check what vehicle does and what does the model do
- Radar -- false detection -- and after rcs filter



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


  +++
  title = 'Software and modeling'
  date = 2026-02-13T00:47:17+05:30
  draft = true
  +++



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
  -

  Architecture diagram
