# MSP430 WiFi Autonomous Car

A real-time, microcontroller-based closed-loop control system built on the **TI MSP430FR2355 MCU**. Designed as an **autonomous line-following vehicle**, the car uses an array of infrared (IR) phototransistors to continuously monitor surface reflectivity and navigate a course by tracking a **black line of electrical tape**. 

The system integrates multi-channel analog signal acquisition, deterministic timer interrupt scheduling, dual eUSCI UART ring-buffer wireless telemetry via an ESP32 Wi-Fi module, and dynamic power management.

> **Power Systems & P&C Engineering Relevance:** Although implemented as a mobile robotic vehicle, this architecture directly mirrors the design of an **Intelligent Electronic Device (IED) / Digital Protective Relay**. It continuously polls analog sensor channels (analog input sampling), processes decision logic via deterministic finite state machines (protection logic), controls high-current switching hardware via MOSFET drivers (circuit breaker tripping/actuation), and relays status telemetry over a remote TCP/IP link (SCADA gateway).

---

## 🛠️ P&C Engineering Functional Mapping

| Embedded Hardware / Software Feature | Power Systems & P&C Application Equivalent |
| :--- | :--- |
| **MSP430FR2355 MCU & FRAM Memory** | **Intelligent Electronic Device (IED)** handling real-time deterministic execution and non-volatile event storage. |
| **ESP32 Wi-Fi Module (eUSCI UART)** | **SCADA / Substation Gateway** managing secure remote command parsing and telemetry packet transport. |
| **10/12-bit ADC & IR Phototransistors** | **Analog Signal Conditioning (CT/PT Inputs)** monitoring line parameters and detecting contrast transitions across the course line. |
| **MOSFET H-Bridge Driver Board** | **Breaker Control & Output Tripping Logic** translating low-power MCU logic into high-current DC motor switching. |
| **Timer Interrupts (`Timer0_B0`) & FSM** | **Automated Scheme Logic & Interlocking** maintaining non-blocking, time-synchronized steering control loops. |

---

## 📐 System Architecture & Hardware Design

```mermaid
graph TD
    subgraph POWER_RAIL [Power Distribution Subsystem]
        BATT[4x AA Battery Bank <br/> 6.0V DC Nominal] --> REG[LT1615 Switching Regulator]
        REG --> BUS[3.3V Logic & Power Rail]
    end

    subgraph CORE_LOGIC [Processing Core - Intelligent Electronic Device]
        BUS --> MCU[TI MSP430FR2355 MCU]
        MCU <-->|eUSCI_A0 UART| ESP32[ESP32 Wi-Fi SCADA Gateway]
    end

    subgraph FIELD_IO [Field I/O & Actuation]
        MCU -->|GPIO / PWM Signals| HBRIDGE[MOSFET H-Bridge Driver Board]
        HBRIDGE --> MOTORS[DC Motors - Drive Actuation]
        
        IR[IR Phototransistor Array <br/> Black Tape Line Tracking] -->|Analog Channels A2, A3, A5| MCU
    end

    classDef power fill:#fef3c7,stroke:#d97706,stroke-width:2px,color:#78350f;
    classDef core fill:#e0f2fe,stroke:#0284c7,stroke-width:2px,color:#0369a1;
    classDef io fill:#f1f5f9,stroke:#475569,stroke-width:2px,color:#0f172a;
    
    class BATT,REG,BUS power;
    class MCU,ESP32 core;
    class HBRIDGE,MOTORS,IR io;
