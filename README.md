

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
Our project consists of creating an automated land vehicle capable of navigating an environment marked by colored obstacles. the vehicle takes different paths depending on the color of the obstacle. We use an ESP32-based program (C++) that includes codes for the ultrasonic sensors (hc-sr04), which can detect objects at a predetermined distance. When an obstacle is detected, the sensors send a signal to the ESP32 circuit board, which then directs the vehicle’s movement system to turn det either clockwise or counterclockwise, as pre-established. Additionally, the vehicle is equipped with a camera module (Pixy v2) to detect colored obstacles and avoid them based on their color

the core of the system is based on the ESP32 v1 devkit microcontroller, which provides a versatile platform with sufficient input/output pins and processing capability to manage multiple sensors and actuators for basic robotics applications. the vehicle features a DC motor equipped with an encoder, allowing for precise feedback on wheel rotation to enable closed-loop speed and distance control. Four ultrasonic sensors are strategically mounted to the vehicle, giving comprehensive environmental awareness by measuring distances to obstacles in front and on both sides. A motor driver shield is used to manage power delivery and control signals to the motor, while a servomotor handles steering actuation. the chassis for this vehicle was custom-designed using 3D modeling software and subsequently fabricated with a 3D printer.



## the Challenge
Teams are challenged to create, assemble, and program a robotic car that can drive itself around a racetrack that is dynamically altered for every round. there are two primary objectives in the competition: finishing laps with randomized obstacles and pulling off a flawless parallel parking manoeuvre. Teams must incorporate cutting-edge robotics ideas with an emphasis on innovation and dependability, such as computer vision, sensor fusion, and kinematics.

This challenge emphasizes all aspects of the engineering process, including:

- Mobility Management: Developing efficient vehicle movement mechanisms.
- Obstacle Handling: Strategizing to detect and navigate traffic signs (red and green markers) within specified rules.
- Documentation: Showcasing engineering progress, design decisions, and open-source collaboration through a public GitHub repository.

## Photos of ALPHA




## Management

### Mobility Management

