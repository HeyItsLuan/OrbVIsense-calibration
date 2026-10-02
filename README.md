# OrbViSense — Camera-IMU Calibration Guide

Procedure for obtaining the calibration parameters of a monocular camera and an IMU using **ROS 1 Noetic**, **allan_variance_ros**, and **Kalibr**, and subsequently transferring them to the mono-inertial configuration of **ORB-SLAM3**.

---

## 0. Download the repository

The calibration procedure and conversion script are available in the following repository:

**GitHub:** https://github.com/HeyItsLuan/OrbVIsense-calibration

To clone the repository directly into the Desktop:

```bash
cd "$HOME/Escritorio"

git clone https://github.com/HeyItsLuan/OrbVIsense-calibration
```

Then enter the repository:

```bash
cd "$HOME/Escritorio/OrbVIsense-calibration"
```

The repository contains:

```text
OrbVIsense-calibration/
├── README.md
├── dataset_to_rosbag.py
└── .gitignore
```

The `README.md` contains this complete calibration guide, while `dataset_to_rosbag.py` contains the script used to convert the datasets into ROS bags.

The calibration datasets are kept outside the repository, directly on the Desktop:

```text
$HOME/Escritorio/
├── OrbVIsense-calibration/
├── dataset_CAMIMU/
└── dataset_IMU/
```

---

# 1. Calibration workflow

The calibration is performed using **two independent recordings**:

```text
                            ┌──────────────────────┐
                            │   dataset_IMU        │
                            │                      │
┌──────────────────────┐    │Phone completely      │
│ dataset_CAMIMU       │    │stationary            │
│                      │    └──────────┬───────────┘
│ Phone moving         │               │
│ in front of AprilGrid│               ▼
└──────────┬───────────┘          Allan variance
           │                           │
┌──────────────────────┐               ▼
│    aprilgrid.yaml    │            imu.yaml
│                      │               │
│ Physical description │               │
│ of the AprilGrid     │               ▼
└──────────┬───────────┘               │
           └────────────┬──────────────┘
                        ▼
                     Kalibr
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
      Camera calibration    Camera-IMU calibration
             │                     │
             └──────────┬──────────┘
                        ▼
             Parameters for ORB-SLAM3
                        │
                        ▼
                    EuRoC.yaml
```

The final result of this guide is the set of parameters required to build the mono-inertial configuration file used by ORB-SLAM3.

---

# 2. Requirements

## System

* Ubuntu 20.04
* ROS 1 Noetic
* Python 3
* `catkin`
* `catkin_tools`

## Tools

* `allan_variance_ros`
* Kalibr
* Git

## Hardware

* Phone or camera with IMU
* AprilGrid

---

# 3. Dataset preparation

The datasets should initially have the following structure:

```text
dataset_#######/
├── cam0/
│   ├── data/
│   └── data.csv
└── imu0/
    └── data.csv
```

The file:

```text
imu0/data.csv
```

must contain:

```text
timestamp,wx,wy,wz,ax,ay,az
```

with timestamps expressed in **nanoseconds**.

The images in:

```text
cam0/data/
```

must be named using their timestamps.

Two datasets are required.

---

## 3.1. Dataset for Allan variance

Name:

```text
$HOME/Escritorio/dataset_IMU/
```

Initial structure:

```text
dataset_IMU/
├── cam0/
│   ├── data/
│   └── data.csv
└── imu0/
    └── data.csv
```

During this recording:

> The phone must remain completely stationary for approximately 30 minutes or longer.

This dataset is used exclusively to characterize the IMU noise using **Allan variance**.

The camera is not involved in the Allan variance calculation, although it may be part of the original dataset structure.

---

## 3.2. Dataset for camera-IMU calibration

Name:

```text
$HOME/Escritorio/dataset_CAMIMU/
```

Initial structure:

```text
dataset_CAMIMU/
├── cam0/
│   ├── data/
│   └── data.csv
└── imu0/
    └── data.csv
```

During this recording:

> The phone is moved in front of the AprilGrid while observing it from different positions and orientations.

This dataset is used for:

* camera intrinsic calibration;
* camera-IMU calibration;
* estimation of the transformation between the camera and IMU;
* estimation of the time offset between both sensors.

---

# 4. AprilGrid description

