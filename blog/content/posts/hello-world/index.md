
---
title: "Step1 to Autonomy"
date: 2026-02-10
draft: false
---

## Whats in here
The step 1 is to understand the platform you are working with, this article is for those who are trying to convert a normal vehicle to autonomous vehicle for research/testing purposes. I would be sharing ideas on how I approached the problem and alternatives with good references I found along the way.

This work is majorly on translating hardware system to seamlessly interface with software, specifically in context of electric cars.
Starting with basic understanding of the electric vehicle powertrain to implementing software backbone enabling the vehicle to track reference trajectory, with provision for safety override by the user.

This is divided into sections
  * Vehicle specifications and powertrain
  * Steering actuator and takeover strategy
  * Modeling and vehicle drive by wire
  * Software stack and data structures

## The Vehicle

About the vehicle
    * General specs
        -- make it a table
        Motor type                          Induction motor
        Mass of vehicle                     600Kg
        Motor controller                    FOC controller, switching freq-  , power rating , link
        Access we have                      To all the configurable parameters in motor controller
        Max speed, acceleration and brake   30Kmph ,     
        Battery type and voltage            LFP battery, 50V battery,
        Power curve of motor                

### Power curve and FOC
Objective is to understand what to expect from the powertrain/drivetrain of the vehicle.
On a high level how things work, how does it affect the modeling - mainly sources of delay and actuation limits.

