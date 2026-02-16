+++
title = 'High speed hydrodynamics'
date = 2026-02-14T11:21:33+05:30
draft = false
+++

This article is about how I made the ground effect UAV takeoff from water. I have in general found high speed hydrodynamics lack modern resources. This would be a small contribution from my side.

Making a plane take off from water is harder than making a fast boat and a fast plane. This isn't a vertical lift off UAV, and on top runway is chaotic.
There is a huge story of why Russians didn't succeed, how NASA tried to make a blue whale - which never got out of paper [LINK](https://drive.google.com/file/d/1-fZBEmZvyxrRGafyKTTqvci3ul463WaA/view?usp=sharing)
Few companies making GE planes are poseidon, next closest is Regent.

There is very less litrature in high speed hydrodynamics, the ones that exists are mostly experimental. Especially for seaplanes there is no structured textbook or resources. Closest one is for high speed marine craft by Savitsky, which is very extensive emperical study, resulting in emperical equations for planning hull designs.

## Prismatic Hull
Hence I will start with key learning from this paper, objective being to understand the limits. The paper is majorly about the force balance and how the wetted surface is formed. How is the pressure distribution and resulting drag , buoyancy and moments on the moving hull.

<div style="display:flex; gap:20px; justify-content:center;">
  {{< figure src="images/hull_force_balance_simple.png" alt="Force balance with Normal force from water, thrust, drag and weight" >}}
  {{< figure src="images/hull_force_balance.png" alt="Moment balance on CG for a given trim" >}}
</div>

<p style="text-align:center;">
Force and Moment balance though CG
</p>

$L_k$: Wetted keel length
$L_c$: Wetted chine length
$LCG$: Longitudinal distance of center of gravity from transom (measured along keel)
$V$: Horizontal velocity of planing surface
$\Delta$: Load on water
$T$: Propeller thrust
$N$: Component of resistance force normal to bottom
$D_f$: Frictional drag-force component along bottom surface
$a$: Distance between $D_f$ and CG (measured normal to $D_f$)
$f$: Distance between $T$ and CG (measured normal to $T$)
$c$: Distance between $N$ and CG (measured normal to $N$)
$d$: Vertical depth of trailing edge of boat (at keel) below level water surface
$\tau$: Trim angle of planing area
$\epsilon$: Inclination of thrust line relative to keel line

As all the mechanics problem we start with identifying all the forces and moments by drawing a free body diagram. We will compute magnitude of each as we go. The weight and thrust forces are obvious ones. If you didn't understand all naming don't worry and tag along for next section.

### Force and moment balance (longitudinal axis)
Major questions to ask is how to find the normal forces location.

First we look at how is the pressure distributed if a flat plate was move though water
{{< figure
  src="images/flat_plate_pressure_distribution.png"
  alt="Spray around the plane"
  caption="The spray rises high and touched the wings, causing reduction in lift during takeoff and higher drag"
>}}

Now to understand the structure of the hull and some naming, we look at how a prismatic hulls wetted surface area looks like.
{{< figure
  src="images/pris_hull_naming_water_level.png"
  alt="Spray around the plane"
  caption="The Hull in discussion is a prismatic hull, wetted surface area while moved through water at a trim angle is shown here. "
>}}

Now we need to understand how does the water flow through the hull's wetted surface area and how is the spray formed.
{{< figure
  src="images/hull_spray_characteristic.png"
  alt="Spray around the plane"
  caption="Initially the water has high momentum and velocity, on contact spray rises from spray root and a pressure difference arises at the surface of hull and water which sucks water into the surface and carries it towards the end of chine causing the transverse flow"
>}}

The suction force created at the interface of the water and hull is the reason boats can't fly using wings. We will discuss this.

The normal force can be found is we can map the water contact pressure. The paper splits this normal into dynamic forces and buoyancy force and dynamic force is considered to have center of pressure at 75% of mean wetted area (inspired from flat plate pressure distribution) and buoyancy's COP is considered to be at 33% from the transom. Now these values gives good match for test data and empirical formula but once you deviate from the prismatic shape form depending on the shape you should modify these values. Based on this generelisation forces can be empericaly found for given trim angle and hull parameters in the paper. Do checkout the paper for the formulas and charts.

