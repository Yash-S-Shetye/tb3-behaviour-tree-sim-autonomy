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
