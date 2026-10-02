# Arduino Autonomous Line-Following Robot

An Arduino-based autonomous mobile robot developed to follow a predefined track using **three infrared reflectance sensors, real-time line detection and differential PWM motor control**.

The project began with a stripped-back motor-control framework provided for the robot hardware. I developed the additional sensing and autonomous-control functionality required for the robot to detect a line, determine its position relative to the track and continuously adjust the left and right motor speeds to remain on course.

The project provided practical experience in **embedded programming, sensor processing, autonomous decision-making and closed-loop-style robotic steering**.

---

## Project Overview

The robot uses three downward-facing infrared sensors positioned across the front of the chassis:

```text
            ROBOT
              ↑
              │
       Direction of Travel

       L       C       R
       ●       ●       ●
       │       │       │
       └───────┼───────┘
               ▼
             LINE
```

Each sensor measures the reflectivity of the surface beneath the robot.

The sensor measurements are converted into binary states and combined to determine the position of the line relative to the robot.

The complete control process is:

```text
          Three IR Sensors
                 │
                 ▼
        Read Analog Values
                 │
                 ▼
        Apply Threshold
                 │
                 ▼
        Binary Sensor State
                 │
                 ▼
       Three-Bit Line Pattern
                 │
                 ▼
        Determine Error
                 │
                 ▼
    Calculate Motor Speeds
          ┌──────┴──────┐
          ▼             ▼
      Left PWM       Right PWM
          │             │
          └──────┬──────┘
                 ▼
          DRV8833 Driver
                 │
                 ▼
            DC Motors
                 │
                 ▼
       Correct Robot Heading
                 │
                 └──────► Repeat
```

This allows the robot to continuously react to its position relative to the track.

---

# My Contribution

The project was developed from a **basic motor-control framework** supplied as the starting point for the assignment.

The supplied framework provided the low-level functionality required to drive the robot's motors.

My implementation added the functionality required to turn this basic mobile platform into an autonomous line-following robot.

## Provided Starting Framework

The supplied framework included the basic:

- DRV8833 motor-driver interface
- DC motor control
- Motor direction control
- PWM motor-speed control

## My Implementation

I developed the additional functionality for:

- Three-sensor IR line detection
- Analog sensor acquisition
- Sensor thresholding
- Binary line-state representation
- Three-bit sensor-pattern generation
- Line-position interpretation
- Steering-error calculation
- Variable steering corrections
- Differential motor-speed adjustment
- Lost-line recovery behaviour
- Previous-error tracking
- Integration of sensing and motor control into the autonomous navigation loop
- Hardware testing and tuning

Conceptually:

```text
       PROVIDED FRAMEWORK
              │
              ▼
      Basic Motor Control
              │
              ▼
       MY IMPLEMENTATION
              │
      ┌───────┴────────┐
      ▼                ▼
 IR Sensing       Line Detection
      │                │
      └───────┬────────┘
              ▼
      Sensor-State Model
              │
              ▼
       Steering Error
              │
              ▼
   Differential PWM Control
              │
              ▼
   Autonomous Line Following
```

Original attribution associated with the supplied framework has been retained in the source code.

---

# Technologies

- Arduino
- Embedded C/C++
- Infrared reflectance sensing
- Analog sensor acquisition
- DRV8833 motor driver
- PWM motor-speed control
- Differential-drive robotics
- Autonomous control

Development/testing also included experimentation with an HC-SR04 ultrasonic distance sensor.

---

# Repository Structure

```text
arduino-line-following-robot/
│
├── README.md
├── LICENSE
│
├── src/
│   └── line_following_robot.ino
│
├── tests/
│   ├── motor_drive_test.ino
│   ├── ultrasonic_sensor_test.ino
│   └── ultrasonic_newping_test.ino
│
└── media/
    └── ...
```

The `src` directory contains the final line-following implementation.

The `tests` directory contains development programs used while experimenting with individual hardware components.

