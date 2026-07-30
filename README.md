# Wireless Distributed Control & Autonomous Embedded System

A real-time, micro-controller-based closed-loop control system built on the **TI MSP430FR2355 MCU**[cite: 1]. The system incorporates multi-channel analog signal acquisition[cite: 1], deterministic timer interrupt scheduling[cite: 1], dual eUSCI UART ring-buffer wireless telemetry[cite: 1], and dynamic power management[cite: 1].

> **Power Systems & P&C Engineering Relevance:** Although implemented on a mobile robotic platform, this architecture directly mirrors the design of an **Intelligent Electronic Device (IED) / Digital Protective Relay**. It continuously polls analog sensor channels (analog input sampling)[cite: 1], processes decision logic via deterministic finite state machines (protection logic)[cite: 1], controls high-current switching hardware via MOSFET drivers (circuit breaker tripping/actuation)[cite: 1], and relays status telemetry over a remote TCP/IP link (SCADA gateway)[cite: 1].

---

## 🛠️ P&C Engineering Functional Mapping

| Embedded Hardware / Software Feature | Power Systems & P&C Application Equivalent |
| :--- | :--- |
| **MSP430FR2355 MCU & FRAM Memory**[cite: 1] | **Intelligent Electronic Device (IED)** handling real-time deterministic execution and non-volatile event storage[cite: 1]. |
| **ESP32 Wi-Fi Module (eUSCI UART)**[cite: 1] | **SCADA / Substation Gateway** managing secure remote command parsing and telemetry packet transport[cite: 1]. |
| **10/12-bit ADC & IR Phototransistors**[cite: 1] | **Analog Signal Conditioning (CT/PT Inputs)** monitoring line parameters via continuous channel sampling[cite: 1]. |
| **MOSFET H-Bridge Driver Board**[cite: 1] | **Breaker Control & Output Tripping Logic** translating low-power MCU logic into high-current DC switching[cite: 1]. |
| **Timer Interrupts (`Timer0_B0`) & FSM**[cite: 1] | **Automated Scheme Logic & Interlocking** maintaining non-blocking, time-synchronized control loops[cite: 1]. |

---

## 📐 System Architecture & Hardware Design

```
+-----------------------------------------------------------------------+
|                            DC POWER SUPPLY                            |
|             4x AA Batteries (6.0V Nominal DC Power System)            |
+-----------------------------------+-----------------------------------+
                                    |
                                    v
+-----------------------------------+-----------------------------------+
|                     POWER & VOLTAGE REGULATION                        |
|        LT1615 Switching Regulator & Power Board (3.3V Logic Rail)     |
+------------------+--------------------------------+-------------------+
                   |                                |
                   v                                v
+------------------+--------------+   +-------------+-------------------+
|      ESP32 IOT MODULE           |   |      MSP430FR2355 MCU            |
|  (TCP/IP Server / SCADA Link)   |   |   (Central Controller / IED)    |
+------------------+--------------+   +-------------+-------------------+
                   |                                |
            UART (eUSCI_A0)           ADC Channels  | GPIO / PWM Drive
                   |                  (A2, A3, A5)  | Signals
                   v                                v
+------------------+--------------+   +-------------+-------------------+
|      SERIAL RING BUFFER         |   |    IR SENSING & MOTOR DRIVERS    |
|   (Asynchronous Telemetry)      |   |   (Optics & MOSFET H-Bridge)    |
+---------------------------------+   +---------------------------------+
```

### 1. DC Power Subsystem & Efficiency Analysis
* **Primary Source:** $6.0\text{ V DC}$ nominal supply ($4 \times 1.5\text{ V}$ AA battery bank)[cite: 1].
* **Regulation:** Board-level LT1615 switching converters and decoupling networks step voltage down to a stabilized $3.3\text{ V}$ logic bus (`PWR3_3`)[cite: 1].
* **Power Load Profile:**
  $$\text{Usable Capacity} \approx 6.0\text{ V} \times 2.0\text{ Ah} \times 0.85 = 10.2\text{ Wh}$$
  Under maximum drive load ($\sim 10\text{ W}$ total system draw with LCD backlight and high-current motor actuation), continuous operational endurance is calculated at approximately **1.0 hour**[cite: 1].

### 2. Sensor Signal Processing (CT/PT Analog Equivalent)
The optical sensing array utilizes an IR emitter diode teamed with left/right phototransistors to measure surface contrast[cite: 1]. The raw voltages are digitized using the MSP430's integrated ADC on pins `A2` and `A3`[cite: 1]:
* **Sampling Rate Optimization:** Driven by `Timer_B1` interrupts operating on a $20\text{ ms}$ polling cycle to prevent aliasing and eliminate missed line-crossing events[cite: 1].
* **Noise Reduction:** Raw 12-bit ADC values ($0\text{--}4095$) are scaled and filtered to manage noise margins without blocking main execution thread[cite: 1].

