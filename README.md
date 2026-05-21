# SAI + Orbbec Setup on Ubuntu 24.04
## Posed RGB-D Data Collection for Task-Oriented Grasping

This documents the setup of the Spectacular AI (SAI) + Orbbec Femto Bolt pipeline for collecting posed RGB-D data, which feeds into SAM3D for clean point cloud generation and ultimately AnyGrasp for task-oriented grasp proposals.

---

## Pipeline Overview

```
Orbbec Femto Bolt (RGB + Depth + IMU)
        ↓
SAI (SLAM) → 6-DoF camera pose per frame
        ↓
Posed RGB-D dataset (data.mkv + data2.mkv + data.jsonl + calibration.json)
        ↓
SAM3D → clean per-object point clouds
        ↓
AnyGrasp → grasp proposals
        ↓
Part segmentation + VLM reasoning → task-oriented grasp
```

---

## System

- **OS:** Ubuntu 24.04 LTS
- **Camera:** Orbbec Femto Bolt
- **OrbbecSDK:** v1.10.11 (commit `81da0b3`) — newer versions have known issues
- **SAI Orbbec Plugin:** v1.34.0 — v1.36.0 causes SAI to freeze
- **ROS2:** Humble (via Mamba/Robostack)

---

## Part 1: Mamba + ROS2 Humble

### Install Miniforge

```bash
wget "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"
bash Miniforge3-$(uname)-$(uname -m).sh
# Say yes to conda init, then open a new terminal
```

### Create ROS2 Environment

```bash
mamba create -n ros_cg python=3.10
mamba activate ros_cg

conda config --env --add channels conda-forge
conda config --env --add channels robostack-staging
conda config --env --remove channels defaults

mamba install ros-humble-desktop
mamba install compilers cmake pkg-config make ninja colcon-common-extensions catkin_tools rosdep
```

### Verify

```bash
ros2 --help  # should show list of commands
```

---

## Part 2: OrbbecSDK

### Set Environment Variables

```bash
export REPO=~/lab_ws
mkdir -p $REPO
echo 'export REPO=~/lab_ws' >> ~/.bashrc
```

### Clone and Pin Version

```bash
cd $REPO
git clone https://github.com/orbbec/OrbbecSDK.git
cd OrbbecSDK
git checkout 81da0b392a0ccee031147c3eedbd3a6617ae43e2

export ORBBEC_SDK=$REPO/OrbbecSDK
echo 'export ORBBEC_SDK=$REPO/OrbbecSDK' >> ~/.bashrc
```

### Install udev Rules

```bash
cd $ORBBEC_SDK/misc/scripts
sudo chmod +x ./install_udev_rules.sh
sudo ./install_udev_rules.sh
sudo udevadm control --reload && sudo udevadm trigger
```

### Build (examples disabled due to gets() deprecation in GCC 14)

```bash
cd $ORBBEC_SDK
mkdir build && cd build
cmake .. -DBUILD_EXAMPLES=OFF
cmake --build . --config Release
```

---

## Part 3: SAI Orbbec Plugin

Download `spectacularAI_orbbecPlugin_cpp_non-commercial_1.34.0.tar.gz` from:
https://github.com/SpectacularAI/sdk/releases/tag/1.34.0

```bash
cd $REPO
tar -xzf ~/Downloads/spectacularAI_orbbecPlugin_cpp_non-commercial_1.34.0.tar.gz
mv Linux_Ubuntu_x86-64 sai-orbbec-plugin

export SAI_ORBBEC_PLUGIN=$REPO/sai-orbbec-plugin
echo 'export SAI_ORBBEC_PLUGIN=$REPO/sai-orbbec-plugin' >> ~/.bashrc
```

### Create SAI Python Environment

```bash
conda create -n sai python=3.8
conda activate sai

# IMPORTANT: unset PYTHONPATH before installing — ros_cg pollutes it
unset PYTHONPATH
pip install "spectacularAI[full]"

# Verify
python -c "import spectacularAI; print('ok')"
```

