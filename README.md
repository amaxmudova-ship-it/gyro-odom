# Lab 10 — Odometry and gyro localization

Trajectory: digits **7** and **8** (10 straight segments), start at (0, 0) facing +x.
Controller: turn in place, then drive. `r = 4.2 cm`, `B = 19.4 cm`, gyro sign `gs = −1` (Lab 9).

| Run | Course used by the robot |
|---|---|
| A | encoders: `course = course_odom` |
| B | gyro: `course = course + gyro_inc` (bias measured in the first 10 s) |

## Files
- `lab10_both_runs.qrs` — robot program (only `MODE` differs)
- `lab10_runA_078.txt`, `lab10_runB_078.txt` — logs: `t, theta_l, theta_r, gz, x, y, course, course_odom, course_gyro`
- `Lab10_analysis.ipynb` — Colab analysis

## Results

| Seg. | course_ref (°) | course A (°) | course B (°) | d_course A / B (°) | path_ref (cm) | path A (cm) | path B (cm) | d_path A / B (cm) |
|---|---|---|---|---|---|---|---|---|
| 1 | 0.0 | 0.1 | −0.8 | 0.1 / −0.8 | 25.0 | 23.5 | 23.1 | −1.5 / −1.9 |
| 2 | −116.6 | −121.2 | −121.1 | −4.7 / −4.6 | 44.7 | 48.2 | 50.7 | 3.5 / 6.0 |
| 3 | 29.7 | 31.6 | 34.9 | 1.9 / 5.1 | 40.3 | 43.5 | 46.8 | 3.2 / 6.5 |
| 4 | −90.0 | −93.8 | −93.7 | −3.8 / −3.7 | 20.0 | 20.0 | 22.9 | 0.0 / 2.9 |
| 5 | 0.0 | 5.1 | 1.5 | 5.1 / 1.5 | 25.0 | 29.2 | 26.5 | 4.2 / 1.5 |
| 6 | 90.0 | 90.6 | 93.5 | 0.6 / 3.5 | 20.0 | 20.6 | 21.3 | 0.6 / 1.3 |
| 7 | 180.0 | 185.7 | 179.5 | 5.7 / −0.5 | 25.0 | 30.1 | 24.6 | 5.1 / −0.4 |
| 8 | 90.0 | 86.3 | 86.3 | −3.7 / −3.7 | 20.0 | 21.0 | 21.6 | 1.0 / 1.6 |
| 9 | 0.0 | −1.0 | −5.6 | −1.0 / −5.6 | 25.0 | 24.8 | 27.1 | −0.2 / 2.1 |
| 10 | −90.0 | −94.2 | −93.4 | −4.2 / −3.4 | 20.0 | 21.4 | 20.7 | 1.4 / 0.7 |
| **Summary** | | | | mean \|d_course\|: **3.1 / 3.2** | **265.0** | **282.3** | **285.3** | **+6.5 % / +7.6 %** |

- Final error (robot's estimate) to the last waypoint: **A** 1.2 cm, −1.1° · **B** 1.7 cm, +0.3°
- Offline recomputation matches the robot: 0.0 mm in `x, y`, 0.00° in `course` (both runs)
- End of the 4 recomputed paths, distance to the last waypoint:

| | encoder course | gyro course |
|---|---|---|
| log of run A | 1.2 cm (robot) | 3.2 cm |
| log of run B | 5.7 cm | 1.7 cm (robot) |

## Analysis

**1. Which run is closer to the ideal trajectory?**
- **Course:** the logged course is equally close in both runs (mean |d_course| 3.1° vs 3.2°), because the controller steers the course it uses onto `course_ref`. The other sensor shows the error the robot could not see: in run A the gyro course stays closer (mean |d_course| 2.3°, end 3.2 cm) than the encoder course in run B (4.1°, end 5.7 cm).
- **Path:** run A is slightly closer (+6.5 % vs +7.6 %).
- **Why path errors are similar but course errors are not:** the path comes from the encoders in **both** runs; the course comes from **different** sensors.

**2. Which error source dominated?**
- **Run A — wheel slip in turns.** The gyro − encoder course difference grows almost only while turning in place (4.0° while turning, 0.6° while driving straight), and its sign follows the turn direction: the encoders counted 1.6 % more rotation than the gyro. Not `B`: in Lab 9 encoders and gyro agreed within 0.5 % on slow turns. Not scale factor or saturation: `s ≈ 0` (Lab 9), max |gz| = 93 °/s < 286 °/s.
- **Run B — gyro bias.** The turns are fine for the gyro, but on the straight parts of segments 6–10 the gyro course drifts away from the encoder course by up to 1.4 °/s (−8.4° in total while driving). An error that grows with time on straight lines is a bias error. The bias measured at the start (130 mdeg/s) differs from Lab 9 (464 mdeg/s), so it is not stable.

**3. How to combine `course_odom` and `gyro_inc`?**
Use the gyro for fast changes (turns, no slip) and the encoders to slowly pull back the gyro drift (straight lines, no bias):
`course = 0.01 · course_odom + 0.99 · (course + gyro_inc)` — the complementary filter of Lab 11.
