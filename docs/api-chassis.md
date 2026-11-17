<h1 align="center">CHASSIS API</h1>

This page documents every chassis class in `hex_driver_robot`: what to import, how to start
a driver, which methods each chassis has, and what limits travel with each command. 

## Contents

- [1. Quick start](#1-quick-start)
- [2. Parameters](#2-parameters)
- [3. Driver classes](#3-driver-classes)
- [4. All methods](#4-all-methods)
- [5. Recommended control commands](#5-recommended-control-commands)
- [6. Command methods](#6-command-methods)
  - [`set_chs_vel_cmd()`](#set_chs_vel_cmd)
  - [`set_chs_per_motor_spd_cmd()`](#set_chs_per_motor_spd_cmd)
  - [`set_chs_mit_cmd()`](#set_chs_mit_cmd)
  - [`clear_odom_bias()`](#clear_odom_bias)
- [7. State methods](#7-state-methods)
  - [`get_chassis_motor_status()`](#get_chassis_motor_status)
  - [`get_battery_info()`](#get_battery_info)
  - [`get_vehicle_speed()`](#get_vehicle_speed)
  - [`get_vehicle_position()`](#get_vehicle_position)
  - [`get_warning()`](#get_warning)
  - [`get_status_summary()`](#get_status_summary)

The four chassis classes differ in one thing only — the per-motor command they accept:

| Class | Chassis | Motors | Motion commands |
|---|---|:--:|---|
| `HexRobotTriggerA3Lr1` | Trigger A3 LR1, three omni wheels | 3 | `set_chs_vel_cmd()`, `set_chs_per_motor_spd_cmd()` |
| `HexRobotTriggerA3H1` | Trigger A3 H1 | 3 | `set_chs_vel_cmd()`, `set_chs_mit_cmd()` |
| `HexRobotMaverX4H1` | Maver X4 H1 | 8 | `set_chs_vel_cmd()`, `set_chs_mit_cmd()` |
| `HexRobotMaverL4H1` | Maver L4 H1 | 8 | `set_chs_vel_cmd()`, `set_chs_mit_cmd()` |

Everything else — whole-chassis velocity and state getters — is shared.

---

## 1. Quick start

```python
from hex_util_runtime import HexRate
from hex_driver_robot import HexRobotTriggerA3Lr1, HexRobotTriggerA3Lr1Params

params = HexRobotTriggerA3Lr1Params(
    host="192.168.1.100",   # controller address
    port=8439,              # controller port
    ctrl_rate=500,          # control loop rate [Hz]
)
robot = None
try:
    robot = HexRobotTriggerA3Lr1(params)
    robot.start()                                   # opens the session, starts the control loop

    rate = HexRate(params.ctrl_rate)
    while robot.is_working():
        rate.sleep()
        robot.set_chs_vel_cmd({"vx": 0.0, "vy": 0.0, "omega": 0.1})   # m/s, m/s, rad/s
        speed = robot.get_vehicle_speed()
        if speed is not None:
            print(speed)
finally:
    if robot is not None:
        robot.stop()                                                  # ends the session, joins the thread
```

Three things are enough to get moving:

1. **Send a command, then keep looping.** A motion command is latched: the driver re-sends
   the last one every cycle. 
2. **Read state in the same loop.** `get_*()` returns `None` until data has arrived, so
   always check before using a value.
3. **Put `robot.stop()` in `finally`.** It ends the session and joins the control thread.

## 2. Parameters

Every chassis parameter object extends `HexRobotBaseParams`:

| Field | Default | Unit | Meaning |
|---|---|---|---|
| `host` | `"192.168.1.100"` | — | Controller address |
| `port` | `8439` | — | Controller port |
| `ctrl_rate` | `500` | Hz | Control loop rate |
| `state_buffer_size` | `100` | — | Depth of the state ring buffers |
| `sens_ts` | `True` | — | Stamp states with the controller clock instead of the local clock |

Chassis parameters add:

| Field | Default | Meaning |
|---|---|---|
| `motor_count` | `3` (Trigger A3) / `8` (Maver) | Number of chassis motors; fixed per class, do not override |
| `robot_type` | model-specific | Which chassis model to connect to; fixed per class, do not override |
| `init_timeout` | `10.0` | Seconds to wait for the first motor state before giving up |
| `enable_kcp` | `False` | Use the reliable transport when the network needs it |
| `log_level` | `"INFO"` | Driver log level |

| Class | Parameters class |
|---|---|
| `HexRobotTriggerA3Lr1` | `HexRobotTriggerA3Lr1Params` |
| `HexRobotTriggerA3H1` | `HexRobotTriggerA3H1Params` |
| `HexRobotMaverX4H1` | `HexRobotMaverX4H1Params` |
| `HexRobotMaverL4H1` | `HexRobotMaverL4H1Params` |

## 3. Driver classes

Each chassis ships three classes with the same name prefix:

| Class | Role |
|---|---|
| `HexRobot<Model>` | Polling driver: `get_*()` methods read from internal ring buffers |
| `HexRobot<Model>Callback` | Same driver, but state is pushed to callables you pass to the constructor |
| `HexRobot<Model>Params` | Parameter object described above |

Lifecycle methods, available on all of them:

| Method | Returns | Meaning |
|---|---|---|
| `start()` | `None` | Opens the session and starts the background control loop |
| `stop()` | `None` | Ends the session and joins the control loop; call it from `finally` |
| `is_working()` | `bool` | `True` while the control loop runs — the natural `while` condition |

## 4. All methods

✅ = the method exists on this chassis; — = it does not, and calling it raises
`AttributeError`.

| Method | Kind | Trigger A3 LR1 | Trigger A3 H1 | Maver X4 H1 | Maver L4 H1 |
|---|---|:--:|:--:|:--:|:--:|
| `set_chs_vel_cmd()` | Command | ✅ | ✅ | ✅ | ✅ |
| `set_chs_per_motor_spd_cmd()` | Command | ✅ | — | — | — |
| `set_chs_mit_cmd()` | Command | — | ✅ | ✅ | ✅ |
| `get_chassis_motor_status()` | State | ✅ | ✅ | ✅ | ✅ |
| `get_battery_info()` | State | ✅ | ✅ | ✅ | ✅ |
| `get_vehicle_speed()` | State | ✅ | ✅ | ✅ | ✅ |
| `get_vehicle_position()` | State | ✅ | ✅ | ✅ | ✅ |
| `get_warning()` | State | ✅ | ✅ | ✅ | ✅ |
| `get_status_summary()` | State | ✅ | ✅ | ✅ | ✅ |
| `clear_odom_bias()` | System | ✅ | ✅ | ✅ | ✅ |
| `start()` / `stop()` / `is_working()` | Lifecycle | ✅ | ✅ | ✅ | ✅ |

## 5. Recommended control commands

| Method | Level | Reason |
|---|---|---|
| `set_chs_vel_cmd()` | Recommended | One velocity command for the whole chassis; the wheel speeds are resolved by the controller |
| `set_chs_per_motor_spd_cmd()` | Not recommended | Per-wheel speed targets; the command carries a current cap but no acceleration limit |
| `set_chs_mit_cmd()` | Not recommended | Per-motor torque and damping; the command carries no motion limit, and its stiffness term has no effect (see [`set_chs_mit_cmd()`](#set_chs_mit_cmd)) |

The state methods are read-only and safe to call at any time.

## 6. Command methods

All motion commands take a single `dict`. Unlisted keys keep the default shown. `ts_ns`
(default `ns_now()`) is the command timestamp and rarely needs setting.

### `set_chs_vel_cmd()`

`set_chs_vel_cmd(cmd)` sets the velocity of the chassis as one body. Available on all four classes.

**Usage**

| Key | Type | Default | Unit | Meaning |
|---|---|---|---|---|
| `vx` | float | `0.0` | m/s | Linear velocity, x |
| `vy` | float | `0.0` | m/s | Linear velocity, y |
| `omega` | float | `0.0` | rad/s | Angular velocity around z |
| `ts_ns` | int | `ns_now()` | ns | Command timestamp |

```python
robot.set_chs_vel_cmd({"vx": 0.0, "vy": 0.0, "omega": 0.2})
```

### `set_chs_per_motor_spd_cmd()`

`set_chs_per_motor_spd_cmd(cmd)` sets a per-wheel speed with a current cap. `HexRobotTriggerA3Lr1` only.

**Usage**

| Key | Type | Default | Unit | Meaning |
|---|---|---|---|---|
| `speed` | list of 3 floats | zeros | rad/s | Target wheel speed |
| `max_current` | list of 3 floats | `1.0` each | A | Current limit per wheel |

```python
robot.set_chs_per_motor_spd_cmd({"speed": [1.0, 1.0, 1.0], "max_current": [2.0, 2.0, 2.0]})
```

The command carries a current cap but no acceleration limit, so a step change in `speed`
reaches the wheels directly. Prefer [`set_chs_vel_cmd()`](#set_chs_vel_cmd) unless you
specifically need per-wheel speeds.

On `HexRobotTriggerA3H1`, `HexRobotMaverX4H1` and `HexRobotMaverL4H1` this method does not
exist: calling it raises `AttributeError`.

### `set_chs_mit_cmd()`

`set_chs_mit_cmd(cmd)` sets per-motor direct impedance. `HexRobotTriggerA3H1`,
`HexRobotMaverX4H1`, `HexRobotMaverL4H1` only.

**Usage**

| Key | Type | Default | Unit | Meaning |
|---|---|---|---|---|
| `jnt_pos` | list of `motor_count` floats | zeros | rad | Target motor position |
| `jnt_vel` | list of `motor_count` floats | zeros | rad/s | Target motor velocity |
| `mit_tau` | list of `motor_count` floats | zeros | Nm | Feed-forward torque |
| `mit_kp` | list of `motor_count` floats | zeros | Nm/rad | Stiffness — has no effect, see the note below |
| `mit_kd` | list of `motor_count` floats | zeros | Nm/(rad/s) | Damping |

Every array must have exactly `motor_count` entries (`3` on Trigger A3 H1, `8` on both
Maver classes); a different length raises `ValueError`.

> **Firmware limitation.** The controller forces `kp = 0` for chassis MIT commands, so only
> torque and damping take effect. A position target therefore does **not hold** a wheel.

`HexRobotTriggerA3Lr1` has no such method — that chassis takes wheel speeds instead.

```python
robot.set_chs_mit_cmd({
    "jnt_pos": [0.0] * 8,
    "jnt_vel": [0.5] * 8,
    "mit_kp": [0.0] * 8,
    "mit_kd": [3.0] * 8,
})
```

### `clear_odom_bias()`

`clear_odom_bias()` resets the odometry origin to the current position, so
`get_vehicle_position()` restarts from that point. It is sent once and is not repeated.

**Usage**

```python
robot.clear_odom_bias()
```

## 7. State methods

The getters take no arguments. `get_chassis_motor_status()`, `get_vehicle_speed()`,
`get_vehicle_position()` and `get_warning()` return `None` until data has arrived, so
always check before using a value. `get_battery_info()` and `get_status_summary()` always
return a `dict`.

### `get_chassis_motor_status()`

`get_chassis_motor_status()` returns the motor state, or `None` when no motor data has
arrived. A timestamp key may be absent when the clock has not been synchronised yet.

**Usage**

```python
status = robot.get_chassis_motor_status()
if status is not None:
    print(status["pos"], status["error"])
```

**Return**

```python
dict: {
    "pos": np.ndarray,                               # (n,) float64, rad
    "vel": np.ndarray,                               # (n,) float64, rad/s
    "eff": np.ndarray,                               # (n,) float64, Nm
    "motor_temp": np.ndarray,                        # (n,) float64, °C, nan when not reported
    "driver_temp": np.ndarray,                       # (n,) float64, °C, nan when not reported
    "error": list[list[str]],                        # error names per motor, empty list = no error
    "ts": dict,                                      # {"s": int, "ns": int}, optional, may be absent
}
```

### `get_battery_info()`

`get_battery_info()` always returns a `dict`. Before the first state arrives the values are
`0.0`, `0` and `None`.

**Usage**

```python
battery = robot.get_battery_info()
print(battery["voltage"], battery["thousandth"])
```

**Return**

```python
dict: {
    "voltage": float,                                # V, battery voltage
    "thousandth": int,                               # charge level, 1000 = full
    "charging": bool,                                # None before the controller reports it
}
```

### `get_vehicle_speed()`

`get_vehicle_speed()` returns the chassis speed, or `None` until odometry has arrived.

**Usage**

```python
speed = robot.get_vehicle_speed()
if speed is not None:
    print(speed)
```

**Return**

```python
tuple[float, float, float]: (
    vx,                                              # m/s, linear velocity x
    vy,                                              # m/s, linear velocity y
    omega,                                           # rad/s, angular velocity z
)
```

### `get_vehicle_position()`

`get_vehicle_position()` returns the odometry position relative to the origin set by
[`clear_odom_bias()`](#clear_odom_bias), or `None` until odometry has arrived. This is
odometry, not a measurement, and it drifts.

**Usage**

```python
position = robot.get_vehicle_position()
if position is not None:
    print(position)
```

**Return**

```python
tuple[float, float, float]: (
    x,                                               # m, relative x
    y,                                               # m, relative y
    yaw,                                             # rad, relative yaw
)
```

### `get_warning()`

`get_warning()` returns the current warning, or `None` when there is none.

**Usage**

```python
warning = robot.get_warning()
```

**Return**

```python
int: warning category     # None when there is no warning
```

### `get_status_summary()`

`get_status_summary()` always returns one `dict` with the values below.

**Usage**

```python
summary = robot.get_status_summary()
print(summary["battery"], summary["vehicle_speed"])
```

**Return**

```python
dict: {
    "name": str,                                     # chassis name
    "motor_count": int,                              # number of chassis motors
    "battery": dict,                                 # same as get_battery_info()
    "vehicle_speed": tuple[float, float, float],      # same as get_vehicle_speed()
    "vehicle_position": tuple[float, float, float],   # same as get_vehicle_position()
    "warning": int,                                  # current warning, None when none
    "motor_status": dict,                            # same as get_chassis_motor_status()
}
```

A `dict` from `get_chassis_motor_status()` means motor data has arrived and the first
command can be sent.

---