### 3. Actuation & Drive Controls (Breaker Control Logic)
High-current directional control is managed using an external **MOSFET H-Bridge board**[cite: 1]. Low-power GPIO drive outputs from Port 6 (Pins `P6.1`–`P6.4`) modulate P-Channel and N-Channel MOSFET gates[cite: 1]. Software-defined PWM duty cycles regulate differential wheel velocity to enforce precise trajectory control[cite: 1].

---

## 💻 Software Architecture & Flowcharts

The software architecture is modularized into non-blocking interrupt service routines (ISRs) and state machines[cite: 1].

### Continuous ADC Channel Sampling Loop
To ensure deterministic execution, the ADC utilizes channel sequence interrupts to continuously cycle through inputs without CPU blocking[cite: 1]:

```
    [ START ADC CONVERSION ]
               |
               v
  [ ISR Triggered: Conversion Done? ]
               |
        +------+------+
        |             |
     (Case 0)      (Case 1)
        |             |
   Read Left IR  Read Right IR
   Data (0x02)   Data (0x03)
        |             |
   Switch to A3  Switch to A5
        |             |
        +------+------+
               |
               v
   [ Restart ADC Conversion Loop ]
```

```c
// Abstracted ISR snippet showing non-blocking multi-channel ADC sequence
switch (ADC_Channel++) {
    case 0x00: // Channel A2: Left Optical Sensor
        ADC_Left_Detect = ADCMEMO;
        ADCMCTL0 &= ~ADCINCH_2;
        ADCMCTL0 |=  ADCINCH_3; // Reconfigure for Channel A3
        break;
    case 0x01: // Channel A3: Right Optical Sensor
        ADC_Right_Detect = ADCMEMO;
        ADCMCTL0 &= ~ADCINCH_3;
        ADCMCTL0 |=  ADCINCH_5; // Reconfigure for Channel A5
        break;
        ...
}
```

### Deterministic Interrupt Scheduler (`Timer0_B0`)
System timing is maintained via `Timer0_B0` ISR running on a precise **$50\text{ ms}$ tick rate** (`TB0CCR0_INTERVAL = 25000`)[cite: 1]:

```
               [ Timer0_B0 ISR (50ms Interval) ]
                                |
        +-----------------------+-----------------------+
        |                       |                       |
        v                       v                       v
[ Switch Debounce ]    [ Sensor Acquisition ]   [ Wheel Control State ]
Tracks 20 cycles for   Polls updated ADC values Updates differential 
1.0 sec debounce logic. every 50ms interval.    drive states every 50ms.
```

### Telemetry & Serial Communication (SCADA Protocol)
Remote communication uses dual circular ring buffers across `eUSCI_A0` (ESP32 Wi-Fi) and `eUSCI_A1` (USB Debug)[cite: 1]. Incoming UART bytes trigger receive interrupts (`RXIFG`), storing data in `PC_2_IOT` arrays without dropping packets during heavy processing loads[cite: 1].

```c
// Circular Buffer RX Interrupt Handler Excerpt
#pragma vector = EUSCI_A1_VECTOR
interrupt void eUSCI_A1_ISR(void) {
    switch (_even_in_range(UCA1IV, 0x08)) {
        case 2: // Vector 2: RXIFG
            temp = usb_rx_wr++;
            PC_2_IOT[temp] = UCA1RXBUF;
            if (usb_rx_wr >= sizeof(PC_2_IOT)) {
                usb_rx_wr = 0; // Wrap around circular buffer
            }
            UCA0IE |= UCTXIE; // Trigger TX forward interrupt
            break;
    }
}
```

---

## 📷 System Gallery & Hardware Construction

> *Add high-resolution photos of your 3D-printed vehicle, custom power board layout, LCD operation, and oscilloscope waveform captures below.*

| Vehicle Chassis & Top View | Custom Power & Driver Board |
| :---: | :---: |
| *(Insert Image Link Here)* | *(Insert Image Link Here)* |

| LCD Screen Status Telemetry | Line-Following IR Sensor Mounting |
| :---: | :---: |
| *(Insert Image Link Here)* | *(Insert Image Link Here)* |

---

## 📜 Academic Integrity & License Compliance

**Notice:** To comply with NC State University ECE Academic Integrity standards, complete implementation source files (`main.c` lab-specific drivers) are withheld from this public repository. The provided modules showcase abstract architecture patterns, state machine designs, and general firmware stubs. 

*Full source code access can be made available to recruiters and industry professionals upon request.*
