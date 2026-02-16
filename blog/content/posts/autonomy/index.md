---
title: "Step1 to Autonomy"
date: 2026-02-10
draft: false
math: true
---

## Whats in here
The step 1 is to understand the platform you are working with; this article is for those who are trying to convert a normal vehicle to an autonomous vehicle for research/testing purposes. I would be sharing ideas on how I approached the problem and alternatives with good references I found along the way.

This work is majorly on translating hardware systems to seamlessly interface with software, specifically in context of electric cars.
Starting with a basic understanding of the electric vehicle powertrain to implementing a software backbone enabling the vehicle to track a reference trajectory, with a provision for safety override by the user.

This is divided into sections
  * Vehicle specifications and powertrain
  * Steering actuator and takeover strategy
  * Modeling vehicle's drive by wire
  * Software stack and data structures

## The Vehicle

| General Specs |  |
|------|------|
| Motor type |  AC Induction motor, power rating 5kW |
| Mass of vehicle | 660 Kg |
| Motor controller  |  Curtis 1234SE, Advanced FOC controller |    
| Max speed, acceleration and braking | 30Kmph , 1m/s2 , 2.5m/s2 |  
| Battery type and voltage | LFP battery, 50V battery |  

### Power curve and FOC
Objective is to understand what to expect from the powertrain/drivetrain of the vehicle.
On a high level how things work, how does it affect the modeling - mainly sources of delay and actuation limits.