The AprilGrid used during calibration must be described using a YAML file.

The board used in this procedure has:

```text
4 columns
5 rows
```

and:

```text
tagSize = 0.0439 m
```

Create the file inside the calibration dataset:

```bash
gedit "$HOME/Escritorio/dataset_CAMIMU/aprilgrid.yaml"
```

Contents:

```yaml
target_type: 'aprilgrid'
tagCols: 4
tagRows: 5
tagSize: 0.0439
tagSpacing: 0.1
```

### Parameter meaning

| Parameter     | Meaning                         |
| ------------- | ------------------------------- |
| `target_type` | Type of target used by Kalibr   |
| `tagCols`     | Number of tags in X / columns   |
| `tagRows`     | Number of tags in Y / rows      |
| `tagSize`     | AprilTag side length, in meters |
| `tagSpacing`  | Relative spacing between tags   |

`tagSpacing` is a **ratio relative to `tagSize`**, not a distance expressed in meters.

This file only describes the physical geometry of the AprilGrid. It does not contain camera or IMU parameters.

---

# 5. Converting the dataset to a ROS bag

The calibration repository contains the script:

```text
dataset_to_rosbag.py
```

The script receives the dataset directory as its only argument.

From the repository directory:

```bash
cd "$HOME/Escritorio/orbvisense-calibration"
python3 ./dataset_to_rosbag.py \
    "$HOME/Escritorio/dataset_CAMIMU"
```

This automatically generates:

```text
$HOME/Escritorio/dataset_CAMIMU/camera_imu.bag
```

For the stationary dataset:

```bash
cd "$HOME/Escritorio/orbvisense-calibration"
python3 ./dataset_to_rosbag.py \
    "$HOME/Escritorio/dataset_IMU"
```

This generates:

```text
$HOME/Escritorio/dataset_IMU/imu.bag
```

The bag type is selected automatically according to the dataset name:

```text
dataset_CAMIMU → camera_imu.bag
dataset_IMU    → imu.bag
```

---

# 6. Allan variance

## 6.1. Install `allan_variance_ros`

Enter the ROS workspace:

```bash
cd "$HOME/ros1_ws/src"
```

Clone the package:

```bash
git clone https://github.com/ori-drs/allan_variance_ros.git
```

Build it:

```bash
cd "$HOME/ros1_ws"

catkin_make -j1
```

Source the workspace:

```bash
source /opt/ros/noetic/setup.bash
source "$HOME/ros1_ws/devel/setup.bash"
```

Verify that ROS can locate the package:

```bash
roscd allan_variance_ros
```

Check its contents:

```bash
ls
```

---

# 7. Inspecting the IMU bag

The generated file is located at:

```text
$HOME/Escritorio/dataset_IMU/imu.bag
```

Run:

```bash
rosbag info "$HOME/Escritorio/dataset_IMU/imu.bag"
```

From this output, verify:

1. recording duration;
2. number of messages;
3. IMU topic;
4. approximate IMU frequency.

The topic used by this procedure is:

```text
/imu0
```

---

## 7.1. Measuring the IMU frequency

Use three terminals.

### Terminal 1

```bash
roscore
```

### Terminal 2

```bash
source /opt/ros/noetic/setup.bash
source "$HOME/ros1_ws/devel/setup.bash"

rosbag play "$HOME/Escritorio/dataset_IMU/imu.bag" --pause
```

The bag will start paused.

Press:

```text
Space
```

to start playback.

### Terminal 3

```bash
rostopic hz /imu0
```

Let the bag play for several seconds until the displayed frequency stabilizes.

The measured frequency will be used as `imu_rate` and `measure_rate`.

---

# 8. Allan variance configuration

Open:

```bash
gedit "$HOME/ros1_ws/src/allan_variance_ros/config/imu_phone.yaml"
```

The structure should be:

```yaml
imu_topic: "/imu0"
imu_rate: <IMU_FREQUENCY>
measure_rate: <IMU_FREQUENCY>
sequence_time: <DURATION_IN_SECONDS>
```

For example:

```yaml
imu_topic: "/imu0"
imu_rate: 232
measure_rate: 232
sequence_time: 3407
```

The `imu_rate` and `measure_rate` values correspond to the frequency measured with:

```bash
rostopic hz /imu0
```

