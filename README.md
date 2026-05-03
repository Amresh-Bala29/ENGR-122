# Autonomous Navigation Robot

ENGR-122 final project — an autonomous robot that navigates a course using ultrasonic obstacle avoidance and camera-based position feedback delivered over MQTT.

The robot drives itself to a sequence of target coordinates inside an arena while avoiding walls and obstacles. An overhead camera tracks the robot's position and orientation using ArUco markers, then publishes that data over MQTT. Onboard, three ultrasonic sensors handle local collision avoidance, and a path-planning routine steers the robot toward each target by minimizing the angle error between its current heading and the desired heading.

The repo contains the final integrated sketch plus the incremental lab tests we built it on.

## Repository Structure

| Folder | Description |
|---|---|
| `Final_Project/` | Integrated sketch combining MQTT position feedback, ultrasonic obstacle avoidance, and waypoint path planning |
| `Camera_Sensing_Lab/` | Week 7 baseline lab for parsing camera position data over MQTT and displaying it on the OLED |
| `Road_Test_1/` | Single-sensor stop-on-wall behavior |
| `Road_Test_2/` | Iteration on stop logic with cleaner motor and display handling |
| `Road_Test_3/` | Extended test (not included in this upload) |
| `left_hand_loop/` | Wall-following demo using right-side correction to traverse a loop |

## Hardware

- Microcontroller with WiFi (programmed via Arduino IDE)
- 2x continuous-rotation servo motors (left on D3, right on D6)
- 3x HC-SR04 ultrasonic sensors
  - Middle: trig D5, echo D8
  - Right: trig D4, echo D7
  - Left: trig TX, echo D0
- SSD1306 OLED display (I²C, address 0x3C, SDA D2, SCL D1)
- Overhead camera + ArUco markers (external, publishes to MQTT)

## Software Dependencies

Install through the Arduino Library Manager:

- `MQTT`
- `Wire`
- `SSD1306Wire`
- `Ultrasonic`
- `Servo`

## How It Works

**Position feedback.** An overhead camera reads the ArUco marker on top of the robot and publishes `x`, `y`, and `z_ang` (heading) to an MQTT topic. The robot subscribes to that topic and uses the values to compute its distance and bearing to the next target.

**Path planning.** For each target waypoint, the robot calculates:
- `dx`, `dy` to the target
- `distance_to_target` using the Pythagorean theorem
- `desired_angle` using `atan2(dy, dx)`
- `angle_error` between the desired heading and the current heading, normalized to [-180, 180]

If the angle error is greater than 15 degrees, the robot makes a left correction. If it's less than -15, a right correction. Otherwise it drives straight. When `distance_to_target < 100`, the robot stops, increments the target index, and moves on to the next waypoint.

**Obstacle avoidance.** Three ultrasonic readings run every loop:
- If the middle sensor reads under 16 cm, the robot turns away from whichever side has more space
- If the left sensor reads under 8 cm, the robot corrects right
- If the right sensor reads under 8 cm, the robot corrects left

Obstacle avoidance takes priority over path planning, so the robot defers heading correction whenever something is in the way.

**Display.** The OLED shows live sensor distances and the current angle error for debugging during runs.

## Configuration

Before running, update the WiFi and MQTT settings in `Final_Project` to match the arena you're testing in. The sketch has commented-out blocks for North Arena, South Arena, and the Integration Lab — uncomment the one you need.

Update the target coordinates (`x_targets`, `y_targets`) for whatever course you're running.

## Tuning Notes

A few values that took the most iteration:
- Forward motor values of 110 / 109 to compensate for left-right drift
- 720 ms delay for a 90-degree turn
- 15-degree angle error threshold before applying corrections (tighter caused oscillation, looser caused wide arcs around targets)
- 100-unit target arrival radius (the camera coordinate system is in pixels, not cm)

## Progression

The lab sketches in this repo show how we got to the final integration:

1. **Road Test 1 / 2** — Single front sensor, stop before hitting a wall
2. **Left Hand Loop** — Wall-following with right-side correction to traverse a closed loop
3. **Camera Sensing Lab** — Receive position data over MQTT and display it on the OLED
4. **Final Project** — All of the above combined with waypoint path planning