The Normal force has 2 components with it, lift and drag (this normal includes the buoyancy forces, and in paper is determined as fractions of buoyancy).
<div style="display:flex; gap:20px; justify-content:center;">
  {{< figure src="images/planning_surface_drag_non_viscous.png" alt="Normal force for non-viscous fluid interaction" >}}
  {{< figure src="images/planning_surface_drag.png" alt="Normal force along with viscous drag from water" >}}
</div>

<p style="text-align:center;">
Components of the normal force into drag and lift. Understanding this picture will simplify the formulas you see in the picture a lot.
</p>

The general formulation of the drag and lift is very similar to aerodynamics.

$$Force = k \times \frac{1}{2} \rho V^2 L^2 $$

The constant depends on $f\left(\frac{VL}{\nu}\right)$  reynolds number, $F\left(\frac{V}{\sqrt{gL}}\right)$ froude number and  $\varphi\left(\frac{V^2L}{\sigma/\rho}\right)$ weber number. In practice it's obtained through test or CFD. The characteric length is generally the beam (width of the hull) or volume^1/3.

The paper derives a closed form equation for $C_l$ and $C_d$ prismatic hull. There are reynolds number based corrections to account for change in frictional resistance due to boundary layer thickness (smaller hulls have higher frictional components ratio).

Moment balance has another major decision maker and that is trim angle.

The performance of the hull is evaluated with drag to lift ratio (very similar to L/D for planes). The prismatic hull is the shape form which creates minimal drag, but with a major problem of spray. All the modification on chine and bow of the hull is to reduce the spray characteristics and increase stability at the cost of drag.

Conclusion is this is the best drag/lift ratio any fast moving shape in water can have. (Depends on surface friction coefficient too, if you use a aluminum hydrophobic coating drag will reduce)
{{< figure
  src="images/drag_components.png"
  alt="Components of the drag for prismatic hull"
  caption="At low speeds viscous drag is much higher because of higher wetted surface area, and the pressure drag takes over"
>}}

@note - There is a important concept of Hump drag in high speed hydrodynamics, which happens at Frude number of ~0.5-0.7 and characterises the transition from displacement mode (the hull smoothly cutting through water) or buoyancy lift to planning mode (where hull rides on top of water) or majority dynamic lift. Essentially at a certain speed range hull has to ride through the waves, created by itself (bow interaction) which increase drag considerable.

@note - Always lookout for Cv definitions and units. Its a normalizer for hull lift and drag coefficients and can be defined based on beam or (load on water)^ 1/3.

