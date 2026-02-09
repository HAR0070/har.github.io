
---
title: "Step1 to Autonomous cars"
date: 2026-02-10
draft: false
---

## Why this blog exists

Most of the blogs and research papers only talk about theory of modelling and autonomy, here we discover the practical aspects and how to connect all the dots
Starting with understanding the electric vehicle powertrain - to implementing drive by wire and giving innate ability to vehicle to track any reference trajectory provided
This is divided into sections - about the vehicle, automating steering, vehicle drive by wire, software stack

## What you’ll find here
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
