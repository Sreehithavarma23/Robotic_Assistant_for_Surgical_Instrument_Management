# Surgical Instrument Delivery Robot

> A ROS 2 and Gazebo simulation of a mobile manipulator that prepares surgical
> instrument trays and transports them between hospital rooms.

This project brings together a differential-drive robot, a six-joint arm,
navigation, manipulation, perception experiments, and ROS 2 task orchestration.
It is a robotics software project and simulation demo—not a medical device.

## What it demonstrates

- **Mobile manipulation:** coordinate a simulated mobile base and six-degree-of-freedom arm.
- **Autonomous navigation:** use Nav2 with a hospital map, AMCL localization,
  lidar-based obstacle sensing, and configured motion planning.
- **Task orchestration:** accept a surgery name, select its tray, collect it,
  deliver it to the operation room, and return it to sterilization after the
  completion command.
- **Simulated grasping:** use a Gazebo link-attacher service to attach and detach
  trays during the mission.
- **Perception experiments:** detect ArUco markers, publish marker poses, map
  overhead-camera observations to tray poses, and generate vision-based pickup
  goals.

## System overview

```text
                         /surgery_request
                                |
                                v
                    +-----------------------+
                    |   Surgery Orchestrator |
                    +-----------+-----------+
                                |
                   +------------+-------------+
                   |                          |
                   v                          v
            Nav2 / hospital map        Arm trajectory action
                   |                          |
                   v                          v
             Mobile base                Six-joint arm
                   |                          |
                   +------------+-------------+
                                |
                       Gazebo link attacher
                                |
                     Instrument tray delivery
```

The source also contains an ArUco perception and vision-goal pipeline. It is
kept separate from the main orchestrator path: **the orchestrator currently
uses the pickup pose from `instrument_db.yaml`, not the live tray-pose topics**.
The experimental vision navigation follower and the orchestrator both respond
to surgery requests and should not be run together unless their navigation
behavior is coordinated.

## Repository layout

| Package | Responsibility |
| --- | --- |
| `surgical_description` | Robot Xacro/URDF, Gazebo worlds and models, simulation launch files, controller configuration, and Nav2 parameters. |
| `surgical_navigation` | Hospital map, room locations, surgery-to-tray/instrument database, and navigation-related nodes. |
| `surgical_task_manager` | Mission orchestration, arm/base task nodes, ArUco detection and localization, and vision-goal helpers. |
| `instrument_moveit_config` | MoveIt configuration, planning parameters, RViz setup, and demo launch. |
| `linkattacher_msgs` | ROS 2 service definitions used to attach and detach simulated models. |
| `ros2_linkattacher` | Gazebo link-attacher plugin used by simulated tray pickup and drop-off. |
| `pymoveit2` | Python MoveIt 2 interface used by the target-pose motion helper. |
| `surgical_interfaces` | Interface package scaffold. |

The workspace also includes map/world assets, frame-graph exports, and a
recorded ROS 2 bag. Build products under `build/`, `install/`, and `log/` are
generated locally and are not needed as source.

## Software requirements

- ROS 2 **Humble-compatible** environment
- Gazebo Classic and the ROS 2 Gazebo integration (`gazebo_ros`)
- Nav2 and its Simple Commander Python API
- MoveIt 2 and `moveit_configs_utils`
- `ros2_control`, joint trajectory controllers, Xacro, RViz 2, and the ROS 2
  Python packages declared by the workspace
- OpenCV with the ArUco module for the marker-detection nodes

The project is configured for Gazebo Classic APIs. Compatibility with other ROS
2 distributions or newer Gazebo releases has not been established here.

## Build the workspace

Run these commands from the workspace root in a terminal with ROS 2 installed:

```bash
source /opt/ros/humble/setup.bash
rosdep update
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
source install/setup.bash
```

If the workspace has already been built, source its overlay in every new
terminal before using `ros2 launch` or `ros2 run`:

```bash
source /opt/ros/humble/setup.bash
source ~/instrument_robot_ws/install/setup.bash
```

Use the actual path where you cloned this workspace if it is not
`~/instrument_robot_ws`.

## Run the simulation

Start each command in its own terminal. Source both the ROS 2 underlay and
workspace overlay in each terminal.

### 1. Start Gazebo and the robot

```bash
ros2 launch surgical_description gazebo.launch.py
```

The launch file defaults to `surgical_world.world`, spawns the robot, enables
the camera, and starts the joint-state broadcaster and arm trajectory
controller. Common launch arguments include:

```bash
ros2 launch surgical_description gazebo.launch.py \
  world:=/path/to/another.world \
  use_robot_camera:=true \
  camera_pitch:=1.11 \
  spawn_x:=-5.0 spawn_y:=-8.0 spawn_yaw:=0.0
```

### 2. Start Nav2

```bash
ros2 launch surgical_description nav2_launch.py
```

This loads the configured hospital map and Nav2 parameters with simulated time.

### 3. Start the task orchestrator

```bash
ros2 run surgical_task_manager surgery_orchestrator
```

The node waits for Nav2 and listens on `/surgery_request`. The current database
defines these surgery names:

