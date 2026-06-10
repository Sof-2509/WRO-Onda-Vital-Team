

## Content

* `t-photos` contains 2 photos of the team (an official one and one funny photo with all team members)
* `v-photos` contains 6 photos of the vehicle (from every side, from top and bottom)
* `video` contains the video.md file with the link to a video where driving demonstration exists
* `schemes` contains one or several schematic diagrams in form of JPEG, PNG or PDF of the electromechanical components illustrating all the elements (electronic components and motors) used in the vehicle and how they connect to each other.
* `src` contains code of control software for all components which were programmed to participate in the competition
* `models` is for the files for models used by 3D printers, laser cutting machines and CNC machines to produce the vehicle elements. If there is nothing to add to this location, the directory can be removed.
* `other` is for other files which can be used to understand how to prepare the vehicle for the competition. It may include documentation how to connect to a SBC/SBM and upload files there, datasets, hardware specifications, communication protocols descriptions etc. If there is nothing to add to this location, the directory can be removed.

the Project
===
This project consists of an autonomous land vehicle capable of navigating an environment with colored obstacles. Depending on the detected color, the vehicle executes a predefined evasive maneuver (clockwise or counterclockwise turn).

The system is controlled by an Arduino Mega microcontroller, which processes all sensors and actuators in real time. The Arduino Uno provides sufficient digital and analog I/O pins to interface with the distance sensors, camera module, and motor driver for basic robotics applications.

For distance detection, the vehicle uses four VL53L1X ToF (Time-of-Flight) sensors, mounted on the front, rear, and sides. These sensors communicate with the Arduino Mega via the SDA/SCL pins. When an obstacle is detected within a predetermined distance (between 15 and 30 cm), the sensor sends a signal to the Arduino.

For color detection, the vehicle is equipped with a HuskyLens AI camera module. It recognizes trained color signatures (red, blue, green, yellow, etc.) and sends the color information to the Arduino Mega via UART (serial communication).

Based on the color received, the Arduino directs the motion system to turn clockwise or counterclockwise. Propulsion is provided by a Feetech STS3215 smart servomotor, while steering is handled by a standard servomotor. The STS3215 is controlled through a URT-1 motor driver.

The vehicle's chassis was custom-designed using 3D modeling software and fabricated with a 3D printer.


## the Challenge
Teams are challenged to create, assemble, and program a robotic car that can drive itself around a racetrack that is dynamically altered for every round. there are two primary objectives in the competition: finishing laps with randomized obstacles and pulling off a flawless parallel parking manoeuvre. Teams must incorporate cutting-edge robotics ideas with an emphasis on innovation and dependability, such as computer vision, sensor fusion, and kinematics.

This challenge emphasizes all aspects of the engineering process, including:

- Mobility Management: Developing efficient vehicle movement mechanisms.
- Obstacle Handling: Strategizing to detect and navigate traffic signs (red and green markers) within specified rules.
- Documentation: Showcasing engineering progress, design decisions, and open-source collaboration through a public GitHub repository.