`sequence_time` corresponds to the actual recording duration obtained using:

```bash
rosbag info "$HOME/Escritorio/dataset_IMU/imu.bag"
```

---

# 9. Preparing the bag for Allan variance

Run:

```bash
python3 "$HOME/ros1_ws/src/allan_variance_ros/scripts/cookbag.py" \
    --input "$HOME/Escritorio/dataset_IMU/imu.bag" \
    --output "$HOME/Escritorio/dataset_IMU/imu_cooked.bag"
```

Check the result:

```bash
rosbag info "$HOME/Escritorio/dataset_IMU/imu_cooked.bag"
```

Verify that:

```text
/imu0
```

is still present and that the IMU samples have been preserved.

---

# 10. Computing Allan variance

Source ROS:

```bash
source /opt/ros/noetic/setup.bash
source "$HOME/ros1_ws/devel/setup.bash"
```

Run:

```bash
rosrun allan_variance_ros allan_variance \
    "$HOME/Escritorio/dataset_IMU" \
    "$HOME/ros1_ws/src/allan_variance_ros/config/imu_phone.yaml"
```

The process generates:

```text
$HOME/Escritorio/dataset_IMU/allan_variance.csv
```

---

# 11. Generating `imu.yaml`

Run:

```bash
source /opt/ros/noetic/setup.bash
source "$HOME/ros1_ws/devel/setup.bash"

rosrun allan_variance_ros analysis.py \
    --data "$HOME/Escritorio/dataset_IMU/allan_variance.csv" \
    --config "$HOME/ros1_ws/src/allan_variance_ros/config/imu_phone.yaml"
```

`analysis.py` generates:

```text
imu.yaml
```

as a relative output file.

Therefore, the file is generated at:

```text
$HOME/ros1_ws/src/allan_variance_ros/imu.yaml
```

Copy it to the dataset:

```bash
cp "$HOME/ros1_ws/src/allan_variance_ros/imu.yaml" \
   "$HOME/Escritorio/dataset_IMU/imu.yaml"
```

The resulting structure is:

```text
dataset_IMU/
├── cam0/
├── imu0/
├── imu.bag
├── imu_cooked.bag
├── allan_variance.csv
└── imu.yaml
```

The file:

```text
imu.yaml
```

contains the IMU noise parameters that will later be used by Kalibr.

---

# 12. Installing Kalibr

Create the workspace:

```bash
mkdir -p "$HOME/kalibr_workspace/src"
```

Enter `src`:

```bash
cd "$HOME/kalibr_workspace/src"
```

Clone Kalibr:

```bash
git clone https://github.com/ethz-asl/kalibr.git
```

Check the installation:

```bash
ls "$HOME/kalibr_workspace/src/kalibr"
```

---

## 12.1. Dependencies

Install the system dependencies:

```bash
sudo apt-get update

sudo apt-get install -y \
    git \
    wget \
    autoconf \
    automake \
    nano \
    libeigen3-dev \
    libboost-all-dev \
    libsuitesparse-dev \
    doxygen \
    libopencv-dev \
    libpoco-dev \
    libtbb-dev \
    libblas-dev \
    liblapack-dev \
    libv4l-dev
```

Install the Python dependencies:

```bash
sudo apt-get install -y \
    python3-dev \
    python3-pip \
    python3-scipy \
    python3-matplotlib \
    ipython3 \
    python3-wxgtk4.0 \
    python3-tk \
    python3-igraph \
    python3-pyx
```

---

# 13. Configuring the Kalibr workspace

Source ROS Noetic:

```bash
source /opt/ros/noetic/setup.bash
```

Enter the workspace:

```bash
cd "$HOME/kalibr_workspace"
```

Initialize `catkin_tools`:

```bash
catkin config --init
```

Configure the workspace to extend ROS Noetic:

```bash
catkin config --extend /opt/ros/noetic
```

Check the configuration:

```bash
catkin config
```

It should show:

```text
Workspace:  $HOME/kalibr_workspace
Source:     $HOME/kalibr_workspace/src
Extending:  /opt/ros/noetic
```

---

# 14. Building Kalibr

Build using a single CPU core:

```bash
cd "$HOME/kalibr_workspace"

catkin build -j1
```

The build may take a significant amount of time.

Using:

```text
-j1
```

