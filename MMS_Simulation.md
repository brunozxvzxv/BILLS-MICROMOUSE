---

## MMS Maze Simulation & Testing

To validate the **Flood-Fill** pathfinding and orientation logic before flashing the firmware onto the physical ESP32-S3, Bills was intensively tested using the **MMS (Micromouse Simulator)** environment.

### Simulation Overviews

Here are the breakdown assessments of the navigation logic across different grid conditions:

#### 1️⃣ Full Maze Navigation with Walls (`MMSsimulator.mp4`)
**Exploration & Mapping Profile:** This run shows Bills interacting with an unknown maze configuration. The mouse maps peripheral walls in real-time, increments cell path costs, handles dead ends efficiently, and tracks the shortest route toward the center coordinates.

<p align="center">
  <video src="https://github.com/user-attachments/assets/efece886-9ecd-467a-81f8-7d26777c4ab8" width="75%" controls></video>
</p>

#### 2️⃣ Open Grid Gradient Flow (`simulator-without-wall.mp4`)
**Ideal Path Convergence:** Testing the mathematical integrity of the Manhattan distance equations inside a blank maze grid. This simulation verifies that the coordinate vectors flawlessly stream down the lowest values of the calculation gradient directly to the center without lateral software trapping.

<p align="center">
  <video src="https://github.com/user-attachments/assets/cd583a6d-9b4e-41ea-ac6e-d701bfb26614" width="75%" controls></video>
</p>

---

## Code Shortcuts

Want to see the actual software controlling these simulations? 
---

### 🔀 Simulation Tracking

*  **[Click here to view the MMS Simulation Logic Code](./firmware/Logic_MMS_simulator.md)** to inspect how the coordinate navigation functions run inside the virtual matrix before flashing.

---
