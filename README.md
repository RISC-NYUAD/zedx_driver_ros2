# zedx_driver_ros2

ROS 2 bringup and fused body tracking for ZED X cameras. The cameras connect to a ZED Box while primary compute runs on a networked workstation with an RTX GPU.

## Hardware Setup

The system is setup and connected as shown below.

<p align="center">
  <img src="docs/hardware_setup.png" alt="setup" />
</p>

## Installation

Run the following steps on both the ZED Box and the networked workstation:

1. Clone the repository:
   ```bash
   git clone --recursive https://github.com/Abanesjo/zedx_driver_ros2
   ```

2. Configure Zenoh (`src/RMW_ZENOH_ROUTER_CONFIG.json5`):
   - **ZED Box:** leave as is.
   - **Workstation:** add the ZED Box router IP to `connect.endpoints`, e.g. `endpoints: ["tcp/192.168.0.133:7447"]`.

3. In `docker/Dockerfile`, set `FROM` and `ROS_DISTRO` to match the machine:

   | Machine | `FROM` | `ROS_DISTRO` |
   |---|---|---|
   | Workstation with RTX GPU | `stereolabs/zed:5.3-gl-devel-cuda12.8-ubuntu24.04` | `jazzy` |
   | JetPack 6 Jetson / ZED Box | `stereolabs/zed:5.3.0-py-devel-l4t-r36.4` | `humble` |

   All machines exchanging ROS topics must use the same ROS distro (`rmw_zenoh` does not bridge Humble and Jazzy). The camera stream between `server.launch.xml` and `body_tracking.launch.xml` is not a ROS topic and is unaffected.

4. Build and start the container:
   ```bash
   cd docker
   docker compose up
   ```

## Usage

All commands below are run inside the container (`docker exec -it zedx_driver_ros2 bash`).

**On the ZED Box:**

1. Start the Zenoh router and leave it running:
   ```bash
   ros2 run rmw_zenoh_cpp rmw_zenohd
   ```

2. Start the ZED stream server:
   ```bash
   ros2 launch zed_launcher server.launch.xml
   ```

**On the Workstation:**

1. Start the Zenoh router and leave it running:
   ```bash
   ros2 run rmw_zenoh_cpp rmw_zenohd
   ```

2. Start body tracking (set `debug:=false` to improve performance):
   ```bash
   ros2 launch zed_launcher body_tracking.launch.xml debug:=true rviz:=true
   ```
   *(Optional: this can also be run directly on the ZED Box.)*

3. Start human angle calculation and collision capsules:
   ```bash
   ros2 launch human_mapping human_mapping.launch.xml
   ```

![rviz](docs/rviz.png)

## Calibration

To calibrate the ZED X cameras, update `src/zed_launcher/calibration/zedx/calibration.json` with the output from [ZED360](https://www.stereolabs.com/docs/fusion/zed360) and rebuild `zed_launcher`.

## ZED Mini

USB ZED Mini cameras plugged directly into a single computer are also supported without a stream server. Start the Zenoh router as above, then run:

```bash
ros2 launch zed_launcher zed_mini.launch.xml debug:=true rviz:=true
```

- By default this uses `BODY_18` (colliders only in `human_mapping`); pass `body_format:=BODY_38` to compute joint angles.
- Override resolution and frame rate with `camera_resolution:=` and `camera_fps:=` (valid combinations are listed in `src/zed_launcher/launch/zed_mini.launch.xml`).
- For different ZED Minis, pass `camera_serials:="[<serial1>, <serial2>]"` (shown by `ZED_Explorer -a`) and update `src/zed_launcher/calibration/zedm/calibration.json`.