> **Note:** Always run `unset PYTHONPATH` when activating the `sai` environment.
> The `ros_cg` environment sets PYTHONPATH which causes pip and imports to break.

### Install System Dependencies

```bash
sudo apt install ffmpeg make cmake build-essential libgl-dev libglu1-mesa-dev
```

### Test SAI Recording

Plug in the Orbbec camera, then:

```bash
conda activate sai
unset PYTHONPATH
cd $SAI_ORBBEC_PLUGIN/bin/
./sai-record-orbbec
```

Move the camera around slowly. Press `Ctrl+C` to stop. Output saved to `data/<timestamp>/`:
- `data.mkv` — RGB video
- `data2.mkv` — depth video
- `data.jsonl` — camera poses (JSONL format)
- `calibration.json` — camera intrinsics/extrinsics

### Test SAI as Library

```bash
export MY_INSTALL_PREFIX=~/.local
cd $SAI_ORBBEC_PLUGIN
make PREFIX=$MY_INSTALL_PREFIX install
make PREFIX=$MY_INSTALL_PREFIX ORBBEC_DIR=$ORBBEC_SDK example

# Run with camera plugged in — move camera around
cd examples/target
./vio_jsonl
```

You should see a continuous stream of JSON pose objects with `"status":"TRACKING"`.

---

## Part 4: ROS2 Wrapper (ros2_orbbec_slam)

### Add Library Paths

```bash
echo 'export LD_LIBRARY_PATH=$SAI_ORBBEC_PLUGIN/lib:$LD_LIBRARY_PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=$ORBBEC_SDK/lib/linux_x64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

### Set Up Workspace and Clone

```bash
mkdir -p $REPO/ros2_ws/src
export ROS2_WS=$REPO/ros2_ws
echo 'export ROS2_WS=$REPO/ros2_ws' >> ~/.bashrc

conda activate ros_cg
cd $ROS2_WS/src
git clone git@github.com:siqizhou/ros2_orbbec_slam.git
```

### Build

```bash
cd $ROS2_WS
colcon build
source $ROS2_WS/install/setup.bash
```

### Run

```bash
# Terminal 1 — start publisher (camera must be plugged in)
conda activate ros_cg
source $ROS2_WS/install/setup.bash
ros2 run sai_orbbec sai_publisher

# Terminal 2 — verify pose frequency (~30 Hz expected)
conda activate ros_cg
source $ROS2_WS/install/setup.bash
ros2 topic hz spectacular_ai/pose
```

---

## Recording a Dataset

```bash
conda activate sai
unset PYTHONPATH
cd $SAI_ORBBEC_PLUGIN/bin/
./sai-record-orbbec
```

**Scanning tips:**
- Place objects on a table (mugs, bottles, tools — things relevant to grasping)
- Move the camera **slowly** — fast movement causes tracking loss
- Scan from multiple angles: left, right, above, closer, further
- 30–60 seconds of footage is sufficient
- `TRACKING_LOST` in output should only appear briefly if at all

---

## Known Issues & Fixes

| Issue | Fix |
|-------|-----|
| `pip` uses wrong environment | `unset PYTHONPATH` before using pip in `sai` env |
| OrbbecSDK build fails (`gets()` error) | Build with `-DBUILD_EXAMPLES=OFF` |
| SAI visualizer `FileNotFoundError` | Harmless — recording still works |
| `colcon` not found | Make sure `ros_cg` env is active |
| `GL/gl.h` not found | `sudo apt install libgl-dev libglu1-mesa-dev` |
| `make`/`cmake` not found | `sudo apt install make cmake build-essential` |

---

## Next Steps

- Set up SAM3D to consume posed RGB-D recordings and produce clean per-object point clouds
- Run AnyGrasp on SAM3D point clouds for grasp proposals
- Part segmentation with DINO/SigLIP/CLIP
- VLM reasoning for task-oriented grasp selection
