<h1 align="center">HEXFELLOW ROBOT DRIVER</h1>

<p align="center">
    <a href="https://github.com/hexfellow/hex_driver_robot/stargazers">
        <img src="https://img.shields.io/github/stars/hexfellow/hex_driver_robot?style=flat-square&logo=github" />
    </a>
    <a href="https://github.com/hexfellow/hex_driver_robot/forks">
        <img src="https://img.shields.io/github/forks/hexfellow/hex_driver_robot?style=flat-square&logo=github" />
    </a>
    <a href="https://doi.org/10.5281/zenodo.18309954">
        <img src="https://zenodo.org/badge/1088506315.svg" alt="DOI">
    </a>
    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
    <a href="https://github.com/hexfellow/hex_driver_robot/issues">
        <img src="https://img.shields.io/github/issues/hexfellow/hex_driver_robot?style=flat-square&logo=github" />
    </a>
    &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
    <img src="https://img.shields.io/badge/python-≥3.10-blue?style=flat-square&logo=python" />
</p>

---

# 📖 Overview

## What is `hex_driver_robot`

`hex_driver_robot` is a Python library for controlling HEXFELLOW robots with a unified interface. It supports the following robot models:

| Model | Arm DOF | Grip | Special Features |
|-------|---------|------|-----------------|
| **Archer Y6** | 6 | Optional (GP80 / none) | MIT / joint position / Cartesian pose commands |
| **Firefly Y6** | 6 | Optional (GP80 / none) | MIT / joint position / Cartesian pose commands |
| **Firefly Y6 H2_40** | 6 | Optional (GP80 / none) | Same interface as Firefly Y6; drives `RtArmFireflyY6_H2_40` (robot type 39) via `HexRobotFireflyY6H2_40` |
| **Hello Y6** | 6 | Joystick grip | RGB LED strip control, joystick input |

The library runs each robot controller in a background thread, streams robot states via deque ring buffers, and exposes consistent readers (`get_arm_state`, `get_grip_state` on Archer/Firefly, or `get_grip_joy` on Hello Y6) plus model‑specific command writers: MIT impedance, joint position, and Cartesian pose commands on Archer and Firefly, and grip RGB LED control on Hello Y6. Each model also provides a callback variant for event‑driven state handling.

## What problem it solves

- **Unified robot interface**: Use the same API pattern for different robot models (Hello Y6, Archer Y6, Firefly Y6).
- **Background control**: Each robot controller runs in its own thread; the main thread only reads the latest state when needed.
- **Low‑latency sharing**: States and commands are passed through deque ring buffers, avoiding serialization overhead.
- **Real‑time control**: Archer and Firefly support MIT impedance, MIT with compensation, joint position, and Cartesian pose commands; Hello Y6 focuses on motor telemetry, joystick input, and LED control on the grip.
- **Callback support**: Every model offers a callback class for event‑driven programming alongside the polling‑based class.

## Target users

- Robotics researchers prototyping manipulation pipelines with HEXFELLOW robots.
- Developers who need a simple, consistent way to interface with real robot hardware.
- Anyone who wants to switch between different robot models without changing client code.

## Project structure

```
hex_driver_robot/
├── hex_driver_robot/
│   ├── __init__.py              # Package exports (all public classes)
│   ├── base.py                  # HexRobotBase (abstract) & HexRobotBaseParams
│   ├── robot_archer_y6/
│   │   ├── __init__.py
│   │   └── robot.py             # Archer Y6 driver + callback + params
│   ├── robot_firefly_y6/
│   │   ├── __init__.py
│   │   └── robot.py             # Firefly Y6 driver + callback + params
│   └── robot_hello_y6/
│       ├── __init__.py
│       └── robot.py             # Hello Y6 driver + callback + params
├── example/
│   ├── archer_y6_example.py
│   ├── archer_y6_callback_example.py
│   ├── firefly_y6_example.py
│   ├── firefly_y6_callback_example.py
│   ├── firefly_y6_h2_40_example.py
│   ├── hello_y6_example.py
│   └── hello_y6_callback_example.py
└── docs/
    ├── api-arm.md               # Arm API reference (motion arms and Hello Y6)
    ├── api-chassis.md           # Chassis API reference
    └── example.md               # Usage walkthroughs
```

# 📦 Installation

## Requirements

- **Python** ≥ 3.10
- **OS**: `Linux` (macOS/Windows may work but are less tested)
- **Core dependencies**:
  - `hex_util_runtime` ≥ 0.0.0, < 0.1.0
  - `hex_util_robot` ≥ 0.0.0, < 0.1.0
  - `hex_util_urdf` ≥ 0.0.0, < 0.1.0
  - `hex_util_msg` ≥ 0.0.0, < 0.1.0
  - `hex_device` ≥ 1.4.0, < 1.5.0

## Install from PyPI

```bash
pip install hex_driver_robot
```

## Install from Source

For those who need examples, you can install the package from source code with examples.

**Note**: We use [**uv**](https://github.com/astral-sh/uv) to manage the Python environment. Please install it first.

1. Clone and install in editable mode. The `venv.sh` script expects [uv](https://github.com/astral-sh/uv).

    ```bash
    git clone https://github.com/hexfellow/hex_driver_robot.git
    cd hex_driver_robot
    ./venv.sh
    ```

   - `./venv.sh` — creates `.venv`, installs `hex_driver_robot` in editable mode.

2. Activate before running examples:

    ```bash
    source .venv/bin/activate
    ```

# 📑 API

See [Arm API](docs/api-arm.md) for arm driver classes, parameters and commands, and [Chassis API](docs/api-chassis.md) for chassis driver classes and commands.

# 💡 Example

See [**Example**](docs/example.md) for usage aligned with [`example/archer_y6_example.py`](example/archer_y6_example.py), [`example/firefly_y6_example.py`](example/firefly_y6_example.py), and [`example/hello_y6_example.py`](example/hello_y6_example.py).

# 📄 License

Apache License 2.0. See [LICENSE](LICENSE).

# 🌟 Star History

[![Star History Chart](https://api.star-history.com/svg?repos=hexfellow/hex_driver_robot&type=Date)](https://star-history.com/#hexfellow/hex_driver_robot&Date)

# 👥 Contributors

<a href="https://github.com/hexfellow/hex_driver_robot/graphs/contributors">
    <img src="https://contrib.rocks/image?repo=hexfellow/hex_driver_robot" />
</a>
