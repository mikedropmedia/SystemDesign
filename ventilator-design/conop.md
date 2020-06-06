# RespiraWorks Ventilator Concept of Operations (conop)

## Purpose

- Build a common understanding of ventilator internal operation amongst different parts of the team.

- Caution: this document has not been formally reviewed and is a work in progress. Be responsible to understand the provenance of documentation.

## Document Content

- Summary of key constraints
- Understanding of current intra-breath cycle control for the development (“Pizza”) board.
- Preliminary description of system operation for next iteration of system with O2 mixing
- Envisioned Inter and Intra-breath cycle control states 
- Open questions / points to resolve.

## Functional Diagram

![Functional Block Diagram](functional-block-diagram.png)

## Alpha “Pizza Build” Intra-breath Cycle Control

The Alpha system flow path is:
- Fan
- Air Valve (not on current pizza build)
- Tee <-> Test Lung
- Exhale valve
- Exhaust

The Alpha build (first iteration) did closed loop control on the blower speed and setpoint control on the exhale valve state to achieve the desired target pressure. However, this left significant shortfalls in both pressure rise and pressure reduction times.

The intra-breath control cycle for the Alpha build is:
- **Inhale:**
Increase pressure set point for closed loop control to PIP pressure 
Exhale moves to exhale position (fully closed)
Controller adjusts blower speed based on PID control closed around the measured patient pressure
PIP pressure target is maintained for a specified amount of time - then step transition to exhale
- **Exhale:**
Exhale valve moves to exhale position (fully open)
Controlled adjusts pressure set point for closed loop control to PEEP pressure - controller adjusts blower speed to meet PEEP.
PEEP pressure target is maintained for a specified amount of time - then transition back to inhale

Alpha build has no inter-breath or “adaptive” control on top of this.

## v0.2 (Covent) Intra-breath Cycle Control

For this next iteration of the ventilator, the CONOPS are changed in order to support the following:
* 100% Oxygen flow, in addition to 100% air (fine FiO2 control to come in a future iteration)
* Closed loop control of the inhale air valve (when using 100% air) or the oxygen valve (when using 100% oxygen) to achieve the target pressure. This decision supports (1) faster rise times than when close loop controlling the blower and (2) eventually moving towards a control set up that coordinates the set points of the two valves in order to give variable FiO2 control - two valves are easier to control in synchrony than one valve and one blower.
* A minimum amount of constant flow through the system to enable more accurate flow measurements from the venturis

The setup of this build can be seen below:
![Diagram](pneumatic-system/assets/pneumatic-diagram.png)

**Intra-cycle control for Pressure Control**
* Inhale triggered by a timer based on the user specified Respiratory Rate and Inhalation Time
    * Note: supporting an Assist mode where breaths can be triggered by the patient are a stretch goal for v0.2 but not a current main focus for this iteration
* Inhale cycle:
    * Close loop control O2 proportional solenoid (for user specified 100% FiO2) or inhale air pinch valve (for user specified 21% FiO2) to achieve and maintain user specified PIP.
    * Blower is maintained running at either constant speed or through a breath-determined, open loop cycle (meaning that it does not changed based on system measurements)
        * Note: open loop cycle triggered at the start of the inhale - cycle goes until end of exhale
    * Exhale air valve is controlled open loop to maintain a minimum amount of flow through the full system
        * Note: open loop cycle triggered at the start of the inhale - cycle goes until end of exhale
* Inhale -> Exhale transition occurs on a timer based on the user specified Respiratory Rate and Inhalatino Time
* Exhale cycle:
    * Close loop control O2 proportional solenoid (for user specified 100% FiO2) or inhale air pinch valve (for user specified 21% FiO2) to achieve and maintain PIP.
    * Blower still running open loop
    * Exhale valve set point still managed open loop

**Open Questions/things to think about (not exhaustive)**
* What will the blower/exhale air valve open loop set points be?
        * Also, are these open loop set points adjusted automatically inter-cycle by the system or are they fixed?
* Does the system want to close loop control the exhale valve on exhale instead of the inhale air or oxygen valves? Unlikely, but will assess system performance.

## Potential Control Strategy for Next Ventilator Iteration

The following reflects discussions around CONOPS once FiO2 control is added.

**NOTE: This is a work in progress and may not reflect the final decisions that are made**

**Key constraints driving CONOPS decisions**
* User constraints (external requirements/desires for functionality)
* Be able to provide fine control of the FiO2 delivered to the patient from 21% (all air) to 100% (all O2)
* Support a breath cycle (inhale - achieve PIP and hold, exhale - maintain PEEP) at high breathing rates
* Minimize amount of wasted O2 (wasted O2 is O2 that flows through the system and doesn’t go to the patient - e.g. O2 used to maintain PEEP) - **this is now under question and may be removed, #todo figure this out**
* Note: that this short list above is not a substitute for the system requirements documentation. 

**Design constraints (internal requirements/desires for managing complexity)**
* Maintain at least some flow through the system at all times to help minimize stall/surge from the blower and ensure a “bias flow”
* Minimize the number of components being independently closed loop controlled at any one time in order to simplify system dynamics

