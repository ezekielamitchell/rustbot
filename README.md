# rustbot

`rustbot` is a small differential-drive unmanned ground vehicle built around a Raspberry Pi-class computer. The project separates real-time-adjacent control and safety logic from perception and detection logic:

- **Rust** owns the control plane: motor control, GPIO/PWM, watchdog timeout, command validation, and safety-critical logic.
- **Python** owns the perception plane: camera input, OpenCV processing, detection/tracking, target selection, and command generation.

Python sends high-level tracking commands to Rust over a simple IPC boundary. The initial transport is newline-delimited JSON over TCP. Rust validates every command and remains authoritative for motor safety and execution.

The primary goal is to build a low-latency, low-false-positive vision-to-action loop on edge hardware while keeping motor safety independent from perception reliability.

## Mission

Build a modular edge robotics platform for controlled vision-guided movement, starting with a simple tracking loop and expanding toward autonomous recon, threat-aware perception, and navigation in constrained environments.

The first version prioritizes:

- Safe differential-drive control
- Clear separation between perception and actuation
- Low-latency camera-to-command behavior
- Conservative command validation
- Robust handling of stale commands, invalid commands, crashes, and communication loss
- A codebase that can grow into mapping, navigation, and more advanced perception without rewriting the safety core

## System Architecture

```text
+-----------------------------+          JSON over TCP          +-----------------------------+
|        Python Plane         |  -----------------------------> |          Rust Plane         |
|                             |                                 |                             |
|  Camera Input               |                                 |  TCP Command Server         |
|  OpenCV Processing          |                                 |  Command Validation         |
|  Detection / Tracking       |                                 |  Watchdog Timeout           |
|  Target Selection           |                                 |  Safety State Machine       |
|  Command Generation         |                                 |  Motor Control              |
|                             |                                 |  GPIO / PWM                 |
+-------------+---------------+                                 +--------------+--------------+
              |                                                                |
              v                                                                v
        Camera Sensor                                                   Motor Driver
                                                                            |
                                                                            v
                                                                  Differential Drive Base
```

The architecture is intentionally asymmetric:

- Python may suggest movement.
- Rust decides whether movement is safe.
- Rust owns all motor output.
- Loss of Python, malformed messages, stale commands, or IPC failure must result in a stop condition.

The perception plane is replaceable. The control plane is conservative and authoritative.

## Hardware

Initial target hardware:

- Raspberry Pi-class SBC
- Differential-drive chassis
- Left/right DC motors
- Motor driver supporting PWM and direction control
- Camera module or USB camera
- Battery pack sized for motors and compute
- Optional voltage/current monitoring
- Optional hardware emergency stop
- Optional wheel encoders for later closed-loop control

Minimum expected hardware interfaces:

```text
Camera -> Python perception process
GPIO/PWM -> Rust control process -> Motor driver -> Motors
```

Recommended safety hardware:

- Physical power switch
- Motor power cutoff
- Emergency stop button
- Fuse or current protection
- Separate motor and logic power rails where appropriate
- Clearly labeled battery disconnect

## Software Stack

### Rust Control Plane

Responsibilities:

- TCP command server
- JSON command parsing
- Command schema validation
- Motor command limiting
- GPIO/PWM output
- Watchdog timeout
- Startup/shutdown safety behavior
- Fault state handling
- Safety-critical logic

Expected crates may include:

- `tokio` for async networking
- `serde` / `serde_json` for message parsing
- `gpio-cdev`, `rppal`, or platform-specific GPIO/PWM access
- `tracing` for logs
- `anyhow` / `thiserror` for error handling

### Python Perception Plane

Responsibilities:

- Camera input
- OpenCV frame processing
- Detection/tracking
- Target selection
- Command generation
- TCP client to Rust control server

Expected packages may include:

- `opencv-python`
- `numpy`
- `pydantic` or `dataclasses` for command models
- `pyyaml` for configuration
- `pytest` for tests

### Why Rust + Python

Rust is used where correctness, explicit state handling, and safe execution matter most: motor control, validation, watchdogs, and the safety envelope.

Python is used where iteration speed and library availability matter most: camera input, OpenCV experiments, model integration, and perception pipeline development.

The boundary between them is deliberate. Perception can fail, crash, or produce noisy output without directly controlling the motors.

## Repository Structure

