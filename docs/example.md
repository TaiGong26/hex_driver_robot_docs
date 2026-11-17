<h1 align="center">EXAMPLE</h1>

---

# Overview

This document shows how to use each robot model provided by `hex_driver_robot`. The snippets below follow the same structure as the scripts under `example/`.

**Note**: Install the required dependencies (`hex_device`, `hex_util_runtime`, `hex_util_robot`, `hex_util_urdf`, `hex_util_msg`) and point `host` / `port` at your running robot controller.

---

# Polling Examples

All polling examples share the same lifecycle:

1. Build a params object (`HexRobot*Y6Params`) with connection and model options.
2. Construct the driver, call `start()`, then poll state / send commands in a loop (typically with `HexRate` from `hex_util_runtime`).
3. On exit, call `stop()` inside `finally` so cleanup runs after `Ctrl+C`.

| Name | Robot model | Source | Notes |
|------|-------------|--------|-------|
| `archer_y6_example` | Archer Y6 | [example/archer_y6_example.py](../example/archer_y6_example.py) | Uses `ctrl_rate=500`, `state_buffer_size=200`, `grip_type="empty"`, joint position `set_arm_pos_cmd` + `set_grip_pos_cmd`. |
| `firefly_y6_example` | Firefly Y6 | [example/firefly_y6_example.py](../example/firefly_y6_example.py) | Same pattern as Archer; sample uses `port=9439`, `state_buffer_size=100`. |
| `hello_y6_example` | Hello Y6 | [example/hello_y6_example.py](../example/hello_y6_example.py) | Uses `led_buffer_size=10`; sends `set_rgb_cmd` and reads arm state + grip joystick state (no motor commands). |

---

## Running an Example

Activate the virtual environment (if you installed from source) and run the desired script:

```bash
source .venv/bin/activate
python example/archer_y6_example.py
python example/firefly_y6_example.py
python example/hello_y6_example.py
```

Press `Ctrl+C` to stop an example.

---

## Archer Y6 and Firefly Y6 (joint position + grip)

The repository examples drive the arm with joint position commands via `set_arm_pos_cmd()`, and send grip joint targets with `set_grip_pos_cmd()` when a grip is configured. The worker loop inside the driver runs at `2 * ctrl_rate`; the sample user loop uses `HexRate(2 * params.ctrl_rate)` to stay in step.

```python
import traceback
import numpy as np
from hex_util_runtime import ns_now, HexRate
from hex_driver_robot import HexRobotArcherY6, HexRobotArcherY6Params


def main() -> None:
    params = HexRobotArcherY6Params(
        host="172.18.23.197",
        port=8439,
        ctrl_rate=500,
        state_buffer_size=200,
        sens_ts=False,
        grip_type="empty",  # or "gp80" when a grip is attached
    )
    robot = None
    try:
        robot = HexRobotArcherY6(params)
        robot.start()
        print(f"dofs: {robot.get_dofs()}")

        arm_pos_tar = np.array([0.0, -1.5, 3.0, 0.0, 0.0, 0.0])
        grip_pos_tar = np.array([0.5])

        rate = HexRate(2 * params.ctrl_rate)
        while robot.is_working():
            rate.sleep()

            arm_state = robot.get_arm_state()
            if arm_state is not None:
                delay_ms = (ns_now() - arm_state["ts_ns"]) * 1e-6
                print(f"arm state delay: {delay_ms:.3f} ms")

            robot.set_arm_pos_cmd({
                "ts_ns": ns_now(),
                "jnt_pos": arm_pos_tar.copy(),
            })

            grip_state = robot.get_grip_state()
            if grip_state is not None:
                delay_ms = (ns_now() - grip_state["ts_ns"]) * 1e-6
                print(f"grip state delay: {delay_ms:.3f} ms")
            robot.set_grip_pos_cmd({
                "ts_ns": ns_now(),
                "jnt_pos": grip_pos_tar.copy(),
            })
    except KeyboardInterrupt:
        pass
    except Exception:
        traceback.print_exc()
    finally:
        if robot is not None:
            robot.stop()
```