**Envisioned/Proposed intra-cycle control for Pressure Assist/Control**
* Inhale triggered by one of the following:
    * Patient attempting to breathe by detecting an increase change in flow on the patient inspiratory flow sensor. (rather than trying to detect a drop in pressure)
    * Timer expiring (determined by min respiratory rate)
* Inhale
    * Close loop control O2 valve and inhale air valve set point together to achieve and maintain PIP.
        * Together means that the valves are controlled as one unit - with the controller set point mapping to valve position proportionally based on desired FiO2 (as discussed below, current thought is for this ratio for FiO2 will be controlled inter-cycle - with the FiO2 measured through the O2 sensor being used to adjust this ratio across cycles - meaning that only pressure affects the intra-cycle control)
    * Blower running at either constant speed or through a breath-determined cycle
        * The inhale valve is controlled to maintain pressure
        * Note: open loop cycle triggered at the start of the breath - cycle goes until end of exhale
    * Exhale air valve running open loop (but open loop cycle triggered at the start of the breath
        * Exhale air valve open to ensure some amount of flow through the full system even when PIP has been achieved
        * Note: open loop cycle triggered at the start of the breath - cycle goes until end of exhale
    * Pressure is maintained for a specified amount of time
* Exhale
    * O2 valve is shut (conserve oxygen)
    * Close loop control inhale air valve to achieve and maintain PEEP
    * Blower still running open loop
    * Exhale valve still running open loop
**Open Questions/things to think about (not exhaustive)**
* Major
    * How many breaths can be delivered to the patient with the wrong FiO2 (and how “wrong” is acceptable)? Need to ensure this allows controlling FiO2 inter-cycle (believe it does)
    * Exact blower/exhale air valve open loop set points TBD
        * Also, are these open loop set points adjusted automatically inter-cycle by the system or are they fixed?
        * Could consider closed loop controlling exhale valve on exhale instead of inhale valve (desire is to avoid closed loop controlling both independently at the same time)
    * How are we going to detect when the patient is attempting a breath?
        * If the controller is actively maintaining PEEP then when the patient starts to breathe, the pressure may not actually drop much - may be better to look at the change in flow

## Inter-cycle Ventilator States

Note: these are extremely preliminary, much of this is up for discussion

**Boot**
* User turns the ventilator on, it goes through some initialization cycle (including a self-test?) and then says it is good to go
* Does this happen before the ventilator is inserted into the patient or after? Seems important to do this before (make sure the machine is working)?
* Details of this TBD (discussions about this occurring in requirements channels as well)

**Startup (everything that only happens when things are just getting started until a first full breath cycle occurs)**
* User inserts the ventilator into the patient (probably with the ventilator on but not running?), enters the desired settings, and hits start
* Based on some estimates or prior configuration, the controller picks initial values for the proportional control ratio between the o2 valve and inhale air valve for closed loop control on inhale
    * Depends on ventilator characteristics, as well as info about the O2 source (either put at a default or entered by the user)
* Begin first cycle with the initial set points
    * Likely just trigger first breath whenever the system is ready to kick things off, then go with patient breathing?
* Once the first cycle is complete, go to Tuning

**Tuning (adjust set points across cycles in order to achieve desired parameters)** - need to figure out the exact logic here, but at a high level:
* Note - different kinds of tuning
    * Steady state tuning: user provided inputs haven’t changed, just trying to tune system to achieve them better
    * Transition tuning: user provides new inputs and need to adjust the system to achieve them
    * Likely important to split these out eventually since they’ll probably have different alarms/bounds but not doing that for the moment
* Current approach:
    * Any changes to user defined PIP, PEEP, min RR, I-Time that occur during a cycle are used to update the set points for the next cycle (so don’t try to change things intra-cycle)
    * Ratio for how much the ox valve and air valve are moved relative to each other during proportional closed loop control during inhale is adjusted based on measured FiO2 (using the O2 sensor)
    * Compare the FiO2 from the last cycle(s) to the desired FiO2 and then adjust the ratio accordingly
    * Note: if the O2 supply has a lower percent O2 than the desired FiO2, then the FiO2 might not be achievable
    * TBD what other things we might want to tune inter-cycle (e.g. open loop set points for blower/exhale valve)

**Completion - turning things off and removal from patient**
* TBD

**Storage - doesn’t really belong here but needs to go somewhere**
* TBD

## Failure modes
* Keep opening ox valve more and cannot hit FiO2/PIP -> if sustained for a certain amount of time, compensate by opening the air valve. What happens on future cycles? Keep trying the ox valve and then compensating, or just use the air valve by default for a certain number of cycles?
* Keep opening inhale valve and cannot hit PIP -> if possible, increase blower set point for the next cycle; if this doesn’t work, shut the exhale valve completely; if this doesn’t help, do we ever augment with oxygen (and allow a higher FiO2) or just alarm and let the doctor do something?
* Can’t maintain PEEP -> open inhale valve all the way, close exhale valve all the way; still doesn’t work, do we ever augment with oxygen (and allow a higher FiO2) or just alarm and let the doctor do something?

## Other things to do
* Other modes (Pressure support, HFNV)
* Transitions between modes