```text
rustbot/
├── rust/
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs
│   │   ├── control/
│   │   ├── gpio/
│   │   ├── ipc/
│   │   ├── motor/
│   │   ├── safety/
│   │   └── validation/
│   └── tests/
├── python/
│   ├── pyproject.toml
│   ├── rustbot_perception/
│   │   ├── camera.py
│   │   ├── detector.py
│   │   ├── tracker.py
│   │   ├── command_client.py
│   │   └── main.py
│   └── tests/
├── config/
│   ├── control.yaml
│   ├── detector.yaml
│   └── robot.toml
├── interfaces/
│   ├── tracking_command.schema.json
│   └── README.md
├── docs/
│   ├── bringup.md
│   ├── safety.md
│   └── architecture.md
├── scripts/
│   ├── run_control.sh
│   ├── run_perception.sh
│   └── bench_latency.sh
└── README.md
```

## Getting Started

This repository is being built from the ground up. The commands below describe the intended development workflow.

### Prerequisites

Install system dependencies:

```bash
sudo apt update
sudo apt install -y \
  build-essential \
  pkg-config \
  libopencv-dev \
  python3 \
  python3-venv \
  python3-pip
```

Install Rust:

```bash
curl https://sh.rustup.rs -sSf | sh
rustup default stable
```

Create a Python virtual environment:

```bash
cd python
python3 -m venv .venv
source .venv/bin/activate
pip install -U pip
pip install -e ".[dev]"
```

Build the Rust control plane:

```bash
cd rust
cargo build
```

Run Rust tests:

```bash
cargo test
```

Run Python tests:

```bash
cd python
pytest
```

## Bring-Up Sequence

The robot should be brought up incrementally. Do not begin with autonomous movement.

### First-Run Workflow

1. Bench test without motor power.
   - Boot the SBC.
   - Confirm camera access.
   - Confirm the Rust control process starts.
   - Confirm startup state reports motors off.

2. Validate IPC without motors.
   - Start the Rust control server.
   - Send valid JSON commands from a test client.
   - Send invalid JSON and confirm rejection.
   - Stop sending commands and confirm watchdog timeout enters stop state.

3. Enable motor driver with wheels off ground.
   - Apply motor power.
   - Keep the robot physically restrained or elevated.
   - Send low-speed commands.
   - Confirm left/right motor direction.
   - Confirm stop behavior on stale commands.

4. Run perception without movement.
   - Start camera pipeline.
   - Run detector/tracker.
   - Log generated commands without transmitting them.

5. Run perception-to-control loop at low limits.
   - Enable command transmission.
   - Use conservative speed caps.
   - Keep a physical kill switch available.
   - Verify stop on Python crash or TCP disconnect.

6. Ground test in controlled space.
   - Use low speed.
   - Avoid people, pets, stairs, roads, and fragile objects.
   - Increase limits only after repeated safe runs.

## IPC Message Protocol

The initial command protocol is newline-delimited JSON over TCP.

Python connects to the Rust control server and sends tracking command messages. Rust validates every message before updating motor intent.

### Example Tracking Command

```json
{
  "type": "tracking_command",
  "version": 1,
  "timestamp_ms": 1730000000000,
  "command_id": "cmd-000042",
  "target": {
    "class": "person",
    "confidence": 0.82,
    "bbox": {
      "x": 318,
      "y": 146,
      "w": 92,
      "h": 210
    },
    "center_x_norm": 0.49,
    "center_y_norm": 0.53
  },
  "motion": {
    "linear": 0.18,
    "angular": -0.08
  },
  "ttl_ms": 250
}
```

### Command Rules

Rust must reject commands that are:

- Not valid JSON
- Missing required fields
- Using an unsupported protocol version
- Outside configured speed limits
- Outside configured angular limits
- Older than the allowed command age
- Carrying a TTL greater than the configured maximum
- Based on insufficient confidence
- Internally inconsistent or malformed

### Stale Command Behavior

A stale command is a stop condition.

If Rust does not receive a valid command within the configured watchdog interval, it must stop motor output. Python crash, process stall, network disconnect, or camera failure must not result in runaway motion.

```text
valid command received -> update motor intent
invalid command        -> reject, keep safe state
command timeout        -> stop
IPC disconnect         -> stop
startup                -> motors off
shutdown               -> motors off
```

