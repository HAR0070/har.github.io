+++
title = 'Plane'
date = 2026-02-28T21:12:14+05:30
draft = false
is_gallery = true
weight = 3
+++

## Plane flying

{{% comment %}}
https://youtu.be/Ocp6qKS3Chk
{{% /comment %}}

{{< youtube Ocp6qKS3Chk >}}

## 8Kg Payload - 36KG MTOW

{{< figure
  src="images/big_plane.png"
  alt="36KG MTOW plane"
  caption="36Kg MTOW, 8Kg payload plane"
>}}



## System specification from mission profile
Given the mission profile, we determine the velocites and accelerations that the plane should handle
Based on that VN diagram is made
{{< figure
  src="images/VN-diagram.png"
  alt="VN diagram for plane"
  caption="Flight envelop graph - giving loadfactor for the Velocities required my mission profile"
>}}

Given the mission profile, To achieve required lift in all the scenarios - (take off, landing, cruise speed, climb, max speed, banking/turning ...)
{{< figure
  src="images/wing_area_sizing.png"
  alt="Wing area sizing for the plane"
  caption="Wing area sizing according to mission requirements"
>}}

Sample of wing optimization using XFLR5

{{% comment %}}
https://youtu.be/RkYkpLNP6Es
{{% /comment %}}

{{< youtube RkYkpLNP6Es >}}

## Blended wing GE concept

{{< figure
  src="images/blended_wing_velocity.png"
  alt="Blended wing Ground effect concept"
  caption="Pathlines from blended wing surface - CFD showing forward swept wing characteristics"
>}}


## Validation CFD Cases

{{< figure
  src="images/naca0012_0.png"
  alt="Validation case for 0 aoa NACA0012 airfoil"
  caption="Validation case for 0 aoa NACA0012 airfoil"
>}}

{{< figure
  src="images/naca0012_mesh.png"
  alt="Mesh used in the study"
  caption="Validation case - mesh used for the study"
>}}

{{< figure
  src="images/naca_high_aoa.png"
  alt="Validation case for higher aoa"
  caption="Validation case for high lift generation"
>}}