reduces resource consumption and prevents the system from being overloaded during compilation.

Once the build finishes:

```bash
source "$HOME/kalibr_workspace/devel/setup.bash"
```

The main executables used in this procedure are located in:

```text
$HOME/kalibr_workspace/devel/.private/kalibr/lib/kalibr/
```

---

# 15. Camera intrinsic calibration

The first calibration performed by Kalibr is the camera intrinsic calibration.

Run:

```bash
"$HOME/kalibr_workspace/devel/.private/kalibr/lib/kalibr/kalibr_calibrate_cameras" \
    --models pinhole-radtan \
    --target "$HOME/Escritorio/dataset_CAMIMU/aprilgrid.yaml" \
    --bag "$HOME/Escritorio/dataset_CAMIMU/camera_imu.bag" \
    --topics /cam0/image_raw
```

The model:

```text
pinhole-radtan
```

corresponds to:

```text
Projection: pinhole
Distortion: radtan
```

If the calibration completes successfully, Kalibr generates files inside:

```text
$HOME/Escritorio/dataset_CAMIMU/
```

Including:

```text
camera_imu-camchain.yaml
camera_imu-results-cam.txt
camera_imu-report-cam.pdf
```

The main output of this stage is:

```text
camera_imu-camchain.yaml
```

---

# 16. Camera-IMU calibration

This stage uses:

* the camera + IMU bag;
* the camera calibration obtained previously;
* the `imu.yaml` generated using Allan variance;
* the `aprilgrid.yaml`.

Run:

```bash
"$HOME/kalibr_workspace/devel/.private/kalibr/lib/kalibr/kalibr_calibrate_imu_camera" \
    --bag "$HOME/Escritorio/dataset_CAMIMU/camera_imu.bag" \
    --cam "$HOME/Escritorio/dataset_CAMIMU/camera_imu-camchain.yaml" \
    --imu "$HOME/Escritorio/dataset_IMU/imu.yaml" \
    --target "$HOME/Escritorio/dataset_CAMIMU/aprilgrid.yaml"
```

The results are written inside:

```text
$HOME/Escritorio/dataset_CAMIMU/
```

The main files are:

```text
camera_imu-camchain-imucam.yaml
camera_imu-imu.yaml
camera_imu-results-imucam.txt
camera_imu-report-imucam.pdf
```

---

# 17. Interpreting the Kalibr results

## 17.1. `camera_imu-camchain.yaml`

Contains the camera intrinsic calibration.

It includes parameters such as:

```text
camera_model
intrinsics
distortion_coeffs
distortion_model
resolution
rostopic
```

---

## 17.2. `camera_imu-camchain-imucam.yaml`

Contains the result of the joint camera-IMU calibration.

In addition to the camera parameters, it contains information about the camera-IMU transformation and the time offset.

---

## 17.3. `camera_imu-results-imucam.txt`

Contains the detailed joint calibration report.

Among its results are:

```text
T_ci: (imu0 to cam0)
T_ic: (cam0 to imu0)
```

and:

```text
timeshift cam0 to imu0
```

This file is used to identify the transformation that must be transferred to the ORB-SLAM3 configuration.

---

## 17.4. `camera_imu-imu.yaml`

Contains the IMU parameters used during the Kalibr calibration.

The noise parameters originally come from the `imu.yaml` generated through Allan variance.

---

# 18. Transferring the results to `EuRoC.yaml`

Kalibr and Allan variance use different variable names from ORB-SLAM3.

Therefore, the calibration parameters must be transferred to the format used by ORB-SLAM3.

The following table provides the mapping:

| Variable in `EuRoC.yaml` | Source file                       | Source variable               |
| ------------------------ | --------------------------------- | ----------------------------- |
| `Camera.type`            | `camera_imu-camchain-imucam.yaml` | `camera_model`                |
| `Camera.width`           | `camera_imu-camchain-imucam.yaml` | `resolution[0]`               |
| `Camera.height`          | `camera_imu-camchain-imucam.yaml` | `resolution[1]`               |
| `Camera.fx`              | `camera_imu-camchain-imucam.yaml` | `intrinsics[0]`               |
| `Camera.fy`              | `camera_imu-camchain-imucam.yaml` | `intrinsics[1]`               |
| `Camera.cx`              | `camera_imu-camchain-imucam.yaml` | `intrinsics[2]`               |
| `Camera.cy`              | `camera_imu-camchain-imucam.yaml` | `intrinsics[3]`               |
| `Camera.k1`              | `camera_imu-camchain-imucam.yaml` | `distortion_coeffs[0]`        |
| `Camera.k2`              | `camera_imu-camchain-imucam.yaml` | `distortion_coeffs[1]`        |
| `Camera.p1`              | `camera_imu-camchain-imucam.yaml` | `distortion_coeffs[2]`        |
| `Camera.p2`              | `camera_imu-camchain-imucam.yaml` | `distortion_coeffs[3]`        |
| `Camera.fps`             | `camera_imu.bag`                  | `/cam0/image_raw` frequency   |
| `Camera.RGB`             | Camera configuration              | Image format                  |
| `IMU.Frequency`          | `dataset_IMU/imu.yaml`            | `update_rate`                 |
| `IMU.NoiseGyro`          | `dataset_IMU/imu.yaml`            | `gyroscope_noise_density`     |
| `IMU.NoiseAcc`           | `dataset_IMU/imu.yaml`            | `accelerometer_noise_density` |
| `IMU.GyroWalk`           | `dataset_IMU/imu.yaml`            | `gyroscope_random_walk`       |
| `IMU.AccWalk`            | `dataset_IMU/imu.yaml`            | `accelerometer_random_walk`   |
| `IMU.T_b_c1`             | `camera_imu-results-imucam.txt`   | `T_ic` — `cam0 to imu0`       |

---

# 19. Interpreting `IMU.T_b_c1`

In the ORB-SLAM3 fork used by OrbViSense, `IMU.T_b_c1` is loaded as `Tbc`.

The ORB-SLAM3 code internally maintains:

```text
mTbc
```

and computes:

```text
mTcb = mTbc.inverse()
```

Therefore, for this calibration workflow, the transformation used as the value of:

```text
IMU.T_b_c1
```

is:

```text
T_ic
```

obtained from Kalibr:

```text
T_ic: (cam0 to imu0)
```

This matrix must **not** be inverted again before being placed in `IMU.T_b_c1`.

---

# 20. Camera-IMU time offset

Kalibr also provides:

```text
timeshift cam0 to imu0
```

with the convention:

```text
t_imu = t_cam + shift
```

This parameter should be preserved as part of the calibration results.

Do not invent an additional variable in `EuRoC.yaml` if the ORB-SLAM3 version being used does not provide a specific parameter for this time offset.

---

# 21. Final dataset structure

After completing the calibration, `dataset_IMU` contains:

```text
dataset_IMU/
├── cam0/
├── imu0/
├── imu.bag
├── imu_cooked.bag
├── allan_variance.csv
└── imu.yaml
```

And `dataset_CAMIMU` contains:

```text
dataset_CAMIMU/
├── cam0/
├── imu0/
├── camera_imu.bag
├── aprilgrid.yaml
├── camera_imu-camchain.yaml
├── camera_imu-camchain-imucam.yaml
├── camera_imu-imu.yaml
├── camera_imu-report-cam.pdf
├── camera_imu-report-imucam.pdf
├── camera_imu-results-cam.txt
└── camera_imu-results-imucam.txt
```

---

# 22. Final result

After completing the procedure, the following results are available:

```text
Allan variance
      │
      └── imu.yaml
             │
             ▼
        IMU parameters

Camera calibration
      │
      └── camera_imu-camchain.yaml
             │
             ▼
        Camera parameters

Camera-IMU calibration
      │
      ├── T_ic
      ├── time shift
      └── joint calibration results
             │
             ▼
        Camera-IMU parameters
```

These results can be used to construct the **mono-inertial ORB-SLAM3 configuration file (`EuRoC.yaml`)** subsequently used by OrbViSense.

The `aprilgrid.yaml` file is used during Kalibr calibration and **is not part of `EuRoC.yaml`**.

Atlas persistence parameters such as:

```yaml
#System.SaveAtlasToFile: "FileName"
#System.LoadAtlasFromFile: "FileName"
```

belong to the configuration and operation of ORB-SLAM3/OrbViSense, not to the calibration procedure. They should therefore be documented in the OrbViSense documentation rather than in this calibration guide.
