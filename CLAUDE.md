# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**rosetta_ws** is a devcontainer workspace for **Rosetta** — a ROS2-to-LeRobot bridge that enables training robot policies from human demonstration. The workflow is: define a contract mapping ROS2 topics to LeRobot features, record demo episodes as ROS2 bags, convert to LeRobot datasets, train a policy, and deploy it back to the robot.

## Key Commands

### Setup & Build

```bash
# First-time setup (rosdep, pip installs, vcs import)
./scripts/setup.sh

# Build (default: dev mixin)
./scripts/build.sh

# Build with specific mixin(s)
./scripts/build.sh --mixin release
./scripts/build.sh --mixin debug test

# Build only a specific package
./scripts/build.sh -- --packages-select rosetta

# Source after build
source install/setup.bash
```

### Lint

```bash
ament_flake8 --linelength 100 src/action
ament_pep257 src/action
ament_xmllint src/action
```

### Test

```bash
./scripts/test.sh                    # All tests
./scripts/test.sh rosetta            # rosetta package only
./scripts/test.sh rosetta --verbose  # Verbose output
colcon test-result --all --verbose   # View results
```

### VS Code Tasks (Ctrl+Shift+P)

- **build** — default dev build
- **test / test package / test rosetta** — run tests
- **lint all / lint python** — flake8 + pep257
- **convert bags to lerobot** — run `port_bags.py`
- **train policy** — run `train_policy.sh` with prompts
- **resume training** — `lerobot-train --resume`
- **upload trained policy** — `huggingface-cli upload`

## Workspace Structure

```
rosetta_ws/
├── src/action/              # ROS2 packages (4 packages)
│   ├── rosetta/             # Core: nodes, contracts, bag conversion
│   ├── rosetta_interfaces/  # .action and .srv definitions
│   ├── lerobot_robot_rosetta/          # Robot bridge layer
│   ├── lerobot_teleoperator_rosetta/   # Teleoperation (experimental)
│   └── rosetta_rl/                # RL tools (coming soon)
├── libs/lerobot/            # LeRobot library (editable install)
├── models/                  # Trained policies by type (act/, pi05/, etc.)
├── datasets/
│   ├── bags/                # Raw rosbag2 (MCAP) recordings
│   └── lerobot/             # Converted LeRobot datasets
├── repos/                   # vcs import definitions (src.repos, libs.repos)
├── scripts/                 # build.sh, setup.sh, train_policy.sh, etc.
├── docker/                  # Dockerfiles (Dockerfile.spark, Dockerfile.x86, etc.)
├── .devcontainer/           # VS Code devcontainer (x86/, jetson/)
└── colcon/                  # Build mixins (mixin/build.mixin.yaml)
```

## Rosetta Package (`src/action/rosetta/`)

The core package with these key files:

| Category | Files |
|----------|-------|
| **Nodes** | `rosetta/rosetta_client_node.py`, `rosetta/episode_recorder_node.py`, `rosetta/rosetta_hil_manager_node.py`, `rosetta/episode_keyboard_node.py` |
| **Launch** | `launch/rosetta_client_launch.py`, `launch/episode_recorder_launch.py`, `launch/rosetta_hil_launch.py`, `launch/episode_keyboard_launch.py` |
| **Params** | `params/rosetta_client.yaml`, `params/episode_recorder.yaml`, `params/rosetta_hil_manager.yaml` |
| **Contracts** | `contracts/so_101.yaml`, `contracts/turtlebot3.yaml`, `contracts/so_101_hil.yaml` |
| **Conversion** | `rosetta/port_bags.py` — ROS2 bag → LeRobot dataset (supports sharding) |
| **Common** | `rosetta/common/` — contract parsing, encoders/decoders, converters, classifier server |

## Architecture

The system has 4 ROS2 packages:

1. **rosetta** — Orchestrates the recording→conversion→training→deployment pipeline. Nodes communicate via ROS2 actions/services.
2. **rosetta_interfaces** — Defines `RunPolicy.action`, `RecordEpisode.action`, `ManageEpisode.action` and `StartRecording.srv`, `StartHILEpisode.srv`.
3. **lerobot_robot_rosetta** — `rosetta.py` bridges LeRobot's inference to ROS2 robot control. Config in `config_rosetta.py`.
4. **lerobot_teleoperator_rosetta** — `rosetta_teleop.py` for remote teleoperation (experimental).

### Key flow
```
Contract (YAML) → EpisodeRecorder (records ROS2 bags)
                → RosettaClient (runs trained policy on robot)
                → RobotBridge (lerobot_robot_rosetta connects LeRobot inference to robot)
```

The **contract** (`contracts/*.yaml`) is central — it declares ROS2 topic mappings to LeRobot observation/action features (keys, types, selectors, image resize, etc.) and is used both during recording conversion and policy inference.

## LeRobot (`libs/lerobot/`)

Cloned LeRobot library installed in editable mode. Contains `COLCON_IGNORE` to prevent colcon from treating it as a ROS2 package. This is the bridge's ML backend — policy training, dataset handling, and inference infrastructure.

## Docker & Devcontainer

Builds from `docker/Dockerfile.spark` (and others). The devcontainer mounts HuggingFace/W&B/SSH credentials and enables GPU passthrough. Open in VS Code → "Reopen in Container".

## Colcon Mixins

Available in `colcon/mixin/build.mixin.yaml`. Common: `dev`, `debug`, `release`, `test`, `ci`, `prod`, `asan`, `tsan`.