---

# IR Sensor System

The line-following controller uses three analog IR sensors:

```cpp
const int irPins[3] = {A5, A4, A3};
```

These represent the left, centre and right sensing positions across the front of the robot.

The controller reads each sensor using:

```cpp
analogRead(irPins[i]);
```

The resulting analog measurements are then compared against a threshold:

```cpp
int threshold = 500;
```

to convert each sensor measurement into a binary state.

Conceptually:

```text
Analog Sensor Reading
        │
        ▼
Compare With Threshold
        │
    ┌───┴───┐
    ▼       ▼
   LOW     HIGH
    │       │
    ▼       ▼
    0       1
```

This converts the raw sensor measurements into a representation that can be used by the steering algorithm.

---

# Three-Bit Line Representation

The individual sensor states are combined into a three-bit value.

```text
Left    Centre    Right
 │        │         │
 ▼        ▼         ▼
 1        1         0

          ↓

         110
```

Different patterns indicate different positions of the track relative to the robot.

For example:

```text
Sensor State        Interpretation

    100          Line towards one side
    110          Small correction required
    010          Robot approximately centred
    011          Small correction required
    001          Line towards opposite side
```

This provides a compact representation of the robot's position relative to the line.

---

# Steering Error

Rather than simply issuing fixed `LEFT`, `RIGHT` and `FORWARD` commands, the controller converts the sensor pattern into a **steering-error value**.

Representative values used by the controller include:

```text
Sensor Pattern       Steering Error

     100                  -80
     110                  -40
     010                    0
     011                  +40
     001                  +80
```

The magnitude of the error represents how strongly the robot needs to correct its heading.

Conceptually:

```text
                 LINE POSITION

 Far Left       Left       Centre       Right       Far Right
    │             │           │           │             │
    ▼             ▼           ▼           ▼             ▼

   -80           -40          0          +40           +80

                    STEERING ERROR
```

This allows different levels of steering correction depending on how far the robot has moved away from the centre of the track.

---

# Differential Steering

The steering error is converted into different PWM commands for the left and right motors.

When the error is positive:

```cpp
leftServoSpeed = maxSpeed;
rightServoSpeed = maxSpeed - error;
```

one side remains at maximum speed while the other is reduced.

For a negative error:

```cpp
leftServoSpeed = maxSpeed + error;
rightServoSpeed = maxSpeed;
```

the opposite motor is reduced.

The resulting behaviour is:

```text
                 Straight

        Left Motor     Right Motor
           FAST           FAST
             \             /
              \           /
                 ROBOT
                   ↑


              Correct Left

        Left Motor     Right Motor
          SLOWER          FAST
             \             /
              \           /
                 ROBOT
                   ↖


             Correct Right

        Left Motor     Right Motor
           FAST         SLOWER
             \             /
              \           /
                 ROBOT
                   ↗
```

This provides smoother steering than simply stopping one motor whenever a correction is required.

---

# Lost-Line Recovery

The controller also includes basic recovery behaviour for situations where none of the sensors detects the line.

This corresponds to:

```text
000
```

Rather than immediately losing all steering information, the program stores the previous steering error.

If the last known error was negative:

```text
error = -130
```

and if the previous error was positive:

```text
error = +130
```

Conceptually:

```text
             Line Detected
                  │
                  ▼
          Store Last Error
                  │
                  ▼
             Line Lost
               (000)
                  │
          ┌───────┴───────┐
          │               │
 Last Error < 0      Last Error > 0
          │               │
          ▼               ▼
 Strong Correction   Strong Correction
   One Direction      Other Direction
          │               │
          └───────┬───────┘
                  ▼
            Search for Line
```

This means the controller retains a small amount of state from the previous sensor reading and uses that information to determine how to respond when the line disappears.

It provides a simple form of **line reacquisition behaviour**.

---

# Main Autonomous Control Loop

The complete navigation loop can be summarised as:

```text
              START
                │
                ▼
        Read IR Sensors
                │
                ▼
       Threshold Readings
                │
                ▼
      Build Sensor Pattern
                │
                ▼
       Determine Position
                │
                ▼
     Calculate Steering Error
                │
                ▼
      Update Motor Speeds
                │
                ▼
          Drive Robot
                │
                ▼
       Store Previous Error
                │
                └──────────► Repeat
```

The repeated sensor-to-actuator cycle allows the robot to continually respond to changes in the position of the track.

---

# Development & Hardware Testing

The repository also contains several smaller development programs used while working with the robot hardware.

These are kept separate from the final line-following controller.

## Motor Drive Test

```text
tests/motor_drive_test.ino
```

This program was used to test basic motor operation and PWM control independently from the complete autonomous controller.

Testing individual subsystems before integration helps separate hardware problems from higher-level navigation problems.

---

## Ultrasonic Sensor Test

```text
tests/ultrasonic_sensor_test.ino
```

A separate HC-SR04 test program was used to experiment with ultrasonic ranging.

The test triggers the sensor, measures the returned echo pulse and converts the measurement into an estimated distance.

The original attribution contained in this example/test code has been retained.

---

## NewPing Ultrasonic Test

```text
tests/ultrasonic_newping_test.ino
```

A second ultrasonic experiment uses the `NewPing` library to obtain distance measurements.

These ultrasonic tests represent development and experimentation with additional sensing hardware.

**Ultrasonic obstacle avoidance is not presented as part of the final line-following implementation in this repository.**

---

# Sensor-to-Actuator Control

One of the key concepts demonstrated by this project is the complete flow of information from environmental sensing to physical robot movement.

```text
            ENVIRONMENT
                │
                ▼
           IR Sensors
                │
                ▼
        Analog Measurements
                │
                ▼
        Sensor Processing
                │
                ▼
        Position Estimate
                │
                ▼
         Steering Error
                │
                ▼
       Motor PWM Commands
                │
                ▼
          Motor Driver
                │
                ▼
            DC Motors
                │
                ▼
          Robot Movement
                │
                ▼
       Changes Sensor Input
                │
                └──────────────►
```

The robot's movement changes its position relative to the line, producing new sensor measurements and causing the control process to repeat.

This provides an early practical example of a **robotic feedback loop**.

---

# Running the Project

Open:

```text
src/line_following_robot.ino
```

in the Arduino IDE.

Select the appropriate Arduino-compatible board and serial port before compiling and uploading the sketch.

The exact sensor threshold and motor characteristics depend on the physical robot, track surface, lighting conditions and sensor positioning.

The value:

```cpp
int threshold = 500;
```

was used by this implementation to distinguish the track from the surrounding surface.

---

# Calibration

Infrared reflectance sensors are affected by environmental conditions including:

- Track colour
- Floor colour
- Ambient lighting
- Sensor height
- Sensor angle
- Surface reflectivity

The threshold therefore needs to be appropriate for the physical environment.

Conceptually:

```text
          Raw Sensor Values
                  │
                  ▼
          Observe Readings
        On Track / Off Track
                  │
                  ▼
        Select Threshold
                  │
                  ▼
         Test on Robot
                  │
                  ▼
        Adjust if Required
```

Motor-speed and steering-error values can similarly be tuned to alter how aggressively the robot corrects its trajectory.

---

# Control Approach

The controller can be viewed as a **discrete error-based steering system**.

It is not a continuous PID controller.

Instead, the three IR sensors divide the line position into a number of discrete states:

```text
Strong       Mild       Centred       Mild       Strong
Correction  Correction                Correction  Correction
    │           │           │             │           │
    ▼           ▼           ▼             ▼           ▼
   -80         -40          0            +40         +80
```

Those errors are then translated into motor-speed differences.

This approach provides more nuanced steering than a simple binary left/right controller while remaining computationally lightweight enough for a small Arduino-based robot.