**Motor DC 12v with encoder:**
| Specifications: |
| ------------- |
| Voltage: 12V |
| Gear Ratio: 1:34 |
| Speed (at 12V): 126 RPM |
| Torque: 4.2 kg·cm |
| Weight: 86g |
| Encoder: Optical, 12 counts per revolution |
![encoder2](https://github.com/user-attachments/assets/47492287-88c9-4dba-ba15-793ce49024c4)



the propulsion system relies on a DC motor paired with an encoder. the motor provides the mechanical force required to move the vehicle, while the encoder outputs two pulse signals (c!
hannels A and B) that indicate the rotation direction and speed by measuring the number of pulses per revolution.
We used these motors for their balance between power, efficiency, and cost, as well as their easy integration with common motor drivers like the L298N.


**Servo Motor MG995:**
| Specifications: |
| ------------- |
| Voltage: 4.8-6V |
| Torque: 9.4/11 kg/cm |
| Rotation: 180° ± 10°  |
![617BvnN0VJL _SL1500_](https://github.com/user-attachments/assets/b501ca45-12d2-46de-9c1c-de0f9dc0fb90)


A servomotor is integrated into the system to control steering. the servo receives PWM signals from a dedicated ESP32 pin, allowing precise angular positioning between 0° and 180°. the servo’s position is controlled programmatically to perform smooth and accurate movements.
We chose this servo for its precise position control, ease of use with Arduino libraries, and widespread availability, offering better accuracy and reliability than cheaper or less-documented servos.

**Module l298n motor driver:**
| Specifications: |
| ------------- |
| Power supply voltage: 5 to 35V  |
| Output current:  0-36mA |
| Can handle:  DC motors or stepper motors |
![modulo-l298n-driver-control-motor-puente-h-arduino-0](https://github.com/user-attachments/assets/50e4565b-47fe-46c2-a331-fc19f75f6467)

the L298N is a robust dual H-bridge motor driver IC capable of controlling two DC motors simultaneously or one stepper motor. Each channel of the L298N can deliver up to 2A continuous current (with adequate heat dissipation) and operates within a voltage range of 5V to 35V, making it suitable for a wide range of motors.
This motor driver module interfaces easily with the ESP32 microcontroller, accepting PWM signals from the ESP32 for precise speed control and digital signals for direction control, enabling both forward and reverse movement. the L298N module also includes built-in protection mechanisms such as current sensing and thermal shutdown to safeguard the system during operation.
the L298N was selected for this project due to its ability to control two motors at once, its compatibility with the ESP32.

### Power and Sense Management

**ESP32 DevKit v1:**

|Specifications:|
| ------------ |
|Microcontroller: ESP32 |
|Flash memory: 4 Mb |
|SRAM: 520 Kb | 
|Frequency: 2.4 GHz to 2.5 GHz |
|Input voltage: 2.7 – 3.6V |
|Pins: 30 |
![ModuloESP32-DEVKITV1-30pines-min_1_2048x2048](https://github.com/user-attachments/assets/efbd9da1-9a60-4602-9d2a-44219e70ab41)

ESP32 DevKit V1:
the ESP32 DevKit V1 is a microcontroller board based on the ESP32 SoC, featuring a dual-core Tensilica processor running at up to 240 MHz. It offers integrated Wi-Fi and Bluetooth, multiple digital input/output pins, analog inputs, PWM outputs, a micro-USB connection, onboard voltage regulator, and reset/boot buttons. the ESP32 board runs the code that enables us to accomplish the challenge, processing sensor data to perform the required movements.

We selected the ESP32 DevKit V1 for its wireless connectivity, higher processing power, and compatibility with a wide range of sensors and actuators. Its user-friendly development environment, strong community support, and extensive documentation make it easier to implement complex robotics tasks compared to less-documented microcontroller boards.


**ESP32 Expansion Base board v1:**

|Specifications:|
| ------------ |
|Power supply: 5-12V|
|On-board Regulator : 5V~1A |
|SRAM: 520 Kb | 
|Frequency: 2.4 GHz to 2.5 GHz |
|Input voltage: 2.7 – 3.6V |
![expansion board](https://github.com/user-attachments/assets/0ba4f354-1d29-45aa-8afe-bd514cc25211)

the ESP32 Expansion Board V1 30P is an add-on board designed to extend the functionality of ESP32 development boards, featuring 30 accessible pins for easy connection to various peripherals. It provides organized access to multiple digital and analog inputs/outputs, power rails, and communication interfaces such as I2C, SPI, and UART. the expansion board simplifies prototyping by offering clearly labeled headers, additional power regulation options, and secure mounting for the ESP32 module, enabling robust connections for sensors, actuators, and external modules.

We selected the ESP32 Expansion Board V1 30P for its convenient pin access, stable power distribution, and compatibility with a broad range of hardware components. Its well-organized layout, ease of integration, and community support streamline the development of complex robotics solutions, making it preferable to less modular expansion boards or direct wiring, which can lead to unreliable connections and increased troubleshooting.


**Ultrasonic Sensor (hc-sr04):**
| Specifications: |
| ------------- |
| Accuracy:  ±3% |
| Measurement Range: 2 a 450 cm |
| Resolution:  0.3 cm |
| Input voltage: 5V  |
| Operating Current: 15mA |
![Ultrasonic_Sensor](https://github.com/user-attachments/assets/ff7ceb5b-2965-475b-acc6-ad80540a37db)


It is a sensor that uses ultrasonic sounds to detect the bounce time of sound from one side to the other. Using the ESP32 devkit v1 we can determine the distance based on the time it takes for the wave to return, performing the function of determining when there is a wall nearby, and thus making the corresponding turn.
This sensor was chosen for its accuracy, low cost, and readily available documentation, outperforming other more expensive or less reliable distance-sensing options.

**18650 Battery:**
| Specifications: |
| ------------- |
| Capacity: 3000 mAh |
| Voltage: 3.7 V |
| Discharge rate:  0.25C |
| Weight: 48 g (one battery) |
| Size: 18 mm diameter, 65 mm length |
![image](https://github.com/user-attachments/assets/09afb713-b1cb-4bf8-b15b-2e0a06950587)

we are going to use 3 of these li-ion batterys in parallel to power the vehicle, with an 18650 battery holder. This battery configuration provides a reliable and efficient power source for the robot’s electronics and actuators, supporting extended operation during competition or testing.


**PixyCam v2:**
| Specifications: |
| ------------- |
| Processor: NXP LPC4330 |
| Image sensor: Aptina MT9M114 |
| Resolution:1296×976  |
| Lens field-of-view: 80° horizontal, 40° vertical |
| Power consumption: 140mA |
| RAM: 264K bytes |
| Flash Memory: 2M bytes |
| Power input: 5V |
![pixy-v21-camera-sensor](https://github.com/user-attachments/assets/74f57132-97c9-4abd-84b3-dc63150acd27)

the camera is capable of detecting seven colors simultaneously and It is equipped with an internal processor, which lets us explore just the necessary information for the ESP32 to evade in the necessary way, depending on the obstacle colour.
 We selected this camera for its high-resolution image capture, easy interfacing with microcontrollers, and strong community support, providing superior performance and documentation compared to less common camera modules.

**Voltaje Regulator (Lm2596) :**
| Specifications: |
| ------------- |
| Input Voltage: 4V to 35V |
| Output Voltage:  1.23V to 37V |
| Energy eficiency : 80% |
![voltage regulator](https://github.com/user-attachments/assets/7e0a2466-8ae1-4540-9adb-790cd3f9cec8)

The module LM2596 is a regulator step down that can reduce de voltage of input to a lower voltage in output 