| Request | Tray | Example instruments |
| --- | --- | --- |
| `appendectomy` | `instrument_tray_1` | Scalpel, forceps, scissors |
| `orthopedic` | `instrument_tray_2` | Bone saw, drill, forceps |
| `cardiac` | `instrument_tray_3` | Clamp, retractor, needle holder |

Submit a request from another sourced terminal:

```bash
ros2 topic pub --once /surgery_request std_msgs/msg/String \
  "{data: 'appendectomy'}"
```

After the tray has been delivered and the simulated surgery is complete, request
its return to the sterilization room:

```bash
ros2 topic pub --once /surgery_request std_msgs/msg/String \
  "{data: 'finished'}"
```

The orchestrator also accepts common completion variants such as `done`,
`complete`, and `surgery finished`. It remembers the last delivered tray in
`/tmp/surgical_task_manager_last_tray.txt` so a completion request can identify
which tray to return.

### Optional: visualization and MoveIt demo

For a robot-description and RViz view without the Gazebo task flow:

```bash
ros2 launch surgical_description display.launch.py
```

To launch the MoveIt planning demo:

```bash
ros2 launch instrument_moveit_config demo.launch.py
```

## Perception and alternate vision-goal nodes

The task-manager package provides these console executables:

| Executable | Purpose |
| --- | --- |
| `aruco_detector` | Detect ArUco markers in the configured camera image stream and publish marker poses. |
| `aruco_tray_localizer` | Convert marker pose topics into `/tray_1_pose`, `/tray_2_pose`, and `/tray_3_pose`. |
| `vision_pickup_goal` | Convert tray poses into pickup goals and select a tray goal for a surgery request. |
| `vision_nav_goal_follower` | Navigate to the selected vision-based pickup goal. |
| `aruco_marker_follower` | Follow a detected target pose using base velocity commands. |
| `move_to_target` | Exercise MoveIt motion to a supplied pose or a detected ArUco target. |
| `fixed_pick_tray`, `pick_tray` | Standalone tray-pickup experiments. |

For example, the detector can be started with:

```bash
ros2 run surgical_task_manager aruco_detector
```

The default image and camera-info topics are `/robot_camera_color/image_raw`
and `/robot_camera_color/camera_info`. The tray localizer currently assumes a
particular overhead camera origin and marker-to-tray mapping (marker 2 to tray
1, marker 1 to tray 2, and marker 3 to tray 3). Treat those assumptions as
simulation configuration, not general-purpose calibration.

## Useful interfaces

| Interface | Type | Role |
| --- | --- | --- |
| `/surgery_request` | `std_msgs/msg/String` | Start a named delivery task or issue a completion command. |
| `/cmd_vel` | `geometry_msgs/msg/Twist` | Mobile-base velocity commands used by task helpers. |
| `/arm_position_controller/follow_joint_trajectory` | `control_msgs/action/FollowJointTrajectory` | Six-joint arm trajectories. |
| `/ATTACHLINK`, `/DETACHLINK` | Gazebo link-attacher services | Attach or release a tray in simulation. |
| `/tray_{id}_pose` | `geometry_msgs/msg/PoseStamped` | Tray pose outputs from the ArUco localization helper. |

Inspect the runtime graph with standard ROS 2 tools, for example:

```bash
ros2 node list
ros2 topic list
ros2 topic echo /tray_1_pose
```

## Configuration

- `src/surgical_navigation/config/instrument_db.yaml` maps surgery names to
  tray models, pickup poses, arm joint waypoints, and instrument lists.
- `src/surgical_navigation/config/locations.yaml` stores named room coordinates.
- `src/surgical_navigation/maps/` contains the hospital occupancy map.
- `src/surgical_description/config/nav2_params.yaml` configures Nav2, including
  localization, planning, control, and costmaps.
- `src/surgical_description/config/arm_controllers.yaml` defines the six arm
  joints and trajectory-controller settings.
- `src/instrument_moveit_config/config/` contains MoveIt planning and controller
  configuration.

Room coordinates, pickup poses, camera transforms, tray models, and joint
waypoints are simulation-specific and may need calibration when the world or
robot model changes.

## Tests and checks

The Python ROS packages include `ament` lint tests. Run them after building:

```bash
colcon test
colcon test-result --verbose
```

The repository does not currently provide a complete automated end-to-end
simulation test, and the commands above do not establish clinical or
hardware-level safety.

## Current scope and limitations

- Simulation-only; no real surgical instrument, patient, or clinical workflow
  integration is provided.
- The main orchestrator uses configured pickup coordinates rather than
  consuming the live ArUco tray-pose outputs.
- Navigation and manipulation coordinates and trajectories are tuned to the
  included simulated hospital world.
- The legacy `surgical_navigation` `navigator` entry point is deprecated; use
  `surgical_task_manager surgery_orchestrator` for the active mission flow.
- The root workspace does not declare one unified license. Check individual
  packages and their `package.xml` files for their license metadata.

## License

Licensing is package-specific. Review each package's metadata and bundled
license files before redistributing the complete workspace.