Major options for control algorithms are DTC, FOC and MPC and their variants.

  DTC - direct torque control methods implements a hysteresis controller using a lookup table to fetch the voltage vector for the inverter for maintaining torque and stator flux within specified limits. This works without sensors but having a speed sensor can ensure higher accuracy over full range of operation.
    - Control is fast ~2µs (since it doesn't require PWM) but drive will experience ripples due to parameter variations.

  FOC - Field oriented control methods translates 3phase machine into DC counterpart using special transforms (Clarke and Park) and controls torque and flux in DC frame using simple PID loops (Id and Iq current control). These transforms require accurate position estimate making a encoder necessary for the implementation.
    - Requires higher computation, control loop runs at switching frequency ~10-20KHz (~50-100µs).

  MPC - Model predictive controller is general framework which uses a mathematical model of plant to predict future behaviour and generates optimal control actions respecting constraints, for motor its motor's electrical model, switching states for inverter, system constraints. This requires much higher compute and accurate modeling and hence generally not used in applications like trains etc...

  Key takeaway is
  * Control loops are really fast, we consider it to be 0 time delay.
  * Any torque value can be achieved almost instantaneously without load. (within motors limits)
  * This can cause inertial systems to be very jerky - The commanded torque reference passes through ramp rate and various other limits. This induces the actuation delay and required smoothness for comfortable drive.
  * Hence modeling traction is identifying limits on reference Torque, if motor limits are known.  

Our vehicle uses FOC controller

{{< figure
  src="images/acim_foc.png"
  alt="FOC controller for Induction machine"
  caption="FOC control is industry standard, Our vehicle also uses FOC controller, "
>}}

Power curve of induction motor
{{< figure
  src="images/IM_power_characteristics.png"
  alt="Induction machine power curve"
  caption="Against the common perspective a FOC control can achieve flat torque curve"
>}}

The Loop Time: In traction inverters, the FOC loop typically runs at 10 kHz to 20 kHz (every 50µs to 100µs).

  Every single PWM cycle (e.g., every 50 microseconds), the microcontroller must:
  * Sample Analog Currents (ADC).
  * Read the Encoder/Resolver position.
  * Compute Sin() and Cos() for that angle.
  * Perform Matrix Rotations (Clarke/Park).
  * Run PID algorithms for Flux and Torque.
  * Perform Inverse Rotations.
  * Update the PWM registers.

  For a indepth understanding of how AC motors work- read Chapter 2 - Modern power electronics and AC drives by Bimal K Bose.
  Also describes control methods in chapter 8 and 9

### Regen braking vs Braking using motor
First differentiation is between regen braking and plunging braking - these are 2 regions of operation for the motor ref to fig.

Normal operation of motor is when the stator generates air gap flux (refered as just flux), through current switching rotates in space sinusoidally now if the rotor is in perfect sync with this flux, there will be no force (when motor doesn't have load) but as you put a load, it restricts rotor motion and rotor "slips" from perfect sync - now the more the slip - higher the induction and more the force to bring the rotor back to sync. The rotor will stabilize when the load and magnetic force matches.

Now an obvious way to brake is rotating the air gap flux backward - both the load and magnetic force are in same direction and hence motor stops.
  - Now torque and rpm are opposite, also you are providing energy to generate flux, all this energy is dissipated as heat in the machine itself and is quite stressful for the machine if done for long.

Now instead of going all the way to rotating opposite to rotor - as you might have guessed keeping flux slightly behind the rotor will also cause braking, but now we have great advantage, remember flux is always a mutual phenomenon between 2 coils hence here its like the rotor is pulling the stator (same normal operation but interchange stator and rotor), that means rotor is providing the input power ie. we are converting vehicles KE to electrical energy and this energy can be fed back into battery - a win win.
Its very easy to implement you only need to bother how to feed back power to battery.

{{< figure
  src="images/regen_vs_braking.png"
  alt="Induction machine power curve"
  caption="3 operation regions for a induction motor, notice the slip axis values"
>}}


#### Which one to use
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

Usually this is not in users control and vehicles come with torque control pedal, hence we also went ahead with torque control

### Motor controller smoothing parameters
    Vehicle control comes in different modes -- speed control or torque control
      - Speed mode is only PID loop on top of torque control - all FOC control algorithms track torque references
      - This torque reference goes through multiple filter before being finalized for FOC reference.
      -

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

    Safety parameters
    Under voltage, over voltage --  Initiates emergency braking
    Current limits for drive and regen -- Caps the throttle request

    CANOpen
    CANOpen interlock
    CANOpen init
    Heart beats, baud rate, timeout


## Steering setup
{{< figure
 src="images/steering_motor_pos.png"
 alt="Steering actuator"
 caption="A DC brushless motor is attached to the steering, converting manual steering to steer by wire"
>}}

-Actuator sizing
  This part involves identifying what is the load scenario for steering actuator
  This can be done in multiple ways, but essentially measure the torque required throughout the profile isn't required.

  rather its about identifying the outer bounds of steering torque limits, through literature for track vehicles, steering torque would be maximum ~3Nm
  given we run only in road at low speeds - the same is taken as the limit in our case.

- Expected torque curve - There are 2 UV joints involved in the steering, placed perpendicular to each other, the worm gear ratio for the steering is not known.

- We had AK80-8 CubeMars motor to work with, and it satisfied torque and rpm requirements. This is a brushless dc motor with dual encoder and can save its position within 360 deg of absolute 0. But the steering rotates more than 720 degree to each side in our case hence we have added a absolute position encoder(+-60 deg) with a gear ration of 12.
- Communication with the motor is using normal can, the motor has multiple modes of operation position, speed, torque etc...
- During testing it was found that the motor has unpredictable high jitter in position control mode, hence position control with PID on top of velocity mode control was designed.

PID response graphs are as shown -- the controller parameters are --
-- Code for the arduino based controller - LINK
-- Code for the CANFD device based controller - LINK

- Design - It was decided to have a simple cad as steering column cad and actual system didn't match.
First design choices involved choosing from belt drive or gears,
Belt drive
  - Adding a tensioner pully would give leway from alignment issues
  - But higher number of components
Gear drive
  - Fewer components
  - Need to maintain optimal load to keep the meshing tight

I choose gear drive, with 1:1 gear ratio because our actuator could give more torque than required.
Mounts were designed to keep optimal radial loading on the gears. This was a retrofit gear assembly designed and manufactured using laser cutting, metal bushing, machining, welding and 3D printing.

Image of the CAD --

Note - While working with steering you might suddenly notice its much shorter that at start. The steering rod have a varying length design, they are 2 concentric rods within a cover with a interference fit bearing at start and linear slide bearings at end.

## Steering takeover Design
Objective : First reaction of the users while encountering a misbehaviour by the vehicle is to grab onto the steering, and hence we want to device a way to enable taking over the vehicle control using the steering.

Given the problem there are majorly 2 methods to go about this

Giving the user control to steering with some load acting against them.
- LINK (https://pmc.ncbi.nlm.nih.gov/articles/PMC10451058/pdf/10.1177_0036850420950138.pdf) - As described here create a model of the steering system using the inertial parameters and treat the self aligning torque as disturbance and device a observer using which you can track the setpoint. And since the input torque is what is controller user can give external torque to drive the steering to his liking - given the tracking error reduces - but here there is no way to detect a user input since every external torque is a disturbance, and moreover driver has to apply much higher torque than normal driving. Similar works are - LINK (https://skoge.folk.ntnu.no/prost/proceedings/acc04/Papers/0378_ThA05.4.pdf)

Giving the user the whole vehicles control as he applies some torque on steering.
- This aproach is to model the self aligning torque and add the inertia and damping components to identify the total torque required to be applied by the actuator - here the difficulty is - self alignment torque is a function of tire parameter, terrain and vehicle mass and lot of coefficients ...

Add the eqn here --

Since we are operating at low speeds (25Kmph max) which makes linear approximations clearly valid, this is much less complicated.
But still on trying to curve fit the data to the model, following difficulty were faced:
  * For curve fitting - we first need to identify which is the major component of the steering torque, inertial , damping, self aligning, or frictional.
  * Identifying how the steering inertia scales with the mass of vehicle is another challenge.
  * Backlash in the steering rack and pinion.
  * Presence of 2 UV joints in the steering rod.
  * Least square curve fitting is very prone to outliers

Future work will include more effort into model based approach

Approach to the problem - ML model selection and reasoning
The problem is solved as a unsupervised one, there is no labeling provided for initiation of the takeover and end of it. For this I trained the model to predict the steering torque from data (imu and steering feedback). And if the required torque is higher than the predicted value by 1.5 times the local standard deviation continuously for few timesteps then driver takeover is initiated.

tree based model is used - Xgb boost

- Data cleaning, testing and data creation
- Expected torque curve - There are 2 UV joints involved in the steering, placed perpendicular to each other, the worm gear ratio for the steering is not known.s
  - What all tests were conducted --
  - Yaw cleaning, data segmentation wrt speed,

- Important features, varience bias check,  expected noise, - fine tuning - BO -
  - . So
  - First we need to estimate the noise in the data, to build a expectation on the accuracy of the model. This was done in few ways
    -- an observer
    -- moving average / exponential average
    --

- Data collection, train and test data -- performance on test data
- Real life scenario

------------------------------------- V1 ---------------------------------

Modeling experiments and results - example
    - Sensor responses
    - Vehicle modeling
    - accel and brake
    - Steering modeling

Software stack
    - What are requirements
    - architecture diagram and Algorithms
    - The why
    - The how
    - ability to expand
    - Webots Demo, real Demo