For **Firefly Y6**, swap imports and types for `HexRobotFireflyY6` / `HexRobotFireflyY6Params` and set `port` to the value your controller exposes (the bundled example uses `9439`).

### Cartesian pose control (alternate)

To drive the arm in Cartesian space instead of joint space, replace the `set_arm_pos_cmd` call with `set_arm_pose_cmd`:

```python
arm_pose_tar = (
    np.array([0.3, 0.0, 0.3]),
    np.array([1.0, 0.0, 0.0, 0.0]),
)

robot.set_arm_pose_cmd({
    "ts_ns": ns_now(),
    "pose_pos": arm_pose_tar[0],
    "pose_quat": arm_pose_tar[1],
})
```

The arm state dict from `get_arm_state()` includes both motor fields and Cartesian end‑effector pose:
- `jnt_pos` (6,), `jnt_vel` (6,), `jnt_eff` (6,) – joint space
- `pose_pos` (3,), `pose_quat` (4,) – Cartesian space (forward kinematics)

---

## Hello Y6 (arm state + grip joystick + LEDs)

Hello Y6 exposes `set_rgb_cmd()` for the RGB LED strip. The grip is a joystick device that provides trigger, axis, and button inputs via `get_grip_joy()`. The sample sets a solid green LED command once after `start()`, then prints arm and grip joystick state in the loop.

```python
import traceback
import numpy as np
from hex_util_runtime import ns_now, HexRate
from hex_driver_robot import HexRobotHelloY6, HexRobotHelloY6Params


def main() -> None:
    params = HexRobotHelloY6Params(
        host="172.18.8.96",
        port=9439,
        ctrl_rate=500,
        state_buffer_size=200,
        led_buffer_size=10,
        sens_ts=False,
    )
    robot = None
    rgb_cmd = {
        "r": np.zeros(6, dtype=np.uint8),
        "g": 255 * np.ones(6, dtype=np.uint8),
        "b": np.zeros(6, dtype=np.uint8),
    }
    try:
        robot = HexRobotHelloY6(params)
        robot.start()
        print(f"dofs: {robot.get_dofs()}")
        robot.set_rgb_cmd(rgb_cmd)

        rate = HexRate(2 * params.ctrl_rate)
        while robot.is_working():
            rate.sleep()

            arm_state = robot.get_arm_state()
            if arm_state is not None:
                delay_ms = (ns_now() - arm_state["ts_ns"]) * 1e-6
                print(f"arm state delay: {delay_ms:.3f} ms")

            grip_joy = robot.get_grip_joy()
            if grip_joy is not None:
                delay_ms = (ns_now() - grip_joy["ts_ns"]) * 1e-6
                print(f"grip joy state delay: {delay_ms:.3f} ms")
    except KeyboardInterrupt:
        pass
    except Exception:
        traceback.print_exc()
    finally:
        if robot is not None:
            robot.stop()
```

---

# Callback Examples

The callback variants receive state data in real time via user‑supplied functions, rather than buffering into deques. This is useful when you want to process every state sample without polling.

| Name | Robot model | Source |
|------|-------------|--------|
| `archer_y6_callback_example` | Archer Y6 | [example/archer_y6_callback_example.py](../example/archer_y6_callback_example.py) |
| `firefly_y6_callback_example` | Firefly Y6 | [example/firefly_y6_callback_example.py](../example/firefly_y6_callback_example.py) |
| `hello_y6_callback_example` | Hello Y6 | [example/hello_y6_callback_example.py](../example/hello_y6_callback_example.py) |

## Archer Y6 / Firefly Y6 Callback