## Configuration

Configuration should be explicit, checked into version control where safe, and separated by responsibility.

### `config/control.yaml`

```yaml
ipc:
  bind_host: "127.0.0.1"
  bind_port: 8765
  protocol_version: 1
  max_message_bytes: 4096

watchdog:
  command_timeout_ms: 300
  startup_motor_state: "off"
  stop_on_disconnect: true
  stop_on_invalid_burst: true
  invalid_burst_limit: 5

limits:
  max_linear_mps: 0.35
  max_angular_radps: 0.9
  max_command_ttl_ms: 300
  min_target_confidence: 0.65

motor:
  pwm_frequency_hz: 1000
  left_trim: 1.0
  right_trim: 1.0
  deadband: 0.08
```

### `config/detector.yaml`

```yaml
camera:
  device_index: 0
  width: 640
  height: 480
  fps: 30

detector:
  backend: "opencv"
  model: "simple_color_or_motion"
  min_confidence: 0.65
  max_targets: 4

tracking:
  target_classes:
    - "person"
  center_deadband_norm: 0.08
  lost_target_timeout_ms: 500
  smoothing_alpha: 0.35

command_generation:
  send_rate_hz: 20
  linear_gain: 0.25
  angular_gain: 0.8
  max_linear_mps: 0.25
  max_angular_radps: 0.7
  command_ttl_ms: 250
```

### `config/robot.toml`

```toml
[robot]
name = "rustbot"
drive = "differential"
wheel_base_m = 0.18
wheel_diameter_m = 0.065

[gpio.left_motor]
pwm_pin = 12
dir_pin_a = 5
dir_pin_b = 6

[gpio.right_motor]
pwm_pin = 13
dir_pin_a = 20
dir_pin_b = 21

[safety]
require_control_process = true
motors_off_at_startup = true
motors_off_at_shutdown = true
stale_command_means_stop = true
invalid_command_policy = "reject"
```

## Development Workflow

Use short-lived branches and keep control-plane changes especially reviewable.

Recommended branch naming:

```text
main
feature/rust-ipc-server
feature/python-camera-loop
feature/control-watchdog
fix/motor-stop-on-disconnect
docs/bringup-procedure
```

Suggested workflow:

1. Open a branch from `main`.
2. Keep changes scoped to one subsystem when possible.
3. Add or update tests with behavior changes.
4. Run Rust and Python test suites before merging.
5. Document safety-relevant behavior in `docs/safety.md`.
6. Prefer small pull requests for motor control, watchdog, and validation changes.

Control-plane changes should be reviewed more strictly than perception experiments.

## Testing Strategy

Testing should focus on safety behavior first, then perception quality and latency.

### Rust Tests

Rust tests should cover:

- Command parsing
- Schema validation
- Range checks
- TTL handling
- Watchdog timeout behavior
- Stop-on-disconnect behavior
- Startup motor state
- Invalid command rejection
- Motor output limiting
- Safety state transitions

Example:

```bash
cd rust
cargo test
cargo clippy --all-targets --all-features
cargo fmt --check
```

### Python Tests

Python tests should cover:

- Camera abstraction behavior
- Detector output format
- Target selection
- Command generation
- Confidence thresholds
- Command serialization
- TCP client reconnect behavior

Example:

```bash
cd python
pytest
ruff check .
ruff format --check .
```

### Integration Tests

Integration tests should verify the IPC boundary:

- Python sends valid tracking command
- Rust accepts valid command
- Rust rejects invalid command
- Rust stops after command timeout
- Rust stops when Python disconnects
- Rust remains stopped at startup until valid command arrives

### Hardware Tests

Hardware tests should be staged:

1. No motor power
2. Wheels elevated
3. Low-speed ground test
4. Controlled tracking test
5. Latency and false-positive evaluation

## Safety Rules

The control plane must enforce these rules regardless of perception output:

- Startup state is motors off.
- Shutdown state is motors off.
- Stale command means stop.
- IPC disconnect means stop.
- Python crash means stop.
- Invalid commands are rejected.
- Commands outside configured limits are rejected or clamped according to explicit policy.
- Motor output must never depend directly on unvalidated perception data.
- The Rust process is authoritative for motor execution.
- Perception confidence thresholds must be configurable.
- The robot must be tested with wheels off the ground before any live-drive test.
- A physical power cutoff should be available during bring-up.
- Autonomous behavior should only be tested in controlled environments.