---

# Known Limitations

The controller was developed as an educational autonomous robotics implementation rather than a production navigation system.

Limitations include:

- Only three IR sensing positions
- Fixed sensor threshold
- Discrete steering-error values
- Open-loop motor-speed control
- No wheel encoders
- No measured wheel velocity
- No odometry
- No PID steering controller
- No localisation
- No environment map
- No path planning
- Sensitivity to lighting and surface conditions

The robot reacts to the track immediately beneath it rather than building a representation of the wider environment.

---

# How I Would Approach It Today

A more advanced version could replace the discrete steering controller with a continuous error estimate and PID control.

For example:

```text
IR Sensor Array
      │
      ▼
Estimate Continuous
Line Position
      │
      ▼
Calculate Error
      │
      ▼
      PID
      │
 ┌────┴────┐
 ▼         ▼
Left PWM  Right PWM
      │
      ▼
    Robot
```

A PID controller could use:

```text
P — Current line-position error

I — Accumulated error over time

D — Rate of change of the error
```

to produce smoother and more adaptive steering behaviour.

---

# Potential Extensions

The project could be developed further with:

- Larger IR sensor array
- Automatic sensor calibration
- PID line-following control
- Wheel encoders
- Closed-loop wheel-speed control
- Intersection detection
- Track-marker recognition
- Ultrasonic obstacle detection
- Obstacle avoidance
- IMU integration
- Bluetooth telemetry
- Data logging
- ESP32 implementation
- ROS / ROS2 integration

A more advanced architecture could combine line following with additional perception:

```text
              Sensor System
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
    IR Array   Ultrasonic     IMU
       │           │           │
       └───────────┼───────────┘
                   ▼
             Robot State
                   │
                   ▼
            Control System
                   │
             ┌─────┴─────┐
             ▼           ▼
          Left Motor  Right Motor
```

---

# Skills Demonstrated

This project provided practical experience with:

### Embedded Programming

- Arduino
- C/C++
- Analog inputs
- Digital logic
- PWM output
- Hardware interfacing

### Robotics

- Autonomous mobile robotics
- Differential-drive control
- Line-following behaviour
- Sensor-actuator integration
- Reactive navigation
- Line-loss recovery

### Control

- Error-based steering
- Differential motor-speed adjustment
- Discrete feedback control
- Controller tuning

### Sensors

- Infrared reflectance sensing
- Sensor thresholding
- Sensor-state encoding
- Experimental ultrasonic ranging

### Engineering Development

- Extending an existing code framework
- Developing additional functionality
- Hardware/software integration
- Subsystem testing
- Parameter tuning
- Physical robot testing

---

# Source Attribution

The project was developed from a stripped-back motor-control framework supplied as part of the original robotics exercise.

The basic framework provided low-level motor-driving functionality. The autonomous sensing and line-following functionality documented in this repository was implemented as part of my project work.

The original source contains attribution to **Garth Zeglin** and the **BSD 3-Clause License**. That attribution has intentionally been retained.

One of the standalone ultrasonic development examples also contains its original third-party attribution, which has likewise been retained.

The repository does not claim authorship of those supplied or third-party components.

---

# Portfolio Context

This project represents an important progression from directly commanding motors to developing a robot capable of continuously modifying its own behaviour from sensor feedback.

Basic Motor Control
        │
        ▼
Sensor Acquisition
        │
        ▼
Line Detection
        │
        ▼
Position Interpretation
        │
        ▼
Steering Error
        │
        ▼
Differential Motor Control
        │
        ▼
Autonomous Navigation

The key development was moving from:

Command → Motor

towards:

Sense → Interpret → Decide → Act → Sense Again

That feedback structure is fundamental to autonomous robotic systems.

Although the controller is deliberately lightweight, the project provided practical experience building the sensing and decision-making functionality required to transform a basic motorised platform into an **autonomous line-following robot**.