## Photos of ALPHA
| Front view | Back view | Left view | 
| ------------- |------------- | ------------- |
| | ![DELTA Back](https://github.com/user-attachments/assets/52d4785b-2a04-40ab-bfa1-ace6d514cec9)| ![DELTA Left](https://github.com/user-attachments/assets/4ff15c7c-de28-4c58-83e0-6b2941a30886)|

| Right view | Top view | Bottom view |
| ------------- | ------------- |------------- |
|![DELTA Right](https://github.com/user-attachments/assets/4fad3d60-001c-4e51-b5a7-5031d2bf155f)| ![DELTA Top](https://github.com/user-attachments/assets/2741638f-efc4-4f0a-b089-daeeddca627b) |![DELTA Bottom](https://github.com/user-attachments/assets/dbf5b66c-b878-4635-9ba1-8910e8f668d6) |



## Management

### Mobility Management

**STS3215 servo (7.4V):**
| Specifications: |
| ------------- |
| Voltage: 12V |
| Gear Ratio: 1:345 |
| Speed (at 7.4V): 53 RPM |
| Torque: 5. 0 kg·cm |
| Weight: 55g ± 1g |
| Encoder: Magnetic, 4096 counts per revolution |
|<img width="600" height="600" alt="407893fdcce8f4cda1cb6a1fcec4 jpg" src="https://github.com/user-attachments/assets/e1d177c3-8ff5-417e-9c58-2843a8215edc" />|


The propulsion system uses a smart serial servo that combines a powerful motor, a heavy-duty gearbox, and a high-precision magnetic encoder into a single unit. This setup was chosen over a traditional DC motor because the smart servo handles its own speed and position control internally, freeing up the ESP32 microcontroller to focus entirely on navigation. It also simplifies the vehicle's design by replacing messy wiring with a clean, 3-wire system where multiple servos connect in a single chain, drastically reducing clutter and potential loose connections.

Additionally, the built-in encoder is absolute, meaning the vehicle instantly knows the exact angle of its wheels the moment it turns on without needing any movement or calibration. The servo also constantly monitors its own temperature and electrical load, allowing the system to detect obstacles and prevent the motors from overheating or taking damage. Ultimately, these major benefits in processing efficiency, space saving, and built-in safety are the exact reasons we chose this smart servo architecture instead of a standard DC motor setup.


**Servo Motor MG90s:**
| Specifications: |
| ------------- |
| Voltage: 4.8-6V |
| Torque: 1.8/2.2 kg/cm |
| Rotation: 360° ± 10°  |
<img width="550" height="550" alt="servomg90s" src="https://github.com/user-attachments/assets/8e756081-108b-4ebe-bd81-def2a431791c" />


A servomotor is integrated into the system to provide continuous rotational control for the propulsion or drive mechanism. The servo receives PWM signals from a dedicated ESP32 pin, allowing precise speed and direction control rather than angular positioning. Unlike standard 180° servos, this 360-degree continuous rotation version does not stop at fixed angles; instead, it rotates freely in either direction, with the PWM signal determining the rotation speed (from full speed forward to full speed reverse) and a specific neutral pulse width commanding a full stop.

We chose the MG90S 360-degree servo for its compact size, metal gears, and continuous rotation capability, which offers greater durability and torque than standard plastic-geared servos. It provides smooth variable-speed control in both directions, is easy to implement using standard Arduino servo libraries (by adapting the write() function to set speeds), and is widely available at a low cost. This makes it ideal for applications requiring wheeled movement, conveyor belts, or other continuous motion tasks, while still benefiting from the reliability and straightforward interfacing common to the MG90S family.


**FE-URT-1:**
| Specifications: |
| ------------- |
| Power supply voltage: 5 to 24V  |
| Output Signal:  TTL/RS485 (3.3V or 5V switchable) |
| Can handle:  Feetech SMS series (RS485) and SCS series (TTL) smart servo |
<img width="800" height="517" alt="urt 1" src="https://github.com/user-attachments/assets/a6ab706b-0f16-4437-95d4-46cb8fd4bc56" />

The FE-URT-1 serves as a specialized, hardware-driven communication bridge that allows an Arduino Mega to effortlessly command and read real-time telemetry from complex servo arrays. While the Arduino Mega communicates using separate transmit ($TX$) and receive ($RX$) lines, smart serial servos operate on a single-wire, half-duplex bus; the FE-URT-1 seamlessly bridges this gap using automatic hardware time-sharing circuitry to flip data directions instantly without taxing the Arduino’s processor. This solution vastly outperforms standard PWM controllers or DIY logic shifters by eliminating massive wiring clutter in favor of a clean, daisy-chained connection, allowing the Mega to treat the entire network as a standard plug-and-play serial port. Furthermore, its unique dual-signal architecture simultaneously drives both high-torque RS485 (SMS series) and TTL-level (SCS series) servos on the same bus, providing critical two-way feedback—such as position, current, voltage, and temperature—that blind, traditional controllers simply cannot capture.
Despite these advantages, the board has distinct hardware constraints that require careful integration. Its onboard power tracks have a strict threshold of 6A, meaning any high-load or multi-servo setup will require you to bypass the board's power terminals and route an external power harness directly to the motors to prevent overheating the PCB. Additionally, because SMS and SCS servos often operate at different voltages (typically 12V versus 7.4V), the FE-URT-1 does not provide native voltage regulation; mixing these servo classes on a shared power rail without external buck regulators will permanently damage the lower-voltage components. Finally, the board relies on an outdated Mini-USB port for debugging and uses proprietary JST-style connectors, meaning field maintenance requires keeping a dedicated stock of spare cables on hand.


### Power and Sense Management

**Arduino Mega 2560:**

|Specifications:|
| ------------ |
|Microcontroller: ATMega2560 |
|Flash memory: 256 KB (of which 8 KB is used by the bootloader) |
|SRAM: 8 KB | 
|Frequency: 16 MHz |
|Input voltage: 7 V – 12 V |
|Output voltage: 5V |
|Pins: 54 Digital I/O (15 provide PWM output), 16 Analog Inputs |


The Arduino Mega 2560 is a microcontroller board based on the ATmega2560 chip, featuring an 8-bit AVR processor running at a stable 16 MHz clock speed. It offers an extensive array of 54 digital input/output pins (15 of which provide PWM outputs), 16 analog inputs, 4 hardware serial ports (UARTs), a standard USB connection, a power jack, and an onboard reset button. The Arduino Mega runs the core control loop that enables us to accomplish the challenge, simultaneously managing diverse hardware interfaces and processing incoming sensor data to execute real-time robot movements.

We selected the Arduino Mega 2560 for its massive number of I/O pins, robust 5V operation, and abundance of dedicated hardware serial ports, which allow us to easily isolate and manage complex serial communications—such as bridging data to the FE-URT-1 servo board and the HuskyLens—without the need for unstable software-emulated serial lines. However, its notable limitations include a low 16 MHz clock speed and a highly restricted 8 KB of SRAM, which can quickly bottleneck advanced data processing or large arrays compared to modern 32-bit controllers. Additionally, its large physical footprint occupies considerable chassis space, its lack of built-in Wi-Fi or Bluetooth requires external modules for wireless telemetry, and the lower 40mA current limit per I/O pin means it cannot drive power-hungry components directly.




**Tof Sensor (VL53L1X):**
| Specifications: |
| ------------- |
| Accuracy:  ±5% |
| Measurement Range: 0 a 400 cm |
| Resolution:  0.1 cm |
| Input voltage: 3.3V to 5V  |
| Operating Current: 40mA |
|<img width="431" height="350" alt="TOF SENSOR" src="https://github.com/user-attachments/assets/42626d9d-79f4-477b-a87f-266f059817c6" />|



It is a sensor that utilizes Time-of-Flight technology to measure the travel time of an infrared laser pulse from the emitter to a target and back. Using the ESP32 devkit v1, we can precisely calculate the distance based on the constant speed of light, performing the function of detecting nearby obstacles or walls and executing the required navigation turns. This sensor was chosen for its exceptional accuracy, compact size, and immunity to object color or texture, outperforming traditional ultrasonic or infrared alternatives that are often less reliable in varied environmental conditions.

**Li-ion 7.4V Battery:**
| Specifications: |
| ------------- |
| Capacity: 3000 mAh |
| Voltage: 3.7 V |
| Discharge rate:  0.25C |
| Weight: 48 g (one battery) |
| Size: 18 mm diameter, 65 mm length |


**9V Akaline Battery:**
| Specifications: |
| ------------- |
| Capacity: 3000 mAh |
| Voltage: 3.7 V |
| Discharge rate:  0.25C |
| Weight: 48 g (one battery) |
| Size: 18 mm diameter, 65 mm length |



**HuskyLens:**
| Specifications: |
| ------------- |
| Processor: Kendryte K210 (Dual-core RISC-V 64bit, 400 MHz) |
| Image sensor: Kendryte K210 (Dual-core RISC-V 64bit, 400 MHz) |
| Resolution: 320×240 (Screen) / Up to 640×480 (Image output)  |
| Lens field-of-view: 65.0° horizontal, 47.2° vertical |
| Power consumption: ~230mA (at 5V with backlight) |
| RAM: 8MB (6MB SRAM for CPU + 2MB KPU AI hardware accelerator) |
| Flash Memory: 16MB |
| Power input:  3.3V to 5V |
|<img width="335" height="150" alt="huskey lens img" src="https://github.com/user-attachments/assets/031f09ce-61b5-44ea-840f-0203b420a9c9" />|

The HuskyLens is an easy-to-use AI machine vision sensor that utilizes an onboard Kendryte K210 dual-core processor to run deep learning algorithms locally, allowing a microcontroller to implement advanced vision features via standard I2C or UART without any machine learning or OpenCV programming. By simply pointing the camera at an object and clicking its physical learning button, users can instantly train the module to handle complex real-time tasks like object tracking, face recognition, line following, color classification, and AprilTag decoding, completely freeing the main controller from heavy processing overhead. However, the sensor has distinct drawbacks: its fixed firmware restricts users from uploading custom neural network models, its detection reliability drops noticeably under erratic ambient lighting conditions, and its high power consumption—driven by continuous onboard image processing and a built-in IPS screen—can easily cause 
system brownouts if it is not supplied by a dedicated, stable external power source rather than a standard microcontroller pin.

We switched from the PixyCam v2 to the DFRobot HuskyLens because it upgrades the project from basic color tracking to real Artificial Intelligence (AI).
While the PixyCam only tracks colors, the HuskyLens uses a powerful AI chip and neural networks to perform advanced tasks like face recognition, object tracking, and tag/QR code reading. Additionally, its massive memory upgrade (8MB RAM / 16MB Flash) allows it to process data incredibly fast, and its built-in 2.0-inch screen lets you train and debug the camera instantly without needing a computer. It is simply a much smarter, more versatile sensor for robotics.


**Voltaje Regulator (Lm2596) :**
| Specifications: |
| ------------- |
| Input Voltage: 4V to 35V |
| Output Voltage:  1.23V to 37V |
| Energy eficiency : 80% |
![voltage regulator](https://github.com/user-attachments/assets/7e0a2466-8ae1-4540-9adb-790cd3f9cec8)

The LM2596 is a monolithic step-down (buck) switching regulator that efficiently reduces a higher DC input voltage to a lower, stable DC output using Pulse Width Modulation (PWM). Unlike traditional linear regulators that waste excess voltage by converting it entirely into heat, the LM2596 operates as a rapid electronic switch driven by an internal 150 kHz oscillator. It chops the incoming DC voltage into high-frequency square waves, passes them through a power inductor and capacitor network (an LC filter), and smooths them back out into a steady, lower voltage. This switching method allows the module to routinely achieve power efficiencies between 73% and 90%, making it an ideal choice for powering voltage-sensitive components—such as driving lower-voltage serial servos from a main 12V power rail—without generating massive amounts of waste heat or requiring a bulky heatsink.

However, the module has distinct drawbacks and design constraints that must be accounted for in a robust system. Because it operates at a fixed 150 kHz switching frequency, which is relatively slow compared to modern megahertz-range regulators, it requires physically larger inductors and capacitors to filter out electrical noise, resulting in a bulkier hardware footprint. Additionally, the LM2596 is a non-synchronous regulator that relies on an external Schottky catch diode to complete the circuit when the internal switch is off; this diode introduces a constant voltage drop that reduces overall efficiency under heavy current loads near its 3A maximum limit. Finally, due to the immense popularity of this module, the market is heavily flooded with counterfeit and clone ICs that often run at sub-standard frequencies, overheat quickly, and exhibit high voltage ripple, making it critical to source the components from reputable suppliers to protect your microcontroller and hardware components.
