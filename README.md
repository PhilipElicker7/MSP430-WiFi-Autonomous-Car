# MSP430 WiFi Autonomous Car

A real-time, microcontroller-based closed-loop control system built on the **TI MSP430FR2355 MCU**. The system incorporates multi-channel analog signal acquisition, deterministic timer interrupt scheduling, dual eUSCI UART ring-buffer wireless telemetry, and dynamic power management.

> **Power Systems & P&C Engineering Relevance:** Although implemented on a mobile robotic platform, this architecture directly mirrors the design of an **Intelligent Electronic Device (IED) / Digital Protective Relay**. It continuously polls analog sensor channels (analog input sampling), processes decision logic via deterministic finite state machines (protection logic), controls high-current switching hardware via MOSFET drivers (circuit breaker tripping/actuation), and relays status telemetry over a remote TCP/IP link (SCADA gateway).

---

## 🛠️ P&C Engineering Functional Mapping

| Embedded Hardware / Software Feature | Power Systems & P&C Application Equivalent |
| :--- | :--- |
| **MSP430FR2355 MCU & FRAM Memory** | **Intelligent Electronic Device (IED)** handling real-time deterministic execution and non-volatile event storage. |
| **ESP32 Wi-Fi Module (eUSCI UART)** | **SCADA / Substation Gateway** managing secure remote command parsing and telemetry packet transport. |
| **10/12-bit ADC & IR Phototransistors** | **Analog Signal Conditioning (CT/PT Inputs)** monitoring line parameters via continuous channel sampling. |
| **MOSFET H-Bridge Driver Board** | **Breaker Control & Output Tripping Logic** translating low-power MCU logic into high-current DC switching. |
| **Timer Interrupts (`Timer0_B0`) & FSM** | **Automated Scheme Logic & Interlocking** maintaining non-blocking, time-synchronized control loops. |

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
        
        IR[IR Phototransistor Array] -->|Analog Channels A2, A3, A5| MCU
    end

    classDef power fill:#f9f,stroke:#333,stroke-width:2px;
    classDef core fill:#bbf,stroke:#333,stroke-width:2px;
    classDef io fill:#dfd,stroke:#333,stroke-width:2px;
    
    class BATT,REG,BUS power;
    class MCU,ESP32 core;
    class HBRIDGE,MOTORS,IR io;
