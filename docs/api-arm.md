<h1 align="center">ARM API</h1>

This page documents every arm class in `hex_driver_robot`: what to import, how to start a
driver, which methods each device has, and what limits travel with each command. 

## Contents

- [1. Two arm families](#1-two-arm-families)
- [Part A: manipulators](#part-a-manipulators)
  - [A.1 Quick start](#a1-quick-start)
  - [A.2 Parameters](#a2-parameters)
  - [A.3 Driver classes](#a3-driver-classes)
  - [A.4 All methods](#a4-all-methods)
  - [A.5 Recommended control commands](#a5-recommended-control-commands)
  - [A.6 Command methods](#a6-command-methods)
    - [`set_arm_pos_cmd()`](#set_arm_pos_cmd)
    - [`set_arm_pose_cmd()`](#set_arm_pose_cmd)
    - [`set_arm_mit_cmd()`](#set_arm_mit_cmd)
    - [`set_grip_pos_cmd()`](#set_grip_pos_cmd)
    - [`set_grip_force_cmd()`](#set_grip_force_cmd)
    - [`set_grip_mit_cmd()`](#set_grip_mit_cmd)
  - [A.7 State methods](#a7-state-methods)
    - [`get_arm_state()`](#get_arm_state)
    - [`get_grip_state()`](#get_grip_state)
    - [`get_arm_motor_status()` and `get_grip_motor_status()`](#get_arm_motor_status-and-get_grip_motor_status)
    - [`get_grip_limits()`](#get_grip_limits)
    - [`get_arm_robot_mode()`](#get_arm_robot_mode)
    - [`is_control_able()`](#is_control_able)
    - [`get_dofs()`](#get_dofs)
- [Part B: Hello Y6 read-only arm](#part-b-hello-y6-read-only-arm)
  - [B.1 Quick start](#b1-quick-start)
  - [B.2 Parameters](#b2-parameters)
  - [B.3 Driver classes](#b3-driver-classes)
  - [B.4 Command methods](#b4-command-methods)
    - [`set_rgb_cmd()`](#set_rgb_cmd)
  - [B.5 State methods](#b5-state-methods)
    - [`get_arm_state()`](#get_arm_state-1)
    - [`get_grip_joy()`](#get_grip_joy)
    - [`get_dofs()`](#get_dofs-1)

---

## 1. Two arm families

| Family | Classes | Motor commands | Read-only |
|---|---|:--:|:--:|
| Manipulators | `HexRobotArcherY6`, `HexRobotFireflyY6`, `HexRobotFireflyY6H2_40` | ✅ | — |
| Hello Y6 | `HexRobotHelloY6` | — | ✅ |

The two families are documented separately below because their interfaces differ: a manipulator
takes joint, pose and MIT commands plus an optional grip, while the Hello Y6 arm takes
**no motor command at all** — it is driven by its joystick grip and only reports state. The
parts also have different parameters.

The three manipulator classes share one implementation: identical methods, identical
command keys, identical state layout. Only the connected arm model (and therefore the safe
range of motion) differs.

---

## Part A: manipulators

Applies to `HexRobotArcherY6`, `HexRobotFireflyY6`, `HexRobotFireflyY6H2_40`.

### A.1 Quick start

```python
from hex_util_runtime import HexRate
from hex_driver_robot import HexRobotArcherY6, HexRobotArcherY6Params

params = HexRobotArcherY6Params(
    host="192.168.1.100",   # controller address
    port=8439,              # controller port
    ctrl_rate=500,          # control loop rate [Hz]
    grip_type="empty",      # "empty" (no grip), "gp80", "gr100", "gp100"
)
robot = None
try:
    robot = HexRobotArcherY6(params)
    robot.start()                                   # opens the session, starts the control loop
    print(robot.get_dofs())                         # {'arm': 6, 'grip': 0}

    rate = HexRate(params.ctrl_rate)
    while robot.is_working():
        rate.sleep()
        robot.set_arm_pos_cmd({"jnt_pos": [0.0, -1.5, 3.0, 0.0, 0.0, 0.0]})
        state = robot.get_arm_state()
        if state is not None:
            print(state.arm_state.jnt.position)
finally:
    if robot is not None:
        robot.stop()                                # leaves the session, joins the thread
```

Three things are enough to get moving:

1. **Send a command, then keep looping.** A command is latched: the driver re-sends the
   last one every cycle. Send a new command when the target changes.
2. **Read state in the same loop.** `get_*()` returns `None` until data has arrived, so
   always check before using a value.
3. **Put `robot.stop()` in `finally`.** It ends the session and joins the control thread.

### A.2 Parameters

Every parameter object extends `HexRobotBaseParams`:

| Field | Default | Unit | Meaning |
|---|---|---|---|
| `host` | `"192.168.1.100"` | — | Controller address |
| `port` | `8439` | — | Controller port |
| `ctrl_rate` | `500` | Hz | Control loop rate |
| `state_buffer_size` | `100` | — | Depth of the state ring buffers |
| `sens_ts` | `True` | — | Stamp states with the controller clock instead of the local clock |

Manipulator parameters add:

| Field | Default | Meaning |
|---|---|---|
| `grip_type` | `"gp80"` | `"empty"` (no grip), `"gp80"`, `"gr100"` or `"gp100"`. `"empty"` sets the grip DOF to 0 and turns every grip command into a no-op |
| `enable_kcp` | `True` | Use the reliable transport when the network needs it |
| `init_timeout` | `30.0` | Seconds to wait for the first motor state before giving up |
| `log_level` | `"INFO"` | Driver log level |
| `robot_type` | model-specific | Present on the Firefly classes only; leave at its default unless you are told otherwise |

Which model each class connects to:

| Class | Parameters class | Arm model |
|---|---|---|
| `HexRobotArcherY6` | `HexRobotArcherY6Params` | Archer Y6 H1 |
| `HexRobotFireflyY6` | `HexRobotFireflyY6Params` | Firefly Y6 H1 |
| `HexRobotFireflyY6H2_40` | `HexRobotFireflyY6H2_40Params` | Firefly Y6 H2_40 |

### A.3 Driver classes

Each model ships three classes with the same name prefix:

| Class | Role |
|---|---|
| `HexRobot<Model>` | Polling driver: `get_*()` methods read from internal ring buffers |
| `HexRobot<Model>Callback` | Same driver, but state is pushed to callables you pass as `{"arm_state": fn, "grip_state": fn}` |
| `HexRobot<Model>Params` | Parameter object described above |

Lifecycle methods, available on all of them:

| Method | Returns | Meaning |
|---|---|---|
| `start()` | `None` | Opens the session and starts the background control loop |
| `stop()` | `None` | Ends the session and joins the control loop; call it from `finally` |
| `is_working()` | `bool` | `True` while the control loop runs — the natural `while` condition |

### A.4 All methods

✅ = the method exists on this device. Hello Y6 has none of these methods — see [Part B](#part-b-hello-y6-read-only-arm).

| Method | Kind | Archer Y6 | Firefly Y6 | Firefly Y6 H2_40 |
|---|---|:--:|:--:|:--:|
| `set_arm_pos_cmd()` | Command | ✅ | ✅ | ✅ |
| `set_arm_pose_cmd()` | Command | ✅ | ✅ | ✅ |
| `set_arm_mit_cmd()` | Command | ✅ | ✅ | ✅ |
| `set_grip_pos_cmd()` | Command | ✅ | ✅ | ✅ |
| `set_grip_force_cmd()` | Command | ✅ | ✅ | ✅ |
| `set_grip_mit_cmd()` | Command | ✅ | ✅ | ✅ |
| `get_arm_state()` | State | ✅ | ✅ | ✅ |
| `get_grip_state()` | State | ✅ | ✅ | ✅ |
| `get_arm_motor_status()` | State | ✅ | ✅ | ✅ |
| `get_grip_motor_status()` | State | ✅ | ✅ | ✅ |
| `get_grip_limits()` | State | ✅ | ✅ | ✅ |
| `get_arm_robot_mode()` | State | ✅ | ✅ | ✅ |
| `is_control_able()` | State | ✅ | ✅ | ✅ |
| `get_dofs()` | State | ✅ | ✅ | ✅ |
| `start()` / `stop()` / `is_working()` | Lifecycle | ✅ | ✅ | ✅ |

All six grip methods are no-ops when `grip_type="empty"`: the commands return without
sending anything, and the grip state getters return `None`.

### A.5 Recommended control commands

| Method | Level | Reason |
|---|---|---|
| `set_arm_pos_cmd()` | Recommended | Carries a velocity limit and an acceleration limit, applied to all six joints |
| `set_arm_pose_cmd()` | Recommended | Carries a velocity limit and an acceleration limit for the joint motion that follows |
| `set_grip_pos_cmd()` | Recommended | Carries a speed limit and a torque ceiling, which acts as the squeeze limit |
| `set_arm_mit_cmd()` | Not recommended | No velocity limit, no acceleration limit: your values reach the motors as they are |
| `set_grip_force_cmd()` | Not recommended | Bounds speed only; torque is whatever you command |
| `set_grip_mit_cmd()` | Not recommended | No velocity limit, no acceleration limit |

The state methods are read-only and safe to call at any time.

### A.6 Command methods

All commands take a single `dict`. Unlisted keys keep the default shown. `ts_ns`
(default `ns_now()`) is the command timestamp and rarely needs setting.

#### `set_arm_pos_cmd()`

`set_arm_pos_cmd(cmd)` — the normal way to move an arm.

**Usage**

| Key | Type | Default | Unit | Meaning |
|---|---|---|---|---|
| `jnt_pos` | 6 floats | required | rad | Target joint positions |
| `jnt_eff` | 6 floats | zeros | Nm | Feed-forward torque added on top of control |
| `lim_vel` | float | `8.0` | rad/s | Velocity limit, applied to every joint |
| `lim_acc` | float | `50.0` | rad/s² | Acceleration limit, applied to every joint |
| `grav` | `HexDcBaseVector3` | `None` | m/s² | Gravity vector for compensation; omit for none |
| `ts_ns` | int | `ns_now()` | ns | Command timestamp |

```python
robot.set_arm_pos_cmd({
    "jnt_pos": [0.0, -1.5, 3.0, 0.0, 0.0, 0.0],
    "lim_vel": 1.0,
    "lim_acc": 50.0,
})
```

#### `set_arm_pose_cmd()`

`set_arm_pose_cmd(cmd)` — Cartesian target.

**Usage**

| Key | Type | Default | Unit | Meaning |
|---|---|---|---|---|
| `pose_pos` | 3 floats | required | m | Target end-effector position `[x, y, z]` |
| `pose_quat` | 4 floats | `[1, 0, 0, 0]` | — | Target orientation `[w, x, y, z]` |
| `jnt_eff` | 6 floats | zeros | Nm | Feed-forward torque |
| `lim_vel` | float | `8.0` | rad/s | Velocity limit for the joint motion |
| `lim_acc` | float | `50.0` | rad/s² | Acceleration limit for the joint motion |
| `grav` | `HexDcBaseVector3` | `None` | m/s² | Gravity vector for compensation |
| `ts_ns` | int | `ns_now()` | ns | Command timestamp |

The pose target is resolved by the controller; the velocity and acceleration limits you
pass shape the joint motion it produces. Read the result back from
`get_arm_state().arm_state.pose`.

```python
robot.set_arm_pose_cmd({"pose_pos": [0.3, 0.0, 0.3], "pose_quat": [1.0, 0.0, 0.0, 0.0]})
```

#### `set_arm_mit_cmd()`

`set_arm_mit_cmd(cmd)` — direct impedance.

**Usage**

| Key | Type | Default | Unit | Meaning |
|---|---|---|---|---|
| `jnt_pos` | 6 floats | zeros | rad | Target joint positions |
| `jnt_vel` | 6 floats | zeros | rad/s | Target joint velocities |
| `mit_tau` | 6 floats | zeros | Nm | Feed-forward torque |
| `mit_kp` | 6 floats | zeros | Nm/rad | Stiffness |
| `mit_kd` | 6 floats | zeros | Nm/(rad/s) | Damping |
| `grav` | `HexDcBaseVector3` | `None` | m/s² | Gravity vector for compensation |
| `ts_ns` | int | `ns_now()` | ns | Command timestamp |

```python
robot.set_arm_mit_cmd({
    "jnt_pos": [0.0, -1.5, 3.0, 0.0, 0.0, 0.0],
    "mit_kp": [50.0, 50.0, 50.0, 50.0, 20.0, 20.0],
    "mit_kd": [1.0, 1.0, 1.0, 1.0, 0.5, 0.5],
})
```

> ⚠️ Improper **kp** and **kd** parameters can damage the equipment.

#### `set_grip_pos_cmd()`

`set_grip_pos_cmd(cmd)` — grip position.

**Usage**

| Key | Type | Default | Unit | Meaning |
|---|---|---|---|---|
| `jnt_pos` | 1 float | required | rad | Target grip position |
| `jnt_eff` | 1 float | `3.0` | Nm | Torque ceiling; acts as the squeeze limit |
| `lim_vel` | float | `2.0` | rad/s | Speed limit |
| `ts_ns` | int | `ns_now()` | ns | Command timestamp |

```python
robot.set_grip_pos_cmd({"jnt_pos": [0.5]})
```

#### `set_grip_force_cmd()`

`set_grip_force_cmd(cmd)` — grip torque.

**Usage**

| Key | Type | Default | Unit | Meaning |
|---|---|---|---|---|
| `jnt_eff` | 1 float | required | Nm | Target torque — this is the force you get |
| `lim_vel` | float | `2.0` | rad/s | Speed limit |
| `ts_ns` | int | `ns_now()` | ns | Command timestamp |

```python
robot.set_grip_force_cmd({"jnt_eff": [0.5], "lim_vel": 1.0})
```

#### `set_grip_mit_cmd()`

`set_grip_mit_cmd(cmd)` — grip direct impedance.

**Usage**

| Key | Type | Default | Unit | Meaning |
|---|---|---|---|---|
| `jnt_pos` | 1 float | zeros | rad | Target position |
| `jnt_vel` | 1 float | zeros | rad/s | Target velocity |
| `mit_tau` | 1 float | zeros | Nm | Feed-forward torque |
| `mit_kp` | 1 float | zeros | Nm/rad | Stiffness |
| `mit_kd` | 1 float | zeros | Nm/(rad/s) | Damping |
| `ts_ns` | int | `ns_now()` | ns | Command timestamp |

No motion limit of any kind.

### A.7 State methods

`get_*(latest=True)` drains the buffer and returns the newest sample, discarding older ones;
`get_*(latest=False)` returns the oldest buffered sample and leaves the rest. Both return
`None` while no data has arrived.

#### `get_arm_state()`

`get_arm_state()` returns one arm-state sample, or `None` while no state has arrived.

**Usage**

```python
state = robot.get_arm_state()
if state is not None:
    print(state.arm_state.jnt.position)
```

**Return**

```python
HexDcRoboArmStateStamped: {
    "header": {
        "stamp": HexDcBaseTime,                      # {"secs": int, "nsecs": int}
    },
    "arm_state": {
        "jnt": {
            "position": np.ndarray,                  # (6,) float64, rad
            "velocity": np.ndarray,                  # (6,) float64, rad/s
            "effort": np.ndarray,                    # (6,) float64, Nm
        },
        "pose": {                                    # computed by the controller
            "position": HexDcBaseVector3,            # {"x": float, "y": float, "z": float}, m
            "orientation": HexDcBaseQuaternion,      # {"x": float, "y": float, "z": float, "w": float}
        },
    },
}
```

When the controller reports no pose, `pose.position` is `[0, 0, 0]` and
`pose.orientation` is `[1, 0, 0, 0]`.

#### `get_grip_state()`

`get_grip_state()` returns one grip-state sample, or `None` while no grip state has
arrived. Always `None` when `grip_type="empty"`.

**Usage**

```python
grip = robot.get_grip_state()
if grip is not None:
    print(grip.grip_state.jnt.position)
```

**Return**

```python
HexDcRoboGripStateStamped: {
    "header": {
        "stamp": HexDcBaseTime,                      # {"secs": int, "nsecs": int}
    },
    "grip_state": {
        "jnt": {
            "position": np.ndarray,                  # (1,) float64, rad
            "velocity": np.ndarray,                  # (1,) float64, rad/s
            "effort": np.ndarray,                    # (1,) float64, Nm
        },
    },
}
```

#### `get_arm_motor_status()` and `get_grip_motor_status()`

Both return the same structure, or `None` until motor data has arrived.

**Usage**

```python
status = robot.get_arm_motor_status()
if status is not None:
    print(status["motor_temp"], status["error"])
```

**Return**

```python
dict: {
    "motor_temp": np.ndarray,                        # (n,) float64, °C, nan when not reported
    "driver_temp": np.ndarray,                       # (n,) float64, °C, nan when not reported
    "error": list[list[str]],                        # error names per motor, empty list = no error
}
```

#### `get_grip_limits()`

`get_grip_limits()` returns the grip travel, or `None` when no grip is configured.

**Usage**

```python
limits = robot.get_grip_limits()
```

**Return**

```python
tuple[float, float]: (
    min,                                             # rad, minimum grip position
    max,                                             # rad, maximum grip position
)
```

#### `get_arm_robot_mode()`

`get_arm_robot_mode()` returns the controller mode, or `None` before the first state.

**Usage**

```python
mode = robot.get_arm_robot_mode()
```

**Return**

```python
str: "RmStandby" | "RmRunning" | "RmFatalError" | "Unknown(N)"
```

#### `is_control_able()`

`is_control_able()` returns `True` only when the controller reports `"RmRunning"` **and** this
driver holds the control session.

**Usage**

```python
if robot.is_control_able():
    robot.set_arm_pos_cmd({"jnt_pos": [0.0, -1.5, 3.0, 0.0, 0.0, 0.0]})
```

**Return**

```python
bool: True | False
```

#### `get_dofs()`

`get_dofs()` returns the joint count of each part.

**Usage**

```python
print(robot.get_dofs())
```

**Return**

```python
dict: {
    "arm": int,                                      # 6 for every manipulator
    "grip": int,                                     # 0 with grip_type="empty", otherwise 1
}
```

---

## Part B: Hello Y6 read-only arm

Applies to `HexRobotHelloY6`. This arm has no motor commands: it is driven by its joystick
grip, and the only command the driver sends is the LED strip.

### B.1 Quick start

```python
from hex_util_runtime import HexRate
from hex_driver_robot import HexRobotHelloY6, HexRobotHelloY6Params

params = HexRobotHelloY6Params(
    host="192.168.1.100",
    port=8439,
    ctrl_rate=500,
    led_buffer_size=10,
)
robot = None
try:
    robot = HexRobotHelloY6(params)
    robot.start()
    print(robot.get_dofs())                         # {'arm': 6}

    # one entry per LED, 0-255 per channel
    robot.set_rgb_cmd({"r": [0] * 6, "g": [255] * 6, "b": [0] * 6})

    rate = HexRate(params.ctrl_rate)
    while robot.is_working():
        rate.sleep()
        state = robot.get_arm_state()
        if state is not None:
            print(state.arm_state.jnt.position)
        joy = robot.get_grip_joy()
        if joy is not None:
            print(joy.joystick_x, joy.trigger)
finally:
    if robot is not None:
        robot.stop()
```

### B.2 Parameters

`HexRobotHelloY6Params` extends `HexRobotBaseParams` (`host`, `port`, `ctrl_rate`,
`state_buffer_size`, `sens_ts` — see [A.2](#a2-parameters)) and adds:

| Field | Default | Meaning |
|---|---|---|
| `led_buffer_size` | `10` | Depth of the LED command queue; the newest command in a cycle wins |
| `enable_kcp` | `True` | Use the reliable transport when the network needs it |
| `init_timeout` | `30.0` | Seconds to wait for the first arm state |
| `log_level` | `"DEBUG"` | Driver log level |

### B.3 Driver classes

| Class | Role |
|---|---|
| `HexRobotHelloY6` | Polling driver: `get_*()` methods read from internal ring buffers |
| `HexRobotHelloY6Callback` | Same driver, but state is pushed to callables you pass as `{"arm_state": fn, "grip_joy": fn}` |
| `HexRobotHelloY6Params` | Parameter object described above |

Lifecycle methods:

| Method | Meaning |
|---|---|
| `start()` | Opens the session and starts the background control loop |
| `stop()` | Ends the session and joins the control loop; call it from `finally` |
| `is_working()` | `True` while the control loop runs |

### B.4 Command methods

#### `set_rgb_cmd()`

`set_rgb_cmd(cmd)` drives the LED strip. 

**Usage**

| Key | Type | Default | Unit | Meaning |
|---|---|---|---|---|
| `r` | list of int | required | — | Red channel, one value per LED, `0`–`255` |
| `g` | list of int | required | — | Green channel, one value per LED, `0`–`255` |
| `b` | list of int | required | — | Blue channel, one value per LED, `0`–`255` |

All three lists must have the same length, one entry per LED. The command is sent once per
control cycle and the newest command in a cycle wins.

```python
robot.set_rgb_cmd({"r": [0] * 6, "g": [255] * 6, "b": [0] * 6})
```

### B.5 State methods

`get_*(latest=True)` drains the buffer and returns the newest sample, discarding older ones;
`get_*(latest=False)` returns the oldest buffered sample and leaves the rest. Both return
`None` while no data has arrived.

#### `get_arm_state()`

`get_arm_state()` returns one arm-state sample, or `None` while no state has arrived. Same
layout as the manipulators — see [A.7](#a7-state-methods); `pose` is reported by the
controller.

**Usage**

```python
state = robot.get_arm_state()
if state is not None:
    print(state.arm_state.jnt.position)
```

**Return**

```python
HexDcRoboArmStateStamped: {
    "header": {
        "stamp": HexDcBaseTime,                      # {"secs": int, "nsecs": int}
    },
    "arm_state": {
        "jnt": {
            "position": np.ndarray,                  # (6,) float64, rad
            "velocity": np.ndarray,                  # (6,) float64, rad/s
            "effort": np.ndarray,                    # (6,) float64, Nm
        },
        "pose": {                                    # computed by the controller
            "position": HexDcBaseVector3,            # {"x": float, "y": float, "z": float}, m
            "orientation": HexDcBaseQuaternion,      # {"x": float, "y": float, "z": float, "w": float}
        },
    },
}
```

#### `get_grip_joy()`

`get_grip_joy()` returns one joystick-grip sample, or `None` while no data has arrived.

**Usage**

```python
joy = robot.get_grip_joy()
if joy is not None:
    print(joy.joystick_x, joy.trigger)
```

**Return**

```python
HelloJoystickState: {
    "ts_ns": int,                                    # ns, sample timestamp
    "joystick_x": float,                             # joystick X axis
    "joystick_y": float,                             # joystick Y axis
    "trigger": float,                                # trigger position
    "btn_w": bool,                                   # button state
    "btn_x": bool,                                   # button state
    "btn_y": bool,                                   # button state
    "btn_z": bool,                                   # button state
}
```

#### `get_dofs()`

`get_dofs()` returns the arm joint count.

**Usage**

```python
print(robot.get_dofs())
```

**Return**

```python
dict: {
    "arm": int,                                      # 6
}
```

---
