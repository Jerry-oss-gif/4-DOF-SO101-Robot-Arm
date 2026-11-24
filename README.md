[README.md](https://github.com/user-attachments/files/35319730/README.md)
# 4-DOF-SO101-Robot-Arm# SO-101 Robot Studio

A Python program to control a 4-joint **SO-101 robot arm** with **Feetech STS3215** servos.
It has a window (GUI) with a live **3D view** of the arm. You can also test everything in **simulation**, with no robot connected.

## Features

- 3D view that shows the same pose as the real arm
- Move each joint with sliders (forward kinematics)
- **End Point:** type X, Y, Z in mm and the real arm (or the simulation) moves there (inverse kinematics)
- Editable minimum / maximum limit for X, Y, Z
- Jog the tip in X, Y, Z by holding a button
- TCP speed slider (also sets the servo speed)
- Singularity warning (tells you when the arm is in a bad pose)
- Safety: emergency stop, joint limits, lost-connection stop, "arm is blocked" stop
- Calibration, so the 3D model and the real arm match
- Simulation mode (no robot needed)

#User Interface 
<img width="2878" height="1792" alt="Screenshot 2026-11-24 121044" src="https://github.com/user-attachments/assets/def82764-2890-4d6b-8d81-c02a6069a735" />
Check at youtube :(https://youtu.be/9wxgt4mI1iI)

## What you need

- Python 3.10 or newer
- A PC with a display (the GUI uses Tkinter)
- For the real arm:
  - 4 x Feetech STS3215 servos with IDs 1, 2, 3, 4
  - A USB servo adapter (for example Waveshare Bus Servo Adapter)
  - A power supply for the servos (USB alone does **not** power the motors)

## Install

```bash
git clone https://github.com/Jerry-oss-gif/4-DOF-SO101-Robot-Arm.git
cd 4-DOF-SO101-Robot-Arm
pip install -r requirements.txt
```

## Quick start

Simulation (no hardware):

```bash
python main.py
```

Real arm (connects when the window opens):

```bash
python main.py --real --port COM3
```

On Linux or Mac, use a port like `/dev/ttyACM0` or `/dev/ttyUSB0`.

| Option | Meaning | Default |
|---|---|---|
| `--real` | Use the real arm and connect at start | off (simulation) |
| `--port` | Serial port of the servo adapter | `COM3` |
| `--calibration` | Calibration file to use | `calibration.json` |

> Keep **all files in the same folder** and keep the file names exactly as they are. If you get `No module named 'kinematics'`, a file is missing or has a wrong name (for example `kinematics (1).py`).

## Setup order for a new arm

1. `scan_servos.py` → find the servos
2. `set_servo_id.py` → give each servo a unique ID (1, 2, 3, 4)
3. Calibrate (GUI **Calibrate** button, or `calibrate_home.py`)
4. `diagnose.py` → check that the real arm and the 3D model match
5. `main.py --real` → run the GUI, turn torque ON, press **Home**

## What each file does and where to use it

### Files you run

| File | What it does | When to use it |
|---|---|---|
| `main.py` | Starts the GUI (3D view, sliders, End Point, jog) | Every time you want to control the arm or test in simulation |
| `scan_servos.py` | Lists serial ports and scans baud rates and IDs 0-19 (`--full` = 0-253) | First step. Use it when the arm does not connect |
| `servo.py` | Small, quick version of the scan on one port | Fast check of baud rate and IDs |
| `set_servo_id.py` | Writes a new ID into **one** servo | New servos all have ID 1. Connect one servo at a time and give IDs 1, 2, 3, 4 |
| `calibrate_home.py` | Command-line calibration (torque off, you move the arm by hand) | First setup, or after a servo horn was moved |
| `diagnose.py` | Live table: real arm vs 3D model | After calibration, to check that they match |
| `scanposition.py` | Prints the raw position (ticks and degrees) of each servo | To read positions only |
| `hardware.py`, `kinematics.py` | Have a built-in self-test (no robot needed) | To check that the code works |

Examples:

```bash
python scan_servos.py              # all ports
python scan_servos.py COM5         # one port
python scan_servos.py COM5 --full  # IDs 0-253 (slow)
python servo.py COM5
python set_servo_id.py COM5 1 2    # port, old ID, new ID (ONE servo connected)
python calibrate_home.py COM3
python diagnose.py COM3            # check the calibration pose
python diagnose.py COM3 home       # check the home pose
python hardware.py                 # self-test
python kinematics.py               # self-test
```

For `scanposition.py`, edit the line `PORT = "COM3"` inside the file first, then run `python scanposition.py`.
Do not copy its numbers into `zero_deg` by hand. Use the Calibrate step instead.

### Files used by the program (you do not run them)

| File | What it does |
|---|---|
| `gui.py` | The window: sliders, End Point panel, jog buttons, 3D view. `main.py` loads it |
| `controller.py` | Moves the arm (jog, End Point, joint targets) and handles safety (E-STOP, limits, lost connection) |
| `kinematics.py` | The math of the arm: forward / inverse kinematics, joint limits, home pose, X/Y/Z limits |
| `hardware.py` | Talks to the servos, converts ticks to angles, loads and saves `calibration.json`, has the simulated robot |
| `calibration.json` | The saved calibration (see below) |
| `__init__.py` | Lets you import the files as a package. You can ignore it |
| `requirements.txt` | The Python packages to install |

## calibration.json: make the 3D model and the real arm match

The 3D view (UI) shows **model angles**. The real servos report **encoder angles**.
They are not the same, because every servo horn is mounted a little differently.
`calibration.json` stores the link between the two, **one entry per joint**:

```json
{
  "servo_id": 1,
  "name": "shoulder_pan",
  "sign": -1,
  "zero_deg": -26.89
}
```

| Field | Meaning |
|---|---|
| `servo_id` | The ID of the servo on the bus (1, 2, 3, 4) |
| `name` | Joint name |
| `sign` | **Direction.** `+1` = servo and model turn the same way. `-1` = they turn opposite ways |
| `zero_deg` | The servo angle (-180 to +180 deg) when the **model angle is 0**. It is the offset |

The program uses this formula:

```text
servo angle (deg) = zero_deg + sign * model angle (deg)
servo ticks       = (servo angle + 180) / 360 * 4096
```

Example with the file in this project (arm held in the calibration pose):

| Joint | Model angle | sign | zero_deg | Servo angle | Servo ticks |
|---|---|---|---|---|---|
| shoulder_pan | 0° | -1 | -26.89 | -26.89° | about 1742 |
| shoulder_lift | -143° | +1 | 105.0 | -38° | about 1616 |
| elbow_flex | 99° | +1 | -100.0 | -1° | about 2037 |
| wrist_flex | 67° | +1 | -33.0 | 34° | about 2435 |

So when the real arm is in the calibration pose, the servos read about these ticks, and the 3D model shows the pose `0, -143, 99, 67` degrees. After that, when you move a joint, the model moves in the same direction and by the same angle.

### How to calibrate

Do this the first time, and again whenever a servo horn was moved or the numbers look wrong.

1. Turn **torque OFF** (you move the arm by hand).
2. Hold the real arm in the **calibration pose** (`calib_deg` in `kinematics.py`, see below). Pan straight ahead.
3. Run `python calibrate_home.py COM3`, or use the GUI: **Calibrate → Apply & Save**.
   Move each joint a little in the model's positive direction when asked. This sets `sign`.
   Then hold the pose and press Enter. This sets `zero_deg`.
4. Run `python diagnose.py COM3`. `raw` should be close to `expect`, and every `err` should be close to 0.
5. Move one joint by hand. The 3D model must move in the **same direction** and by the **same angle**.
6. Only then turn **torque ON** and press **Home**.

**Positive direction:** pan = counter-clockwise (seen from above). Shoulder, elbow, wrist = down.

If a joint shows `RISK` in `diagnose.py`, its range crosses the servo wrap point (tick 0 / 4095). Re-mount that servo horn one spline step and calibrate again.

> `calibration.json` belongs to **your** arm. If you share this project, other people must calibrate their own arm.

## ArmGeometry: the arm settings in kinematics.py

These settings are in `kinematics.py`, in `class ArmGeometry`. They describe the size of your arm and its limits.

```python
class ArmGeometry:
    d1: float = 0.075   # base height to shoulder axis (m)
    a1: float = 0.075   # horizontal offset base axis -> shoulder axis (m)
    a2: float = 0.130   # upper arm (m)
    a3: float = 0.120   # forearm (m)
    a4: float = 0.065   # wrist -> TCP (m)
    # pan, shoulder, elbow, wrist
    limits_deg: tuple = ((-65, 65), (-180, -55), (23.5, 180), (-80, 80))
    # home = folded pose inside limits
    home_deg: tuple = (0, -143, 99, 67)
    # calibration pose = "upright": upper arm vertical, forearm horizontal forward, wrist level.
    # Hold the real arm in this pose when calibrating (GUI dialog, calibrate_home.py and diagnose.py all use it).
    calib_deg: tuple = (0, -143, 99, 67)
    # End-point workspace box in mm: the TCP must stay between min and max on every axis.
    # >>> Change these numbers to set the minimum / maximum limit for X, Y, Z <<<
    xyz_min_mm: tuple = (-100, 100, 50)      # X min, Y min, Z min
    xyz_max_mm: tuple = (100, 300, 250)      # X max, Y max, Z max
```

### Link lengths (in meters)

| Line | Meaning |
|---|---|
| `d1 = 0.075` | Height from the base to the shoulder axis: 75 mm |
| `a1 = 0.075` | Horizontal distance from the base axis to the shoulder axis: 75 mm |
| `a2 = 0.130` | Upper arm length: 130 mm |
| `a3 = 0.120` | Forearm length: 120 mm |
| `a4 = 0.065` | Wrist to the TCP (tool center point, the tip): 65 mm |

The maximum reach is `a1 + a2 + a3 + a4 = 75 + 130 + 120 + 65 = 390 mm`.
If you add a gripper or a longer tool, change `a4` to the new length.

### Joint limits

```python
limits_deg = ((-65, 65), (-180, -55), (23.5, 180), (-80, 80))
```

Each pair is `(minimum, maximum)` in **degrees**, in this order: **pan, shoulder, elbow, wrist**.

| Joint | Min | Max |
|---|---|---|
| pan | -65° | 65° |
| shoulder | -180° | -55° |
| elbow | 23.5° | 180° |
| wrist | -80° | 80° |

The sliders, the End Point move and the jog never go outside these limits. If your real arm can turn further, or has a stop that blocks earlier, change these numbers.

### Special poses

| Line | Meaning |
|---|---|
| `home_deg = (0, -143, 99, 67)` | The pose the **Home** button moves to (pan, shoulder, elbow, wrist, in degrees). It is a folded pose and must be **inside** the joint limits |
| `calib_deg = (0, -143, 99, 67)` | The pose you hold with your hands when you **calibrate**. The GUI Calibrate dialog, `calibrate_home.py` and `diagnose.py` all use it |

The comment in the code calls the calibration pose "upright", but the numbers `0, -143, 99, 67` are the folded pose, the same as home. **The numbers are what the program uses.** Hold the real arm in the pose that these numbers describe.
If you want another calibration pose, change `calib_deg`, then hold the arm in that new pose and calibrate again.

### X / Y / Z limits for the End Point

```python
xyz_min_mm = (-100, 100, 50)    # X min, Y min, Z min   (mm)
xyz_max_mm = (100, 300, 250)    # X max, Y max, Z max   (mm)
```

This is a box in space. The tip of the arm (TCP) must stay inside it. The order is always **X, Y, Z**.

- **End Point** and **Jog** are checked against this box. A target outside the box is refused with a message.
- You can also change the limits in the GUI: type new numbers in the **min** and **max** boxes of the End Point panel and press Enter. This is **not saved**. After a restart, the numbers from `kinematics.py` are used again.
- Axes: +Y is forward, +X is to the right, +Z is up. Z = 0 is the base level.
- Each min must be smaller than its max, or the program stops with an error.

## End Point (go to X, Y, Z)

Use the **End Point (IK)** panel. It works the same for the simulation and the real arm (torque must be ON for the real arm).

1. Type **X, Y, Z** in mm. Each value must be inside its min / max limit.
2. Leave **Auto pitch** ticked if only the position matters. The program finds a tool pitch that works. Untick it to type your own pitch.
3. **Straight line** ticked: the tip moves in a straight line at the TCP speed. The whole path is checked first. Untick it for a **joint move**: every joint goes straight to the end point (the path is curved).
4. Press **Move to End Point**. If the point cannot be reached, a message tells you why and the arm does not move.
5. **Copy Current TCP -> End Point** copies where the arm is now into the boxes.

The arm is small (reach 390 mm) and the pan joint only turns -65 to +65 degrees. So not every point in the box can be reached. In the tests, the reachable area was roughly X -275 to +275, Y +25 to +325, Z 0 to +375 mm.

## TCP speed

The **TCP Speed Setting** slider (5 to 150 mm/s) sets how fast the tip moves in Jog and End Point moves. The program also sets the joint speed limit (45 to 180 deg/s) and the servo speed from this slider.
Start with a low speed on a new setup.

## Feetech servo software (official)

To set up or test a servo with a window program (change ID, baud rate, check the servo, update firmware), download the official Feetech software:
(https://www.feetechrc.com/software.html)
<img width="2446" height="1570" alt="Screenshot 2026-11-22 003532" src="https://github.com/user-attachments/assets/51572c55-3ab9-4f74-b4f2-dcd69ef9df97" />
Download FD 1.9...etc (Latest version)

Close this program before you use it, because only one program can use the port at a time.

## Safety tips

- Always **support the arm** before you turn torque off or close the program. It can fall.
- Check that the 3D model matches the real arm **before** you turn torque ON.
- Keep your hand near the **EMERGENCY STOP** button the first time you run a new calibration or a new speed.
- Use a low TCP speed when you test new limits.

## Troubleshooting

**No servo answers:**

1. Is the servo power supply ON?
2. Is it the right COM port? (Look for CH340, CP210x, FTDI, or "USB Serial".)
3. On a Waveshare adapter, the jumper must be on channel **A** (USB).
4. Is the 3-pin cable fully pushed in?
5. Close other programs that use the port (this GUI, LeRobot, Feetech software).
6. Do all servos have the same ID? Give each one a unique ID with `set_servo_id.py`.

**`No module named 'kinematics'`:** put all files in one folder and check the file names.

**The 3D model and the real arm do not match:** calibrate again and check with `diagnose.py`.

## License

Add your license here (for example MIT).