No perception model should be treated as safety-critical unless it is wrapped by conservative control-plane constraints.

## Known Risks / Failure Modes

| Risk | Expected Behavior | Mitigation |
| --- | --- | --- |
| Python process crashes | Robot stops | Rust watchdog timeout |
| TCP connection drops | Robot stops | Stop-on-disconnect policy |
| Camera freezes | Robot stops after stale commands | Command TTL and watchdog |
| Detector false positive | Motion limited or rejected | Confidence threshold, target filters, speed caps |
| Invalid JSON command | Command rejected | Strict schema validation |
| Command spam or malformed burst | Enter safe state | Invalid burst limit |
| GPIO/PWM initialization failure | Motors remain off | Startup safety checks |
| Motor direction reversed | Detected during elevated test | Bring-up checklist |
| Latency too high | Reduced tracking quality | Measure loop timing and tune pipeline |

## Current Status

- [ ] Repository scaffold created
- [ ] Rust control server initialized
- [ ] Rust command schema implemented
- [ ] Rust watchdog implemented
- [ ] Motor driver abstraction implemented
- [ ] GPIO/PWM hardware tested
- [ ] Python camera pipeline initialized
- [ ] Python detector prototype implemented
- [ ] Python tracking command client implemented
- [ ] JSON IPC integration tested
- [ ] Wheels-off motor test completed
- [ ] Low-speed ground test completed
- [ ] Latency benchmark added
- [ ] Safety documentation written

## Roadmap

### Phase 1: Foundation

- Create repository structure
- Define IPC message schema
- Implement Rust TCP command server
- Implement command validation
- Implement watchdog timeout
- Implement motor abstraction
- Add basic Python TCP client
- Add basic camera capture loop

### Phase 2: First Vision-to-Action Loop

- Add simple OpenCV detector/tracker
- Generate high-level tracking commands
- Validate full Python-to-Rust command path
- Test stale command stop behavior
- Benchmark camera-to-command latency
- Run low-speed tracking in a controlled environment

### Phase 3: Reliability

- Add structured logs
- Add integration tests
- Add reconnect handling
- Add detector confidence tuning
- Add false-positive suppression
- Add runtime status reporting
- Add hardware bring-up documentation

### Phase 4: Navigation Expansion

- Add wheel encoder support
- Add closed-loop speed control
- Add obstacle detection
- Add local navigation behaviors
- Add mapping experiments
- Add higher-level autonomy modes

### Phase 5: Recon and Threat-Aware Perception

- Add richer object detection
- Add target classification policies
- Add event recording
- Add environment annotation
- Add operator review tools
- Add constrained-environment navigation behaviors

## Near-Term Milestones

1. Define `tracking_command.schema.json`.
2. Implement Rust command parser and validator.
3. Implement watchdog-driven stop behavior.
4. Implement Python command client with reconnect handling.
5. Add a synthetic command sender for bench testing.
6. Add camera capture and frame display.
7. Add a simple detector/tracker prototype.
8. Measure end-to-end loop latency.
9. Complete wheels-off motor validation.
10. Document first controlled ground test results.

## Design Principles

### Safety First

Motor behavior must remain safe when perception fails. The robot should prefer stopping over guessing.

### Separation of Concerns

Perception and control are separate processes with a narrow, explicit interface. Python does not directly drive GPIO or motors.

### Rust Is Authoritative

Rust owns validation, safety state, watchdog behavior, and motor execution. Python sends intent, not authority.

### Start Simple

The first version should use simple JSON over TCP, simple tracking commands, and conservative limits. More advanced protocols and autonomy can be added after the safety loop is proven.

### Make Failure Explicit

Crashes, disconnects, stale data, invalid messages, and low-confidence detections should have defined behavior.

### Measure Latency

The project goal depends on fast perception-to-action behavior. Latency should be measured early and repeatedly.

### Avoid Hidden Coupling

Message schemas, configuration files, and subsystem responsibilities should be documented and versioned.

### Prefer Conservative Defaults

Initial speed, angular limits, TTLs, and confidence thresholds should favor controlled testing over aggressive behavior.

### Build for Replacement

The detector, tracker, command transport, motor driver, and navigation layers should be replaceable without rewriting the entire system.
