# tb3_autonomy

A ROS2 Humble package that drives a TurtleBot3 through a sequence of predefined
locations using a [BehaviorTree.CPP v3](https://www.behaviortree.dev/) tree,
Nav2 for navigation.

## Overview

The package has two main runtime components:

- **`autonomy_node`** — loads a behavior tree from XML, ticks it on a timer,
  and executes `GoToPose` actions that send goals to Nav2's
  `/navigate_to_pose` action server. Target locations are read from a YAML
  config file at runtime.

## Package layout

```
tb3_autonomy/
├── bt_xml/                     # Behavior tree XML definitions (tree.xml)
├── config/                     # Location YAML files (e.g. sim_house_locations.yaml)
├── launch/                     # Launch files (autonomy.launch.py)
├── include/tb3_autonomy/
│   ├── navigation_behaviors.h  # GoToPose BT node declaration
├── src/
│   ├── autonomy_node.cpp       # BT factory setup, tree ticking, main()
│   ├── navigation_behaviors.cpp# GoToPose BT node implementation
├── CMakeLists.txt
└── package.xml
```

## Dependencies

- ROS2 Humble
- `behaviortree_cpp_v3` (`sudo apt install ros-humble-behaviortree-cpp-v3`)
- `nav2_msgs`, `rclcpp_action` (part of a standard Nav2 install)
- `tf2`, `tf2_geometry_msgs`, `tf2_ros`
- `yaml-cpp` (`sudo apt install libyaml-cpp-dev`)
- `Eigen3` (`sudo apt install libeigen3-dev`)

> **Note:** This package targets BehaviorTree.CPP **v3**, which is the
> version Nav2 on Humble is built against. Do not install BT.CPP v4
> alongside it — mixing versions in the same environment can cause ABI
> mismatches between headers and linked libraries.

## Building

```bash
cd ~/turtlebot3_ws
colcon build --packages-select tb3_autonomy
source install/setup.bash
```

## Running

Bring up TurtleBot3 simulation and Nav2 first (map server, AMCL, Nav2
stack), then launch autonomy:

```bash
ros2 launch tb3_autonomy autonomy.launch.py
```

This starts `autonomy_node`, which loads `bt_xml/tree.xml` and ticks it
every 500ms. The tree sequentially sends `GoToPose` goals for each location
defined in the `location_file` parameter (see `launch/autonomy.launch.py`
for the default path).

## Configuring locations

Locations are defined in a YAML file as `[x, y, theta]`:

```yaml
# config/sim_house_locations.yaml
location1: [-1.0, -0.5, -2.356]
location2: [-1.0, 2.0, 0.785]
location3: [0.0, 0.5, 1.571]
location4: [0.0, -2.0, -1.571]
```

Reference these keys from the `loc` port in `bt_xml/tree.xml`:

```xml
<GoToPose name="go_to_location1" loc="location1" />
```

## Behavior tree nodes

### `GoToPose`

A `StatefulActionNode` that:
1. Reads the `loc` input port and looks up the corresponding pose in the
   YAML location file.
2. Sends a `NavigateToPose` goal to Nav2.
3. Tracks feedback (`distance_remaining`) and cancels the goal if no
   feedback is received for longer than `FEEDBACK_TIMEOUT_SEC` (stuck
   detection).
4. Returns `SUCCESS`/`FAILURE` based on the action result.

## Video Demonstration for autonomy node

[![Watch the demo](https://img.youtube.com/vi/-IHkGq6dSyA/hqdefault.jpg)](https://www.youtube.com/watch?v=-IHkGq6dSyA)

## Web dashboard
`web_dashboard.html` is a standalone browser page that lets you trigger Nav2
goals to the predefined locations without running `autonomy_node` or the
behavior tree — useful for ad-hoc testing and demos. It talks to ROS2
directly over a WebSocket using the
[rosbridge protocol](https://github.com/RobotWebTools/rosbridge_suite/blob/ros2/ROSBRIDGE_PROTOCOL.md),
with no external JS library dependency.
 
### Setup
 
1. Install and launch rosbridge on the machine running Nav2:
```bash
   sudo apt install ros-humble-rosbridge-suite
   ros2 launch rosbridge_server rosbridge_websocket_launch.xml
```
2. Open `web_dashboard.html` in a browser.
3. Set the WebSocket URL (default `ws://localhost:9090`; use the robot's LAN
   IP if connecting remotely, e.g. `ws://192.168.x.x:9090`).
4. Click **Connect**, then click a location button to send that goal. Live
   `distance_remaining` feedback streams into the log panel, and **Cancel
   Current Goal** aborts the in-flight goal.
### ⚠️ Security warning
 
`rosbridge_server` has **no built-in authentication**. Anyone who can reach
its WebSocket port can fully command the robot (navigation goals, and
potentially any other topic/service/action exposed on the graph). Only run
this on a trusted local network. **Do not expose port 9090 to the public
internet** without putting an authenticating reverse proxy in front of it.
 
### ⚠️ roslibjs `ROSLIB.Action` does not reliably work for ROS2 actions
 
If you're extending this dashboard or writing your own: **avoid
`ROSLIB.ActionClient`/`ROSLIB.Goal`** (roslibjs's original API) — these
implement the ROS1 actionlib wire format (separate `/goal`, `/feedback`,
`/result` topics with message types like `NavigateToPoseGoal`), which does
not exist in ROS2 and will fail with errors like:
 
```
Unable to import msg class NavigateToPoseGoal from package nav2_msgs
```
 
roslibjs also added a newer `ROSLIB.Action` class intended for ROS2, but as
of testing (Sept 2026) the npm-published build resolved by common CDNs
(`cdnjs`, `jsdelivr @latest`) does **not** include it — `new ROSLIB.Action(...)`
throws `TypeError: ROSLIB.Action is not a constructor` even though the
class exists in the library's source. Since this dashboard doesn't need any
other roslibjs feature, it works around the gap by speaking the rosbridge
v2.1.0 protocol directly over a plain `WebSocket`:
 
- Send a goal: `{ op: "send_action_goal", id, action, action_type, args, feedback: true }`
- Cancel a goal: `{ op: "cancel_action_goal", id, action }`
- Listen for `action_feedback` / `action_result` messages matching your `id`
No `advertise_action` call is needed when calling an *existing* action
server (like Nav2's) — that op is only for a client acting as its own
action server, the same relationship `call_service` has with
`advertise_service`.
 
If a future roslibjs release fixes this, `ROSLIB.Action` would be a cleaner
approach — worth re-testing periodically.

## Video Demonstration for web dashboard
[![Watch the demo](https://img.youtube.com/vi/VhdxiyQQRhU/hqdefault.jpg)](https://www.youtube.com/watch?v=VhdxiyQQRhU)
