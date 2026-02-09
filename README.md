# har.github.io
Showcasing my builds

Play book to convert manual cars to drive by wire.

About the vehicle
    - general specs
    - power electronics part - motor + controller -- mainly how the control algorithm works -- time delays involved -- expected system behaviour
    - Parameters involved in modelling  (regen vs Plunging) (full operating region) (apreciate FOC slightly - vs a variable freq drive)
    - Modelling experiments and results - example

Steering setup
    - Design
    - Actuator sizing - selected actuator and specs
    - Expected torque curve
    - CAD
    - Communication with actuator -
    - System architecture diag

Steering takeover Design
    - How is this different from power steering sizing
    - Approach to the problem - ML model selection and reasoning
    - Data cleaning, data creation
    - Important features, varience bias check,  expected noise, - fine tuning - BO -
    - Data collection, train and test data -- performance on test data
    - Real life scenario

Vehicle drive by wire setup
    - Interfaces
    - CANOpen
    - CANFD device - waveshare device -- clean driver code
    - Control flow

Software stack
    - What are requirements
    - architecture diagram and Algorithms
    - The why
    - The how
    - ability to expand
    - Webots Demo, real life Demo

Hardware stack
    -
    -

How to do end-to-end autonomy
    -


Objectives / goal
The vehicle we have in iit madras - autonomous systems lab is a normal electric vehicle platform

The vehicle is powered by a low voltage battery Lithium ion battery pack, and has 3phase induction motor
We have access to motor controller configuration. The motor controller uses FOC control algorithm.
This algorithm is briefly mentioned mainly to highlight parameter choices and their impacts - and vehicle limitations.  

( all the edge cases - and limits for the  -  torque control)
First we will delve into motor torque rpm curve
    - Firs differentiation is between regen braking and plunging - these are 2 regions of operation
      in plunging the air gap flux created by the 3phase current in the stator coil will be oposite to the rotation of rotor
        - The braking energy is dicipated as heat in the machine itself -
        -
      in regen the flux created by 3phase current will be ahead of stator coil - creating negative slip
        - in regen the energy can be pushed back to the source (if allowed) -- this is how the turbines generate energy

      Which one to use
      - The vehicle can be operated in majorly 2 ways
        Regen active - When pedal is at 0 position - nominally regenerative braking is applied according to the braking power map
            This would mean that majority of decceleration will be handled by regen and we dont need to initiate manual braking very often.
            The amount of regenerative braking is usually pre-set or is a constant lookup map wrt to rpm -- lower torque values for higher rpm - to limit the amount of regenerative current pushed back to the battery.

        Plunging brake - Treating
        Manual brakes can be used to assist the vehicle's regen/plunging limits - but electrical braking have faster response
        and only delay is due to communication and added brake smoothing delay (brake ramp rate limit etc ... ) eg. brake rate, brake release rate ...
        Hence its good practice to combine both of them to achieve faster response.
        While regenerative braking doesn't allow dynamic control of rate of braking - this is possible with plunging brake, more or less its same as normal braking behaviour


      Vehicle control comes in different modes -- speed control or torque control
        - Speed mode is only PID loop on top of torque control - all FOC control algorithms track torque references
        - This torque reference goes through multiple filter before being finalized for FOC reference.
        -

      Power limitation of the induction motor
      - variable frequency drive and
      - Understanding what control algorithm the motor controller uses helps you identify what are limits of your vehicle easier.

      If using a PMSM motor
      - The

  Vehicle specs - Vehicle control loop design -- and parameter lists

FOC model - used by the motor controller  -- block diag https://in.mathworks.com/help/mcb/gs/implement-motor-speed-control-by-using-field-oriented-control-foc.html
    - The motor controller has set of checks and parameters -- output of which is a reference torque profile
    - Motor controller had FOC in torque control mode

    The throttle pedal / can throttle command is giving reference for required torque
    Parameters defining user comfort in manual driving  -- but not so in control by wire system
      Accel rate,  accel release rate -- time delta to ramp up/ down the torque
      Brake rate, brake release rate  --
      Neutral braking,  Neutral braking taper speed --

    Regenerative braking related Parameters
      Drive current limit  -- kept smaller than the brake current limit
      Regen current limit  -- regenerative braking is involuntary -- applied when throttle is release
      Brake current limit -- Active braking on command
      Power limit map -- Steady state working rpm and change power as a % of max rated power

    Saftey parameters
      Under votlage, over voltage --  Initiates emergency braking
      Current limits for drive and regen -- Caps the throttle request

    CANOpen
      CANOpen interlock
      CANOpen init
      Heart beats, baud rate, timeout



The vehicle is made fully drive by wire using an additional actuator to control the steering and can communication with the motor controller using CANOpen protocol.
The steering uses CubeMars AK80-8 motor for actuation , which is a dual encoder brushless dc motor , with rated torque of 10Nm, operated in speed mode using, normal CAN

The can receiver for CubeMars, vehicle and Radar is a CANFD device, were buffer is stored and feedback messages can be requested. Device driver is written in cpp.
There are higher level device specific driver codes, and ros packages to communicate steering, acceleration and regenerative braking requests.

The steering column was kept intact and mounts were designed to keep optimal loading on the steering column. This was a retrofit gear assembly designed and manufactured using laser cutting, metal bushing, machining, welding and 3D printing.
  During testing it was found that the Cubemars motor has unexpect high jitter in position control mode
  Hence we decided to use position control with PID on top of velocity mode control on motor

  Cubemars parameters - baud rate, feedback freq , current limits, rpm limits, can timeout ()   -ch

  PID response graphs are as shown -- the controller parameters are --
  -- Code for the arduino based controller - LINK
  -- Code for the CANFD device based controller - LINK


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
