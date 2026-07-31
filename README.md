# Large-Language-Model-Driven Costmap

This repository contains the runtime implementation of the natural-language policy interface and its Nav2 costmap plugins. Paper evaluation datasets, benchmark runners, and result-generation scripts are intentionally excluded from this public runtime release.

The interface converts an English operator command into a validated policy, publishes zone and object constraints to Nav2 costmap layers, and executes the requested waypoint mission. The GUI supports single-robot and namespaced multi-robot Nav2 systems.

Project page: https://anonymous10forpaper.github.io/Large-Language-Model-Driven-Costmap_Page/

## Packages

### `policy_bridge`

- PyQt5 operator interface
- Installed Ollama model discovery and selection
- Natural-language policy generation
- Policy schema and execution-constraint validation
- FIFO mission queue, replacement, stop, resume, and cancel controls
- Map YAML display with editable waypoints and policy zones
- Single-robot and namespaced multi-robot missions
- Optional camera-based person detection and object-aware costs

### `my_costmap_layers`

- `KeepoutCommandLayer` for forbidden zones
- `ZoneSoftCostLayer` for zone-specific costs
- `ObjectAvoidanceLayer` for dynamic object clearance
- Runtime zone geometry updates from the GUI

## Platform

The reference environment uses Ubuntu 22.04, ROS 2 Humble, Nav2, and TurtleBot3. An Ollama-compatible local generate API is expected at `http://localhost:11434/api/generate` by default.

## Install

Place this repository inside a ROS 2 workspace and install the ROS dependencies.

```bash
cd ~/turtlebot3_ws/src
git clone https://github.com/anonymous10forpaper/Large-Language-Model-Driven-Costmap.git
cd ~/turtlebot3_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install --packages-select my_costmap_layers policy_bridge
source install/setup.bash
```

Install PyTorch and Ultralytics when camera-based person detection is required.

```bash
python3 -m pip install torch ultralytics
```

Install and start Ollama, then pull at least one supported model.

```bash
ollama pull qwen3:8b
ollama serve
```

The GUI lists every model reported by the connected Ollama server. A different generate endpoint can be supplied as a ROS parameter.

## Nav2 Configuration

Add the three plugins to the global costmap. Keep the inflation layer after the policy layers so hard and soft costs participate in the final inflated costmap.

```yaml
global_costmap:
  global_costmap:
    ros__parameters:
      plugins:
        - static_layer
        - obstacle_layer
        - keepout_command_layer
        - zone_softcost_layer
        - object_avoidance_layer
        - inflation_layer

      keepout_command_layer:
        plugin: "my_costmap_layers::KeepoutCommandLayer"
        enabled: true
        forbidden_zones_topic: "/forbidden_zones_update"
        zone_geometry_topic: "/zone_geometry_update"

      zone_softcost_layer:
        plugin: "my_costmap_layers::ZoneSoftCostLayer"
        enabled: true
        zone_cost_overrides_topic: "/zone_cost_overrides"
        zone_geometry_topic: "/zone_geometry_update"

      object_avoidance_layer:
        plugin: "my_costmap_layers::ObjectAvoidanceLayer"
        enabled: true
        object_positions_topic: "/object_world_positions"
        object_avoidance_radius_topic: "/object_avoidance_radius"
        avoidance_radius: 1.5
        hold_after_clear_s: 0.6
        decay_ttl_s: 0.6
        decay_step_s: 0.1
```

Restart Nav2 after adding the plugins. The GUI reports how many zone layers are connected.

## Run

Start the robot simulation or hardware Nav2 stack, start Ollama, and launch the GUI.

```bash
source ~/turtlebot3_ws/install/setup.bash
ros2 run policy_bridge policy_bridge_gui
```

To use another Ollama generate endpoint:

```bash
ros2 run policy_bridge policy_bridge_gui --ros-args -p gui_llm_endpoint:=http://127.0.0.1:11434/api/generate
```

## Map Setup

Open a ROS map YAML from **Map / Setup**. Waypoints can be placed by map click or entered in map-frame coordinates. Policy zones can be drawn as rectangles or entered as map bounds. Saving creates a map-specific sidecar named `<map_name>_waypoints.yaml` beside the selected map YAML.

The saved labels are immediately available to command interpretation, validation, waypoint execution, and both zone costmap layers.

## Multi-Robot Setup

Each robot must expose a namespaced Nav2 action such as `/tb1/follow_waypoints`. The GUI discovers robot namespaces from active ROS graph endpoints and creates one mission panel per robot.

Use fleet-wide policy topics in every robot's global-costmap parameters when keepout and soft-cost policies must be shared:

```yaml
keepout_command_layer:
  plugin: "my_costmap_layers::KeepoutCommandLayer"
  enabled: true
  forbidden_zones_topic: "/fleet/forbidden_zones_update"
  zone_geometry_topic: "/zone_geometry_update"

zone_softcost_layer:
  plugin: "my_costmap_layers::ZoneSoftCostLayer"
  enabled: true
  zone_cost_overrides_topic: "/fleet/zone_cost_overrides"
  zone_geometry_topic: "/zone_geometry_update"

object_avoidance_layer:
  plugin: "my_costmap_layers::ObjectAvoidanceLayer"
  enabled: true
  object_positions_topic: "/fleet/object_world_positions"
  object_avoidance_radius_topic: "/fleet/object_avoidance_radius"
```

Route goals remain robot-specific while accepted zone policies are published consistently across the fleet.

## Runtime Files

```text
.
├── policy_bridge
│   ├── policy_bridge
│   │   ├── gui_theme.py
│   │   ├── policybridge.py
│   │   ├── policybridge_gui.py
│   │   ├── waypoint_map.py
│   │   └── waypoint_map_gui.py
│   ├── package.xml
│   ├── setup.cfg
│   └── setup.py
└── my_costmap_layers
    ├── include/my_costmap_layers
    ├── plugins/costmap_plugins.xml
    ├── src
    ├── CMakeLists.txt
    └── package.xml
```

## License

This runtime code accompanies an anonymous manuscript submission and is provided for academic review and reproducibility. All rights are reserved by the anonymous authors.
