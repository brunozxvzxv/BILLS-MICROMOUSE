#  Bills - Micromouse Robot

<p align="center">
  <strong>An autonomous maze-solving robot driven by matrix-based intelligence.</strong>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Project-Micromouse-orange?style=for-the-badge" alt="Project" />
  <img src="https://img.shields.io/badge/Hardware-ESP32--S3-blue?style=for-the-badge" alt="Hardware" />
  <img src="https://img.shields.io/badge/Status-In%20Development-green?style=for-the-badge" alt="Status" />
</p>

---

<p align="center">
  <img width="600" alt="Bills Physical Robot" src="https://github.com/user-attachments/assets/e8ef0b51-64d2-4508-87fe-5a3c08f7f2fa" />
</p>

## Overview

Bills is a micromouse robot designed to solve mazes using advanced logic and matrix-based algorithms. 

---

## Features

*  **Autonomous Maze Solving:** Independent grid exploration and real-time decision making.
*  **Matrix-Based Pathfinding:** Optimized flood-fill and mapping arrays.
*  **Wall Detection Sensors:** Multi-directional infrared real-time tracking.
*  **Motor Control System:** Precise dual-wheel differential drive execution.
*  **Optimized Movement Logic:** Fluid cell-to-cell displacement and tight turning.

---

##  Project Structure

The engineering and development process is divided into distinct key stages:

*  **PCB Design** — Custom hardware routing and component floorplanning.
*  **Electrical Schematics** — Power rail isolation and signal logic definitions.
*  **Firmware Development** — Embedded C++ algorithms for hardware execution.
*  **Sensor Integration** — Calibration and noise filtering for IR data arrays.
*  **Navigation Algorithms** — Flood-fill and shortest-path calculation rules.
*  **Testing and Calibration** — Closed-loop error corrections and speed tuning.

---

##  Hardware & Core Components

### Subsystems & Integrated Parts
*  **Microcontroller Unit (MCU):** High-speed processing for algorithm computing.
*  **IR Sensors Array:** Wall distance tracking and obstacle mapping.
*  **Motor Drivers:** Inductive current control and H-bridge switching.
*  **DC Motors:** High-RPM propulsion system.
*  **Encoders:** Closed-loop wheel odometry and rotation feedback.

---

##  Maze Solving Logic

> **Navigation Principle:** In the simulation environment, Bills creates a dynamic matrix logic that continuously reads the remaining paths, updates the grid weights, and follows the most efficient computed track straight towards the center of the maze.