Major options for control algorithms are DTC, FOC and MPC and their variants.

  DTC - direct torque control methods implement a hysteresis controller using a lookup table to fetch the voltage vector for the inverter for maintaining torque and stator flux within specified limits. This works without sensors but having a speed sensor can ensure higher accuracy over the full range of operation.
    - Control is fast ~2µs (since it doesn't require PWM) but drive will experience ripples due to parameter variations.

  FOC - Field oriented control methods translate a 3-phase machine into its DC counterpart using special transforms (Clarke and Park) and controls torque and flux in DC frame using simple PID loops (Id and Iq current control). These transforms require accurate position estimate making an encoder necessary for the implementation.
    - Requires higher computation, control loop runs at switching frequency ~10-20KHz (~50-100µs).

  MPC - Model predictive controller is a general framework which uses a mathematical model of plant to predict future behaviour and generates optimal control actions respecting constraints, for motor its motor's electrical model, switching states for inverter, system constraints. This requires much higher compute and accurate modeling and hence generally not used in applications like trains etc...

  Key takeaway is
  * Control loops are really fast, we consider it to be 0 time delay.
  * Any torque value can be achieved almost instantaneously without load (within motor limits).
  * This can cause inertial systems to be very jerky - The commanded torque reference passes through ramp rate and various other limits. This induces the actuation delay and required smoothness for comfortable drive.
  * Hence modeling traction is identifying limits on reference torque, if motor limits are known.  

Our vehicle uses FOC controller

{{< figure
  src="images/acim_foc.png"
  alt="FOC controller for Induction machine"
  caption="FOC control is industry standard, Our vehicle also uses FOC controller. "
>}}

Power curve of induction motor
{{< figure
  src="images/IM_power_characteristics.png"
  alt="Induction machine power curve"
  caption="Against the common perspective a AC induction motor with FOC control can achieve flat torque curve. Figure is taken from - Modern power electronics and AC drives by Bimal K Bose"
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

  For an in-depth understanding of how AC motors work- read Chapter 2 - Modern power electronics and AC drives by Bimal K Bose.
  Also describes control methods in chapters 8 and 9

### Regen braking vs Braking using motor
First differentiation is between regen braking and plugging braking - these are 2 regions of operation for the motor ref to fig.

Normal operation of motor is when the stator generates air gap flux (refered as just flux), through current switching rotates in space sinusoidally now if the rotor is in perfect sync with this flux, there will be no force (when motor doesn't have load) but as you put a load, it restricts rotor motion and rotor "slips" from perfect sync - now the more the slip - higher the induction and more the force to bring the rotor back to sync. The rotor will stabilize when the load and magnetic force matches.

Now an obvious way to brake is rotating the air gap flux backward - both the load and magnetic force are in same direction and hence motor stops.
  - Now torque and rpm are opposite, also you are providing energy to generate flux, all this energy is dissipated as heat in the machine itself and is quite stressful for the machine if done for long.

Now instead of going all the way to rotating opposite to rotor - as you might have guessed keeping flux slightly behind the rotor will also cause braking, but now we have great advantage, remember flux is always a mutual phenomenon between 2 coils hence here it's like the rotor is pulling the stator (same normal operation but interchange stator and rotor), that means rotor is providing the input power ie. we are converting vehicles KE to electrical energy and this energy can be fed back into battery - a win win.
It's very easy to implement you only need to bother how to feed back power to battery. But by this very reason plugging brakes are much stronger.

{{< figure
  src="images/regen_vs_braking.png"
  alt="Induction machine power curve"
  caption="3 operation regions for a induction motor, notice the slip axis values, Figure is taken from - Modern power electronics and AC drives by Bimal K Bose"
>}}

Generally all vehicle applies regenerative braking at 0 throttle according to the regen power map. Majority of deceleration will be handled by regen and we don't need to initiate manual braking very often. The amount of regenerative braking is usually pre-set or is a constant lookup map wrt rpm - based on the amount of current battery can accept.

In case of extreme braking requirements it's good practice to combine plugging brake and manual brake as plugging brake will have minimal/no delay (if no brake smoothing for user comfort). Some motor controllers provide emergency braking features.

### Motor controller smoothing parameters
Now we will discuss about major design choices for throttle and braking limits for user comfort.

Vehicle control comes in different modes -- speed control or torque control
  - Speed mode is only PID loop on top of torque control - all FOC control algorithms track torque references
  - This torque reference goes through multiple filter before being finalized for FOC reference.
  - These parameter values are what we are looking for while system modeling.

Actuator deadband, max value, throttle map, accel rate, accel release rate , delay to ramp up/down the reference. (same set for brakes)
Regen braking power map (rpm vs regen), regen braking % limit (wrt drive current), same for plugging brake
Drive current limit (motors power vs rpm map). Emergency braking strength.
Neutral braking,  Neutral braking taper speed, Regen taper speed (at very low rpm regen isn't effective).

In our case, we use speed mode, with regenerative braking active at 100% of charge power (the current reduces across rpm since battery charging power is constant)
Regenerative braking related Parameters
Drive current limit  -- kept smaller than the brake current limit
Regen current limit  -- regenerative braking is involuntary -- applied when throttle is release
Brake current limit  -- Active braking on command
Power limit map      -- Obtained from motors load testing from manufacturer


## Steering setup
{{< figure
 src="images/steering_setup.png"
 alt="Steering actuator"
 caption="A DC brushless motor is attached to the steering, converting manual steering to steer by wire"
 width="400px"
>}}

Actuator sizing:
  This part involves identifying what is the load scenario for the steering actuator, this can be done in multiple ways, but essentially measuring the torque required throughout the profile isn't required. Rather its about identifying the outer bounds of steering torque limits, through literature for track vehicles, steering torque would be maximum ~3Nm and normally ~1Nm. Given we run only in road at low speeds - this is perfectly valid limit in our case.

We had AK80-8 CubeMars motor to work with, and it satisfied torque and rpm requirements. This is a brushless dc motor with dual encoder and can save its position within 360 deg of absolute 0. But the steering rotates more than 720 degree to each side in our case hence we have added an absolute position encoder(+-60 deg) with a gear ratio of 12. Communication with the motor is using normal can, the motor has multiple modes of operation position, speed, torque etc...

During testing it was found that the motor has unpredictable high jitter in position control mode, hence position control with PID on top of velocity mode control was designed.

PID response graphs are discussed in modeling section.

- Code for the arduino based controller - [LINK](https://github.com/HAR0070/ACC/tree/main/canfd)
- Code for the CANFD device based controller - [LINK](https://github.com/HAR0070/Steering/tree/main/CAN_send_v1)

It was decided to have a simple cad as steering column cad and actual system didn't match. Choices involved choosing from belt drive or gears. Other criteria of comparison like gears being more efficient, and less prone to slip etc... are not relevant in this context and rpm and torques are very low.

Belt drive
* Adding a tensioner pully would give leeway from alignment issues
* Higher number of components

Gear drive
* Fewer components
* Need to maintain optimal load to keep the meshing tight

I choose gear drive, with 1:1 gear ratio because our actuator could give more torque than required.
Mounts were designed to keep optimal radial loading on the gears. This was a retrofit gear assembly designed and manufactured using laser cutting, metal bushing, machining, welding and 3D printing. The pre-loading required for the gear assembly is managed by washers on the bolting point choice of thickness of the mild steel plate is choosen accordingly (bending moment calculation for given tip displacement). This resulted in a stable and rugged system.

<div style="display:flex; gap:20px; justify-content:center;">
  {{< figure src="images/steering_cad.png" alt="Steering actuator" >}}
  {{< figure src="images/cad2.png" alt="Steering actuator" >}}
</div>

<p style="text-align:center;">
CAD design for attaching the actuator to the steering column
</p>

Note - While working with steering column (OEM supplied) you might suddenly notice its much shorter that at start. The steering rod have a varying length design, they are 2 concentric rods within a cover with an interference fit bearing at start and linear slide bearings at end.

## Steering takeover Design
Objective : First reaction of the users while encountering a misbehaviour by the vehicle is to grab onto the steering, and hence we want to devise a way to enable taking over the vehicle control using the steering.

Given the problem there are majorly 2 methods to go about this

Giving the user control to steering with some load acting against them.

- [Model predictive control of steering torque in shared driving of autonomous vehicles](https://pmc.ncbi.nlm.nih.gov/articles/PMC10451058/pdf/10.1177_0036850420950138.pdf) - As described here create a model of the steering system using the inertial parameters and treat the self aligning torque as disturbance and devise an observer using which you can track the setpoint. And since the input torque is what is controller user can give external torque to drive the steering to his liking - given the tracking error reduces - but here there is no way to detect a user input since every external torque is a disturbance, and moreover the driver has to apply much higher torque than normal driving. Similar works are - [Vehicle State Estimation Using Steering Torque](https://skoge.folk.ntnu.no/prost/proceedings/acc04/Papers/0378_ThA05.4.pdf)

Giving the user the whole vehicles control as he applies some torque on steering.

- This approach is to model the self aligning torque and add the inertia and damping components to identify the total torque required to be applied by the actuator - here the difficulty is - self alignment torque is a function of tire parameter, terrain and vehicle mass that a lot of coefficients...

$$
J_{eq} \ddot{\delta} + B_{eq} \dot{\delta} + \tau_{di} + \tau_f \operatorname{sign}(\dot{\delta}) = N_s \tau_m
$$
Where:

* **$\delta$ (Delta):** The steering angle (position) of the wheels.
* **$\ddot{\delta}$ and $\dot{\delta}$:** The steering angular acceleration and velocity, respectively.
* **$J_{eq}$ (Equivalent Inertia):** The total rotational inertia of the system, combining the inertia of the steering column, motor rotor, and gears, reflected to the steering shaft.
* **$B_{eq}$ (Equivalent Damping):** The viscous friction coefficient representing resistance proportional to speed (e.g., grease viscosity in gears).
* **$\tau_{di}$ (Disturbance Torque):** The sum of external forces acting on the steering rack, primarily the **Self-Aligning Torque (SAT)** generated by the tire-road contact patch.
* **$\tau_f$ (Coulomb Friction):** The static/dry friction torque inherent in the mechanical assembly (gears, bearings) that opposes motion direction.
* **$N_s$:** The steering gear ratio (mechanical advantage of the steering column).
* **$\tau_m$:** The electromagnetic torque generated by the steering actuator motor.

Since we are operating at low speeds (25Kmph max) which makes linear approximations clearly valid, this is much less complicated.
But still on trying to curve fit the data to the model, following difficulty were faced:
  * For curve fitting - we first need to identify which is the major component of the steering torque, inertial , damping, self aligning, or frictional.
  * Identifying how the steering inertia scales with the mass of vehicle is another challenge.
  * Backlash in the steering rack and pinion.
  * Presence of 2 UV joints in the steering rod.
  * Least square curve fitting is very prone to outliers

Future work will include more effort into model based approach. Current Approach to the problem is through Machine learning techniques.

The problem is solved as an unsupervised one, there is no labeling provided for initiation of the takeover and end of it. For this I trained the model to predict the steering torque from data (imu and steering feedback). And if the required torque is higher than the predicted value by 1.5 times the local standard deviation continuously for a few timesteps then driver takeover is initiated.

Test and Train dataset:
The steering was controlled by joystick and the current and rpm of the steering motor was monitored along with steering commands. We drove the vehicle over distance of ~2km, this was considered as training data, next for a smaller distance an external opposing force was applied on the steering, this is taken as test data.

A small snipet of data
{{< figure
 src="images/steering_torque_vs_rpm.png"
 alt="Steering actuator"
 caption="raw data showing steering torque and rpm, it can be seen that even to keep the steering at a position varying torque is required"
>}}

To understand how much noise is there in data we can make some assumption
 * signal is smooth, so deviation from the rolling mean is noise
 * higher frequency components in the signal is noise

{{< figure
src="images/estimating_noise_steering_data.png"
alt="Steering actuator"
caption="Torque signal is reconstructed after removing higher frequency components from the power spectrum"
>}}

Now that we have estimate of noise, various models can be tried out and based on whether it's a variance or bias issue, we can work further. I fitted DecisionTreeRegressor model to the data to understand the feature importance. In general found that current timestep data points are not enough for the prediction.

Hence had to increase the feature vector space.
- Initial features were steering position, speed and acceleration. Vehicles roll, pitch, yaw, lateral velocity and longitudinal velocity.
- Added features are vehicle sideslip angle and path curvature.

Then I performed autocorrelation and partial autocorrelation study on torque values. And subsequently added lag parameters by 1,2,3,4,5 and 10 steps, rolling mean and exponential mean, difference by 1,2,3,4,5 steps for all the parameters.

By using GradientBoostingRegressor from sklearn, with K-fold spliting, trained the model to analyze the feature importance. Then from the above ~150 features dropped down to 30 top features. On these features I performed a gaussian process minimization to get optimal depth (2-14) and learning rate (log scale 1e-4 - 1) and took parameters on stable location (12,13,14th iterations)

{{< figure
src="images/gp_minimize.png"
alt="Steering actuator"
caption="Notice that test error doesn't change much and we found that ~0.2Nm noise is present hence not a huge advantage is seen in gp minimization"
>}}

Finally while implementing 30 parameters were found to be hard to track, hence another reduction to get it to 10 variables was performed. And the 10 most important features came out to be
* torque_lag_1
* torque exponential moving avg 5 points
* steering speed
* steering speed lag 3 steps
* position
* position difference of 1 and 2 steps
* steering acceleration difference of 3 and 4 steps
* yaw difference on 1 step

With these features the accuracy was good, during testing driver was comfortably able to takeover the steering.
{{< figure
 src="images/steering_takeover_pred.png"
 alt="Steering actuator"
 caption="Points shown in green are the prediction values from the ML model, the way the model is used, data leak is not possible hence gives confidence on predictions. "
>}}

@note - steering motor had command timeout of ~1 second, and in earlier testing, post takeover the steering wheel would still move for 1 second, and it felt like driver had to exert so much force to overtake. Which was not the case. Initially I tackled this by giving 0 torque command immediately after overtake (before I realised timeout was the cause)

## Modeling vehicle's drive by wire

Modeling experiments and results - example
- Sensor responses - IMU frequency response - show the IMU vibration
  what all does the sensor do - in terms of fusion - RTK gps,  - acceleration has lot of vibration but velocity is clean and good. Comparison with vehicle odometry value
- Vehicle modeling (accel and brake)- in torque mode --  time delay, ramp rates -- --

- Steering modeling - PID response for step input while loaded -   sequence of input -- what steering does and what model does

### IMU data

The IMU used is fixposition visual inertial navigation unit. It is a self-contained product, all calculations and processes such as
sampling, coning & sculling compensation and the sensor fusion algorithm run on board at 200Hz. RTK corrections are from Survey of India CORS portal. There is communication support through ethernet/wifi tcp, CAN and RS232. SDK support ros1 and ros2.

| General Specs |  |
|------|------|
|Dynamic Drift Roll (°/hr) rms| 0.4|
|Dynamic Drift Pitch (°/hr) rms| 0.4|
|Drift Yaw (°/hr) | 0.4|
|Velocity (m/s) | 0.1|
|Accel Range (g)| ±20 g |
|Accel Noise (mu g/√Hz)| 65 |
|Gyro in run bias stability (deg / hr)| 2|
|Gyro Noise (°/s/√Hz)| 0.003|

If we publish the vehicle odometry, the IMU can integrate it into it's fusion.
The RTK corrections are used to fine converge the IMU, post that RTK isn't required.

To first understand the noise characteristics of the IMU we do a static reading. As you can see, there is no static noise.
{{< figure
  src="images/static_imu_vibration.png"
  alt="static_imu_vibration"
  caption="Static imu frequency power density check"
>}}

When the vehicle is moving the observed data is
{{< figure
  src="images/moving_imu_vibration.png"
  alt="moving_imu_vibration"
  caption="Moving imu frequency power density check"
>}}

@note - If you do resampling by mean - it acts as a low pass filter, also filling NAN with forward/backward fill or drop NAN will induce error - be thoughtful

Now we need to check the frequency component at 2-5 Hz is signal or vibration / or how much of it is signal or vibration, few methods I deviced are
* Record data while imu is in your hand and moving
* Record data when vehicle is moving in different speeds
* Look at data from other axis - if y axis spectrum matches z - mostly it's noise
* Sample the imu at much higher rate and check PSD again

What does a power of 0.06 mean? This represents the variance density of the signal at that specific frequency. i.e., here around 2-3 Hz there is a variance concentration of 0.06 for every 1Hz bandwidth. To get the actual noise we should check the area under the PSD curve in this region, which comes out to be (intentionally not dealing with the units here for signal power).

On applying simple jerk limit filter
{{< figure
  src="images/imu_jerk_limit.png"
  alt="imu_jerk_limit"
  caption="Moving imu z axis PSD check after applying jerk limits, shows very good characteristics, across 1.5 to 4Hz RMSE noise is 0.116m/s2"
>}}

{{< figure
  src="images/imu_jerk_limit_ax.png"
  alt="imu_jerk_limit"
  caption="Moving imu x axis PSD check after applying jerk limits, shows very good characteristics, across 1.5 to 4Hz RMSE noise is 0.13m/s2"
>}}

@note - the logarithmic scale of x axis means, even lower value of PSD can cumulate into higher noise, hence I plotted the cumulative area under the curve to understand the contribution of noise across the spectrum.


### Radar data
We are using a SR75 radar, which is a high-precision FMCW radar. The radar emits a continuous radio signal, sweeping its frequency linearly (chirp) over a time period. Unlike pulsed radar.
* Beat frequency formed by delayed echo gives the range
* Doppler shift gives the velocity

These are simultaneous and independently measured of relative distance and velocity. The operation freq is 100ms ie. one sweep takes 100ms and post that all the data is published over CAN at once. Radar can give a raw 4D point cloud or tracked object position and velocity based on internal fusion (but not both, needs parameter configuration). We are using tracked objects.

Distance
* Resolution is 0.1m  
* Precision is +-0.05m

Speed
* Range +-18m/s   
* Speed ratio 0.25m/s   
* Precision +- 0.12m/s

Additionally radar gives radar cross section values, these can be used to filter the ghost objects. A RCS threshold value of 50 worked for us (glass and other penetrable object have low rcs)

Radar uses CANFD because of high rate of transmission at end of each cycle. For reading raw 4D points the read buffer should be very high, atleast ~2000. The CANFD radar driver code is linked here.

### Steering model

The steering dynamics are as shown
{{< figure
  src="images/steering_step_pos.png"
  alt="steering_step_position"
  caption="Steering actuator response to step input command, as you can observe its a PID with input saturation"
>}}

{{< figure
  src="images/steering_step.png"
  alt="steering_step_velocity"
  caption="Steering actuator response to step input velocity command, as you can observe its a perfect linear response with almost no delay"
>}}

Now there is option either to model the PID response using a first order system, by calculating gain and time constant then while optimizing apply a saturation filter.

OR

Directly use the velocity commands to control the steering. In my case, there is a MPC with bicycle model and since its MPC I can easily add constraints hence making the steering rate the actuation command is better choice.

### Vehicle modeling

Model fitting:
From the motor and controller operation details, we know that powertrain's torque generation capacity is limited by limits on reference, hence we are likely to observe a linear ramp of predetermined slope with some initial delay (actuation delay and delay to ramp higher than rolling friction) for acceleration and similar for braking. So this is not same as classical modeling methods where we try to fit 1st order system or similar using time constants and gain.

This would mean - identifying time delay, and the ramp rate, now the ramp could be defined in terms of acceleration of time ie. ramp is constant of 1m/s3 or delta setpoint takes 0.5 seconds (notice its not unit delta, its any delta applied by the user). Additionally have to identify the rate of decay originating from rolling resistance and other drag forces on the vehicle (which again for low speed is const for a given mass of vehicle).

Delay is the next major part, the delay for the model involves time of publishing the command (time or publishing the command as rostopics) to the time where the vehicle response is recorded (feedback recorded in rostopic) - IMU showing corresponding increase in velocity (can be change in vehicle odometry reading too).

{{< figure
  src="images/step_inputs.png"
  alt="step_inputs"
  caption="step input for throttle, out of which time delta is observed."
>}}

| Time delta |  | | | | Average |
|------|------|------|------|------|------|
| From stationary | 1.353 | 1.202 | 1.266 | 1.083 | 1.226 |
| During movement | 0.802 | 0.902 | 0.752 | 0.852 | 0.827 |

| Ramp rates | values |
|------|------|
| Acceleration rate |  1m/s3 |
| Deceleration rate | 2.5m/s3 |


-------------------------------------------

<script type="text/javascript" id="MathJax-script" async
  src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js">
</script>
<script>
  MathJax = {
    tex: {
      inlineMath: [['$', '$'], ['\\(', '\\)']],
      displayMath: [['$$', '$$'], ['\\[', '\\]']]
    }
  };
</script>