```python
import traceback
import numpy as np
from hex_util_runtime import ns_now, HexRate
from hex_driver_robot import HexRobotArcherY6Callback, HexRobotArcherY6Params


def arm_state_callback(arm_state) -> None:
    delay_ms = (ns_now() - arm_state["ts_ns"]) * 1e-6
    print(f"arm state delay: {delay_ms:.3f} ms")


def grip_state_callback(grip_state) -> None:
    delay_ms = (ns_now() - grip_state["ts_ns"]) * 1e-6
    print(f"grip state delay: {delay_ms:.3f} ms")


def main() -> None:
    params = HexRobotArcherY6Params(
        host="172.18.23.197",
        port=8439,
        ctrl_rate=500,
        state_buffer_size=200,
        sens_ts=False,
        grip_type="empty",
    )
    robot = None
    try:
        robot = HexRobotArcherY6Callback(
            params,
            callbacks={
                "arm_state": arm_state_callback,
                "grip_state": grip_state_callback,
            },
        )
        robot.start()
        print(f"dofs: {robot.get_dofs()}")

        arm_pos_tar = np.array([0.0, -1.5, 3.0, 0.0, 0.0, 0.0])
        grip_pos_tar = np.array([0.5])

        rate = HexRate(2 * params.ctrl_rate)
        while robot.is_working():
            rate.sleep()

            robot.set_arm_pos_cmd({
                "ts_ns": ns_now(),
                "jnt_pos": arm_pos_tar.copy(),
            })
            robot.set_grip_pos_cmd({
                "ts_ns": ns_now(),
                "jnt_pos": grip_pos_tar.copy(),
            })
    except KeyboardInterrupt:
        pass
    except Exception:
        traceback.print_exc()
    finally:
        if robot is not None:
            robot.stop()
```

## Hello Y6 Callback

```python
import traceback
import numpy as np
from hex_util_runtime import ns_now, HexRate
from hex_driver_robot import HexRobotHelloY6Callback, HexRobotHelloY6Params


def arm_state_callback(arm_state) -> None:
    delay_ms = (ns_now() - arm_state["ts_ns"]) * 1e-6
    print(f"arm state delay: {delay_ms:.3f} ms")


def grip_joy_callback(grip_joy) -> None:
    delay_ms = (ns_now() - grip_joy["ts_ns"]) * 1e-6
    print(f"grip joy state delay: {delay_ms:.3f} ms")


def main() -> None:
    params = HexRobotHelloY6Params(
        host="172.18.8.96",
        port=9439,
        ctrl_rate=500,
        state_buffer_size=200,
        led_buffer_size=10,
        sens_ts=False,
    )
    robot = None
    rgb_cmd = {
        "r": np.zeros(6, dtype=np.uint8),
        "g": 255 * np.ones(6, dtype=np.uint8),
        "b": np.zeros(6, dtype=np.uint8),
    }
    try:
        robot = HexRobotHelloY6Callback(
            params,
            callbacks={
                "arm_state": arm_state_callback,
                "grip_joy": grip_joy_callback,
            },
        )
        robot.start()
        print(f"dofs: {robot.get_dofs()}")
        robot.set_rgb_cmd(rgb_cmd)

        rate = HexRate(2 * params.ctrl_rate)
        while robot.is_working():
            rate.sleep()
    except KeyboardInterrupt:
        pass
    except Exception:
        traceback.print_exc()
    finally:
        if robot is not None:
            robot.stop()
```

---

# Troubleshooting

## Robot not connecting
- Ensure the robot controller is running and accessible at the specified IP address and port.
- Check network connectivity (ping the controller).
- Verify that the robot model matches the driver (e.g., Archer Y6 vs Firefly Y6) and that `port` matches the controller (Archer sample `8439`, Firefly sample `9439`).

## No states received
- Wait a moment after `start()` – the robot connection needs a few seconds to initialize.
- Increase `state_buffer_size` if you are reading slower than the control loop produces.

## High latency
- Use `latest=True` (the default) to always get the most recent state (drops older queued entries).
- Consider reducing the control rate (`ctrl_rate`) if real‑time performance is not critical.

## Import errors
- Make sure you have installed `hex_util_runtime`, `hex_util_robot`, `hex_util_urdf`, `hex_util_msg`, and `hex_device`.
- Check Python version (≥ 3.10).

## Command not accepted
- Verify that the command dictionary contains all required fields with correct array shapes (see [Arm API](api-arm.md) and [Chassis API](api-chassis.md)).
- For Archer/Firefly Y6 with a physical grip, set `grip_type="gp80"` and confirm `get_dofs()` reports a non‑zero grip DOF.
