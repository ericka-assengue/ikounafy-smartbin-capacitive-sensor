<img width="179" height="93" alt="image" src="https://github.com/user-attachments/assets/67e1e8b0-1d1f-423d-a548-3a19418294b1" /># Ikounafy Smartbin — Capacitive Fill-Level Sensing for Waste Bins

<img width="1200" height="1600" alt="photo ikounafy" src="https://github.com/user-attachments/assets/2245e0bd-5bae-421c-b305-a90fc3de2a8b" />

An embedded waste monitoring system that measures a bin's fill level using
capacitive sensing and streams it in real time to a web dashboard — so
collection crews know which bins actually need emptying, and which don't.

Developed as part of an engineering project at École des Mines de
Saint-Étienne (ISMIN), with Mariem Boussarsar.


## Key Features

- Non-contact capacitive fill-level sensing — electrodes never touch the
  waste directly, unlike immersed-probe or weight-based alternatives
- Frequency measurement via STM32 hardware Timer Input Capture rather than
  software polling — precise, non-blocking
- Custom analog conditioning circuit (relaxation oscillator, Schmitt
  trigger, V→I converter) designed on a custom KiCad PCB
- COMSOL electrostatic simulation used to validate the sensing principle
  before building any hardware
- Serial-to-cloud gateway (Python) forwarding readings to a live web
  dashboard, so fill levels are visible remotely in real time

## Motivation


Inefficient waste collection is a well-documented challenge, not just an intuition. According to the World Bank's *What a Waste 2.0* report, around **29% of municipal solid waste worldwide remains uncollected**, while **collection and transportation account for 60–80% of waste management costs** in low- and middle-income countries. A significant part of this inefficiency stems from a lack of visibility: collection crews often have no way of knowing which bins actually need emptying without inspecting each one individually.

Our smartbin, **Ikounafy** was developed to address this challenge by continuously estimating a bin's fill level using non-contact capacitive sensing and making this information available through a live dashboard. By replacing fixed collection schedules with real-time data, the system demonstrates how embedded systems and IoT technologies can support smarter, more efficient waste collection.

> **Source:** Kaza, S., Yao, L., Bhada-Tata, P., & Van Woerden, F. (2018). *What a Waste 2.0: A Global Snapshot of Solid Waste Management to 2050*. World Bank.


## How it works

1. A relaxation-oscillator circuit uses the waste bin itself as one plate of
   a capacitor (lid electrode + base electrode). As the bin fills, the
   dielectric between the plates changes, which changes the capacitance —
   and therefore the oscillation frequency of the conditioning circuit.
2. The STM32 (STM32F301K8Tx) reads this frequency using **Timer 2 in Input
   Capture mode** (`PA0` / `TIM2_CH1`): it timestamps two consecutive rising
   edges and computes the frequency from the time difference.
3. The firmware averages 30 consecutive measurements to reduce noise, then
   maps the result to a 0–100% fill level using a linear interpolation
   between two calibrated reference points:
   - Empty bin: 65500 Hz
   - Full bin: 43900 Hz
4. The result is sent over UART (`PA2` / `USART2`, 38400 baud) to a Python
   gateway script (`gateway/smartbin_gateway.py`), which forwards it to a
   live web dashboard over a REST API call.

<img width="1774" height="639" alt="how_it_works_ikounafy" src="https://github.com/user-attachments/assets/14588a14-40d7-4359-af30-fa86cac62629" />

## Hardware

| Component        | Description                             |
| ---------------- | --------------------------------------- |
| MCU              | STM32F301K8Tx                           |
| PCB              | Custom KiCad board                      |
| Analog Front-End | Relaxation oscillator + Schmitt trigger |
| Enclosure        | 3D-printed PLA                          |




## Experimental Validation

The sensing approach was validated through both numerical simulation and experimental calibration.

### COMSOL Simulation

Before hardware implementation, the sensor was modeled in **COMSOL Multiphysics** using an electrostatic model of a parallel-plate capacitor with a variable dielectric. Simulations predicted a monotonic increase in capacitance as the waste level increased, confirming the feasibility of using capacitance as a proxy for fill level and guiding the design of the analog conditioning circuit.

### Prototype Calibration

After assembling the prototype, the system was calibrated using five reference fill levels (0%, 25%, 50%, 75% and 100%). At each calibration point, the dashboard output was compared against the known fill level.

| Reference Fill Level | Measured Fill Level | Absolute Error |
|----------------------:|--------------------:|---------------:|
| 0%                    | 0%                  | 0 pp           |
| 25%                   | 22%                 | 3 pp           |
| 50%                   | 47%                 | 3 pp           |
| 75%                   | 71%                 | 4 pp           |
| 100%                  | 96%                 | 4 pp           |

Across the evaluated calibration points, the maximum observed deviation from the reference measurement was **4 percentage points**. The prototype exhibited a monotonic response throughout the tested operating range, demonstrating that the sensing chain—from the capacitive sensor and analog front-end to the STM32 firmware and dashboard—provided consistent fill-level estimation.

## Limitations

- **Sensing range**: the electrodes' signal strength limited reliable
  capacitance detection to a reduced height range relative to the full bin
  depth — a real constraint for scaling to full-size bins, not just the
  test enclosure
- **Waste-type dependency**: calibration (dielectric permittivity) is
  specific to the waste composition tested; not universal across all types
- **No autonomous connectivity**: the STM32 has no onboard Wi-Fi, so a
  computer running the gateway script must stay connected via USB at all
  times to relay data to the dashboard ; this is a proof of concept, not a
  standalone field deployment
- **Wired power**: the board is powered over USB, not battery — it can't
  yet be mounted on a real, mobile waste bin without a permanent power
  connection nearby

## Future Work

- Stronger/redesigned electrodes to extend the reliable sensing range
  across full bin depth
- ToF/ultrasonic sensor as a waste-composition-independent alternative
- Onboard ESP32 Wi-Fi to remove the computer relay and enable autonomous
  deployment
- Battery power with low-power sleep cycles
- LoRaWAN for long-range, low-power deployment across many bins

## See it in action

Because the sensing electronics rely on a custom-built analog conditioning
circuit (not just the STM32 board), this project **can't be reproduced by
simply cloning the repo and flashing firmware** — the physical PCB is part
of the system. The video above shows the fill level being changed by hand
on the physical prototype, with the dashboard updating in real time, as
evidence the measurement genuinely tracks physical fill level rather than
being simulated.



https://github.com/user-attachments/assets/b8830cd1-d0ca-4958-bd71-3b1533ac59b3


## License

This project's own code (`Core/`) is released under the MIT License — see
`LICENSE`. The `Drivers/` folder contains STMicroelectronics' HAL and CMSIS
files, which remain under their original ST license (see the `LICENSE.txt`
files within that folder); the MIT license above does not apply to them.