Although these formulas are heavily test result dependent, there is a huge value for it as a benchmark. If you do CFD, you would know usually all the settings for mesh and solvers are determined using validation resources from NASA Langley Research Center - Turbulence Modeling Resource [LINK](https://turbmodels.larc.nasa.gov/naca0012_val.html) for NACA 0012 airfoil (or high lift with flaps) or from drag prediction workshop or high lift workshop results as references. But there is nothing similar for hydrodynamics, for multiphase interaction or anything close (nor anything for ground effect flights) this prismatic hull serves as the best reference available to calibrate the CFD. You shouldn't believe on any CFD results that isn't calibrated against physical experiment.

### Hydrostatic Stability
General concepts are same for any body floating in water.

In here metacentre is the position along the a line perpendicular to waterline (at equilibrium) passing through center of gravity. The exact point of MC is intersection of above mentioned line with buoyancy force vector. Buoyancy always acts at center of buoyancy which is volumetric center of submerged part of body. (Hence higher water line is better for stability unlike for drag).

{{< figure
  src="images/lat_stability.png"
  alt="transverse stability for general submerged body"
  caption="Any body that has more buoyancy than weight can float in water, but there is a specific condition to keep it floating upright, according to the picture body's metacentre should be higher than CG"
>}}

Looking at the picture try to think when would the body be in unstable equilibrium and what is the range of stable equilibrium. Thats all there it is.

In our context it is important to understand, stability should be analyzed along both axis of the plane. Along the length the seaplane has a long body hence there is a long correcting lever arm but along the width dimensions are really small in comparison (of the fuselage). The waterline area has very small lateral moment of inertia which results in metacenter below cg. So the lateral stability for a normal streamlined plane body is very poor, the best way to improve it is to use the wing tips (one other alternative is stubs [LINK](https://avweb.com/news/dornier-introduces-new-amphibian/)), attach a high buoyancy object at the wing tips for a long corrective lever arm resisting rolling moment.

Now if you keep something at the very tip of wing, structurally its a bad design hence there is a compromise. Empirically its found that if transverse and longitudinal metacentric heights are almost same and transverse not larger than longitudinal then satisfactory stability is achieved.

{{< figure
  src="images/elsevier-seaplane-floats.png"
  alt="Floats on seaplane"
  caption="Position of floats on a seaplane"
>}}

The floats are mounted higher that keel, so as to avoid contact with water at higher speeds and thus reduce drag. But it should be deep enough that we meet the transverse stability criteria. Also floats should have planing bottom to furnish dynamic lift even while fully submerged.

@note - Do consider the dynamic force for moment balance while takeoff.

Now we will discuss about the reserve buoyancy. Reserve buoyancy is the additional buoyancy the floats can provide in comparison to stationary waterline. So think of situation where due to disturbances the planes rolls so much that one float is fully inside water, this would create corrective moment of buoyancy force on float X wing distance pushing back the plane to stability. Now how much extra buoyancy is provided by the float will determine how strong disturbances can be handled and generally its recommend to have 100% reserved buoyancy for floats but you can calculate the number based on the sea state the plane has to survive.

You should also ensure that the center of buoyancy and center of gravity lies on same vertical line normal to the waterline. You can play around with mass distribution in longitudinal axis to have waterline parallel to keel or to achieve the best dynamic trim with lowest moment in takeoff. You should keep mass distribution symmetrical in transverse axis.

@note - these floats have to be perfectly symmetrical as a small imbalance will cause very high yawing moment (forces acting far from cg).

## The Hull

When it comes to hulls, I want you to understand that these are pretty much like airfoils, their behaviour cant be fully predicted by first principles but general design features convey a lot of information. Similar to airfoil here also you will find charts for lift, drag and moment coefficients and as you may have guessed there are standard measurement procedures and normalizations etc... for results to scale across sizes. NASA facility tested a lot of seaplane hulls during world war and later open sourced their archive. Here I am going to go through one such paper, will share links to some other interesting ones.

[AERODYNAMIC AND HYDRODYNAMIC TESTS OF A FAMILY OF MODELS OF FLYING-BOAT HULLS DERIVED FROM A STREAMLINE BODY-NACA MODEL 84 SERIES](https://drive.google.com/file/d/1rKoCDa1SnGaYWayhtA2oYVhR_CcG7RK6/view?usp=sharing)

I would recommend you to read the summary section alone. The conducted study aims to derive a planning hull out of perfectly streamlined body, by evaluating various alteration possible mainly bow design, aft body design, deadrise angle, chine shapes, additional planning surface etc ... through towing tank test and wind tunnel testing. Each alteration has conclusion on performance but most importantly its concluded the aerodynamic drag can be as low as 1.25 times the perfectly streamlines body.

This paper has extensive test results for drag and moments in water and aero (For aero you need to be carefull of the coefficient definitions). Also it provides very clear diagrams and detailed for recreating the hull shapes. We will go through definitions first.

Instrumentation details
- Stevens institute is very well known for their experiments on sea plane designs. And its a honor that I could present my work to [Prof. Raju](https://www.stevens.edu/profile/rdatla) who is leading the towing tank testing.

<div style="display:flex; gap:20px; justify-content:center;">
  {{< figure src="images/towing_tank_pic.avif" alt="Towing tank from Davidson laboratory" >}}
  {{< figure src="images/towing_tank_symantic.png" alt="Symantic representation of towing tank setup" >}}
</div>

<p style="text-align:center;">
Towing tank setup.
</p>

Free trim test - following values are measured.
* vertical displacement of model / heave / rise or change of draft
* angle of stable trim
* resistance values

Fixed trim test - following values are measured.
* vertical displacement of model / heave / rise or change of draft
* Pitching moment exerted by the model
* resistance values

Coefficients
$C_\Delta$ (Load Coefficient): Defined as $\Delta/wb^3$.
$C_R$ (Resistance Coefficient): Defined as $R/wb^3$.
$C_V$ (Speed Coefficient): Defined as $V/\sqrt{gb}$.
$C_M$ (Trimming-Moment Coefficient): Defined as $M/wb^4$.
$C_d$ (Draft Coefficient): Defined as $d/b$.

Variables
$\Delta$: Load on water, measured in pounds.
$w$: Specific weight of water, measured in pounds per cubic foot (63.3 for these tests; usually taken as 64 for sea water).
$b$: Maximum beam, measured in feet.
$R$: Resistance, measured in pounds.
$V$: Speed, measured in feet per second.
$g$: Acceleration of gravity, 32.2 feet per second per second.
$M$: Trimming moment, measured in pound-feet.
$d$: Draft at main step, measured in feet.

{{< figure
  src="images/seaplane_cg_var.png"
  alt="How does the variation of cg affect the drag"
  caption="How does the variation of cg affect the drag"
>}}

- As you can notice there is trim values and resistance values for speed coefficient upto 4. This is not where takeoff happens
- The trim shown here is free trim, ie. the hull is pivoted on cg while testing and after the hydrodynamics moment balance the hull stabilizes itself at this trim. Idea is speed is so low that aerodynamic forces cant control trim at this velocity.
- Notice the shape of the curve how the trim and resistance increase simultaneously in shape of a hump.
- $C_\Delta$ is the weight coefficient, as you can see as $C_\Delta$  double resistance increase almost 3 times.
- If you look at the values of the  $C_\Delta$ /$C_R$, you will notice it comes out to be L/D and its around 6.6 for $C_\Delta$ = 0.4) and 4.4 for $C_\Delta$  = 0.8. This is the point of maximum thrust.

After each section extensive images are captured to show the spray characteristics of that particular change. This is very important because wings and propellors will get highly impacted by spray characteristics. Aand very soon spray will because major hurdle to takeoff.

{{< figure
  src="images/seaplane_stern_spray.png"
  alt="How the stern height change causes spray"
  caption="Spray from the aft of the hull as the stern height is changed. Have a careful look and think about where would the wings come"
>}}

Bow waves are major hurdle for placement of propellors, and like conventional planes they cant be placed on wing level. Just due to spray they have to be high mounted and still designed to ingest spray.
{{< figure
  src="images/seaplane_bow_spray.png"
  alt="How the bow shape change causes spray"
  caption="Spray from the bow of the hull arises at low speeds. These water splashes will be in front of the propellors if they are mounted on wings and propellors suction will pull it in"
>}}

Post this section page 74, there is a new set of study going further from speed coefficient of 4.5 to 9 where the plane takes off.
- This is a more detailed study and each graph set you see have a $C_V$ written on top, resistance and moments are studied vs speed, weight and trims.
- This is different from the earlier graphs shown for fixed trim region.
- Depending on planes trim and weight as speed increases you need to look across graphs for resistance and moments.

The graphs depict what is the resistance for each trim across the speed range also showing trim transition for lowest resistance and lowest moment requirement (design choice based on whether you are power limited vs moment limited)

Post that you will notice the aerodynamic coefficients - I guess its safe to assume everyone know how to read it. But be careful the definitions are not conventional ones and are mentioned in page 73.

I have tested Bow 2B, stern 2C and 4.
{{< figure
  src="images/seaplane_stern_2_drawing.png"
  alt="Drawing for seaplane stern"
  caption="The drawing for hull stern, it is as complicated as it looks and I would strongly advice against using conventional ship drawing tools. Just trace the drawings out in CAD software like solidworks of Fusion360"
>}}

{{< figure
  src="images/seaplane_bow_2B_drawing.png"
  alt="Drawing for seaplane bow"
  caption="The drawing for hull bow"
>}}

After making these drawings into CAD, this is how they look.

<div style="display:flex; gap:20px; justify-content:center;">
  {{< figure src="images/seaplane_cad_bottomview.png" alt="Bottom view of the hull" >}}
  {{< figure src="images/seaplane_cad_topview.png" alt="Top view of the hull" >}}
</div>

<p style="text-align:center;">
CAD done in Fusion360, designed for 3D printing and testing.
</p>

Other such reports
[LINK2]()
[LINK1](https://reports.aerade.cranfield.ac.uk/bitstream/handle/1826.2/3390/arc-rm-2834.pdf?sequence=1&isAllowed=y)


## Evaluating hulls
Now we will take these ideas further and go bit more deep. I recommend [LINK](https://drive.google.com/drive/folders/1-_mFKr5Hu_giDAQSaBu7Lg_rmcxvn8il) going through this. As now you understand drag calculation and how did they select/make that hull design.

First we should be asking why cant prismatic hull fly. The answer is linked to ventilation - hull needs air to be injected beneath it to break off from water. Also to achieve high speeds in water there should be a way to reduce wetted surface area compensating for v^2 increase in dynamic lift without crazy high trim angles. A step in the hull is the solution, we will discuss in takeoff section how trim changes and aft part of the body can lift off from water - but this wont be possible without step due to suction force of water (due to velocity difference at intersection with water). What happens is as the aft of the hull rises air rushes in to fill the lower pressure region and cuts off the suction interface. Now you would think why cant we inject air (have some pneumatics) which would result in a streamlined fuselage and reduction in drag. Answer is yes. But still there should be a small step for the second condition (this can be retractable). Read the takeoff section to get a better idea.

{{< figure
  src="images/stepped_hull_force_moments.png"
  alt="Stepped hull force and moments"
  caption="Force and moment balance for stepped hull"
>}}

There is 2 planing surface for a stepped hull. The forebody can be treated very similar to prismatic hull but the aft body rides on the wave created by the forebody and waterline is below undisturbed water level. Now these surfaces can be considered to be 2 individuals with difference in surrounding water condition. The aft part of the hull has a angle with respect to the step and this is called stern post angle.

For static conditions we can do the same force balance as we discussed earlier, but this time there is 2 normal force component, location of the thrust is different (based on where you mount the propellors) which creates a moment. The split of normal forces can be found through moment balance on cg from buoyant forces acting at center of buoyancy of fore and aft body which sums to weight of the plane. This is the same force balance that works out dynamically also but the water level isn't predictable.

The hump in the resistance curve is critical in determining ability to takeoff. It's primary associated with maximum trim and occurs around same velocity range. Higher the trim at this speed, higher is the resistance (checkout some graphs from the NASA paper). The ways to decrease trim are
* Decrease sternpost angle
* Increase the length of aftbody

If you go back to the Force and moment balance picture, you would realize both these actions result in reduction in aft body buoyancy and that's the key, if there is only forebody in play then its a prismatic hull - which is the least drag form factor.

### Takeoff and Landing sequence
While we are calculating the takeoff keep in mind that we have wings, as speed increases the wings takeup a portion of the load. Hence the buoyancy/dynamic lift requirement actually decreases with speed. The takeoff point is where velocity if just enough that aerodynamic lift from wings can carry whole load of the plane.

As the velocity of the plane increases, dynamic lift increases and wetted length decreases (lower buoyancy required to balance weight) ie. planning surface decreases or trim increases, usually simultaneously. At a speed the dynamic lift will pass through the CG and will be equal to the weight at that point the aft body leaves the water surface. The plane stays in equilibrium and for this CG location is most important, tails moment (or make a underwater flap) can be used maintain this equilibrium (do notice that here tail is providing lift and not downforce). During takeoff the whole seaplane is pivoted at step.

For stable takeoff its essential to consider the transition to being fully airborn. Even though lift forces balance out the difference is drag is very sudden, is a low trim is not maintained this would result in pitching moment which cant be controlled by elevator (if elevator isn't sized to control trim or if its wet due to spray). If elevator is sized to control hydro trim then bigger the plane safer the transition due to higher inertia.

Landing sequence is reverse of takeoff for a perfect landing on smooth water. At touch down the impact should be at the step and velocity tangential to water surface. But generally is far from perfect. The vertical force of impact should be controlled as after touch down the force of impact will proportionately increase the draft line causing rebound and leaving water surface. This causes high damping to vertical and horizontal velocity. But if impact velocity is high it can result in heave oscillations (discussed in next section). The impact force is fully taken by hull (dampened by the water) and is extreme loading condition for structural design.

{{< figure
  src="images/landing_impact.png"
  alt="variation of draft and reaction force on impact"
  caption="Change of draft and impact force due at landing"
>}}

@note - most of the landing impact and rebound dynamics happens only at step, and prismatic hull approximation hold realistic.

### Dynamic stability
There are 2 major instability porpoising  and skipping (its same porpoising from F1). Seaplanes are a region of stable trim angle (lower and upper limit) for stability (do keep in mind that waves influence your trim angle). The reason is low damping for the amount of hydrodynamic forces and hence taking off at lower speed is better. When there is large amount of hydrodynamic lift at high velocity and trim angle a small perturbation causes large force imbalance combined with low inertia resulting in sudden vertical/angular motion. If you loose lift plane suddenly sinks and next instant dynamic lift drastically increases and plane leaves the water surface due to acceleration and cycle repeats.

{{< figure
  src="images/porpoising.png"
  alt="porpoising of seaplane"
  caption="Porpoising of seaplane, this is a high frequency phugoid motion with aoa variation and is catastrophic"
>}}

Skipping is heave oscillation and observed while landing .

## Hydrofoil

By now you would be wondering, seaplanes are great but why do I not see one around. Infact there are very few operational and their operation is very limited to certain weather conditions. The major problem is sea is highly turbulent [youtube](https://www.youtube.com/watch?v=cMNH4nmOims). Even though such waves are not observed in costlines the wave handling capacity of conventional hull seaplanes are very limited.

| Gross buoyancy (lb) |  Wave height (ft) | Examples | Sea State |
|------|------|------|------|
| 2000 | 1 | small STOL planes | 1 |
| 4000 | 1.75-2 | 5 seater jets | 2 |
| 8000  | 2.5 | Bigger than Cessna | 3 |
| 20000 | 3.5 | 20-30 seat Business jets | 3 |
| 100000 | 6 | Heavy private jets | 4 |

If you compare in shores you can easily notice waves around your height (sea state 4 is moderate breeze and small waves), that means no small planes or private jets can actually be put into service on majority of the costlines. To handle sea state 6, a strong wind breeze, as the trend goes it would require atleast Boeing 787-8. But the catch is airplane have much lower structural loading and smaller powertrain sizes hence higher payload capacity for same MTOW. Now its a major problem.

### Why hydrofoil

Bow waves are major problem, additionally at start of the planning for a certain velocity range the water suction (we discussed in prismatic hull case) is higher in magnitude than lift and hence cause rise in water level and higher bow waves - which again results in more drag (trap) only way out is to accelerate through this region fast (bigger powertrain).

The spray height is also a issue, as mostly propellors are mounted on the wings, the propellor sucks the spray and can result in tail of the plane being fully wet that you loose pitching control.

A hull capable of doing 2ft of waves can do 4ft with 2ft long retractable hydrofoil. That's equivalent to hulls of 3-5 times the buoyancy. To tackle the rougher sea condition for given size of plane this is the only way.

### How hydrofoil

The hydrofoil should be able to lift you above the water surface at a very low velocity, but also it shouldn't cavitate before the takeoff velocity. Now at very low velocity the moment balance through aerodynamic tail is not possible due to high moments from the water, and hence if a passive hydrofoil is designed - it should be placed on the CG location, with active controlled flaps (or radars) to control the pitching moment.  Or uprooting should be done at a speed where tails control surface cam balance the moments (will result is huge tail design).

If designing a active hydrofoil, its not very different from fast hydrofoil boats, very high speed active control of the rear hydrofoil is required because it will be in the wake of leading hydrofoil. Similar to wings wake, hydrofoil wakes carry high energy (almost all momentum to keep the plane floating is balanced by the wake). [A good visualization of active hydrofoil system](https://youtube.com/playlist?list=PLKzcubLcBdlTMpCYshZnDhOZDETBflw2V&si=hmjk9oGX_zNWLo_e)

Hydrofoils - velocity range to deploy them, why active control is required, XFLR5 images.

Regent hydrofoil is a great masterpiece.
{{< figure
  src="images/regent_main_hydrofoil_mechanism.png"
  alt="Regent's front hydrofoil mechanism"
  caption=" The hydrofoil mechanism , picture is taken from Regents patent filing document"
>}}

The string marked in yellow rotates the pully marked in yellow, which though belt drive rotates the gears attached to the screw drive linear actuator. The linear actuator pulls the hydrofoil up/down. Do notice that there is no flaps on the main foil. Watch out SailGp tech video to have a good look at titanium made hydrofoils, how long, smooth and sharp are they. Also notice the spray from SailGp boats.

{{< figure
  src="images/regent_hydrofoil_side_view.png"
  alt="Regent's rear hydrofoil mechanism"
  caption=" The rear hydrofoil has control over the flaps and can retract, picture is taken from Regents patent filing document"
>}}

Spray from the hydrofoil on a seaplane.
{{< figure
  src="images/hydrofoil_water_spray.png"
  alt="Spray around the plane"
  caption="The spray rises high and touched the wings, causing reduction in lift during takeoff and higher drag"
>}}


## The proto



--------------------------------------
