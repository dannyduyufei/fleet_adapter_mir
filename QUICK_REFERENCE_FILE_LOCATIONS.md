# Quick Reference: File Locations and Setup Summary

## Files to Copy FROM fleet_adapter_mir repo

```
fleet_adapter_mir/
├── fleet_adapter_mir/              → Copy to: egh_adapters/fleet_adapter_mir/
│   ├── fleet_adapter_mir/
│   │   ├── __init__.py
│   │   ├── fleet_adapter_mir.py
│   │   ├── robot_adapter_mir.py
│   │   └── mir_api.py
│   ├── package.xml
│   ├── setup.py
│   └── setup.cfg
│
├── fleet_adapter_mir_actions/      → Copy to: egh_adapters/fleet_adapter_mir_actions/
│   ├── fleet_adapter_mir_actions/
│   │   ├── __init__.py
│   │   ├── mir_action.py
│   │   ├── rmf_cart_delivery.py
│   │   └── rmf_wait_until.py
│   ├── package.xml
│   └── setup.py
│
└── missions/
    └── rmf_missions.json           → Copy to: egh_adapters/fleet_adapter_mir/missions/
```

## Files to CREATE in Your HLL Repo

### 1. MiR Configuration
**Location**: `egh/config/mir_config.yaml`
**Template**: Use `TEMPLATE_mir_config_for_egh.yaml`
**Purpose**: Main configuration for MiR robots, coordinate transforms, missions

### 2. Launch File (Optional)
**Location**: `egh/launch/egh_mir.launch.xml`
**Template**: Use `TEMPLATE_egh_mir.launch.xml`
**Purpose**: Launch file to start MiR adapter with all components

## Files That Must Already Exist

### 1. Navigation Graph
**Location**: `egh_maps/maps/egh/nav_graphs/0.yaml`
**Generated from**: `egh_maps/maps/egh/egh.building.yaml`
**Command to generate**:
```bash
cd egh_maps/maps/egh
ros2 run rmf_building_map_tools building_map_generator egh.building.yaml
```

### 2. Building Definition
**Location**: `egh_maps/maps/egh/egh.building.yaml`
**Purpose**: Defines waypoints, lanes, lifts (already exists in your repo)

## Final Directory Structure

```
your_hll_repo/
│
├── egh/
│   ├── config/
│   │   ├── deliveryRobot_config.yaml      # Existing sim config
│   │   └── mir_config.yaml                # NEW: MiR config
│   └── launch/
│       ├── common.launch.xml              # Existing
│       ├── egh.launch.xml                 # Existing (modify to add MiR)
│       ├── egh_sim.launch.xml             # Existing sim launch
│       └── egh_mir.launch.xml             # NEW: MiR launch (optional)
│
├── egh_adapters/
│   ├── egh_sim_fleet_adapter/             # Existing sim adapter
│   ├── fleet_adapter_mir/                 # NEW: MiR adapter
│   │   ├── fleet_adapter_mir/
│   │   ├── missions/
│   │   │   └── rmf_missions.json
│   │   ├── package.xml
│   │   └── setup.py
│   └── fleet_adapter_mir_actions/         # NEW: Action plugins
│       ├── fleet_adapter_mir_actions/
│       ├── package.xml
│       └── setup.py
│
└── egh_maps/
    └── maps/
        └── egh/
            ├── egh.building.yaml          # Existing
            ├── egh_lvl_1.png              # Existing
            └── nav_graphs/
                └── 0.yaml                 # Generated or existing
```

## Configuration Checklist

### In mir_config.yaml:

- [ ] **Robot IP address** (line ~107)
  ```yaml
  base_url: "http://YOUR_ROBOT_IP/api/v2.0.0/"
  ```

- [ ] **Robot credentials** (line ~109)
  ```yaml
  password: "Basic YOUR_BASE64_HASH"
  ```
  Get from: MiR web UI → System → API

- [ ] **MiR map name** (line ~32)
  ```yaml
  maps:
    L1: "YOUR_MIR_MAP_NAME"
  ```
  Get from: MiR web UI → Maps

- [ ] **Coordinate reference points** (lines ~12-18)
  ```yaml
  reference_coordinates:
    L1:
      rmf: [[630.5, 1386.5], [533.1, 940.6], [1250.4, 1673.0]]
      mir: [[X1, Y1], [X2, Y2], [X3, Y3]]  # Measure these!
  ```

- [ ] **Charger waypoint name** (line ~103)
  ```yaml
  robots:
    mir_1:
      charger: "deliverybot_charger"  # Must match nav_graph
  ```

- [ ] **Robot physical parameters** (lines ~73-90)
  Adjust for your MiR model (100/250/500/1000)

- [ ] **Fleet name** (line ~71)
  ```yaml
  name: "deliveryRobot"  # Match existing fleet name
  ```

## Build Commands

```bash
# Navigate to repo
cd /path/to/your/hll_repo

# Source ROS 2 and RMF
source /opt/ros/humble/setup.bash
source /path/to/rmf_ws/install/setup.bash  # If separate

# Build MiR packages
colcon build --packages-select fleet_adapter_mir fleet_adapter_mir_actions

# Source workspace
source install/setup.bash
```

## Launch Commands

### Method 1: Direct Command
```bash
ros2 run fleet_adapter_mir fleet_adapter_mir \
  -c $(pwd)/egh/config/mir_config.yaml \
  -n $(pwd)/egh_maps/maps/egh/nav_graphs/0.yaml \
  -r $(pwd)/egh_adapters/fleet_adapter_mir/missions/rmf_missions.json
```

### Method 2: Launch File
```bash
ros2 launch egh egh_mir.launch.xml
```

## Verification Commands

```bash
# Check robot connection
ros2 topic echo /fleet_states

# Monitor tasks
ros2 topic echo /task_summaries

# Check traffic
ros2 topic echo /rmf_traffic_schedule

# List all RMF topics
ros2 topic list | grep rmf
```

## Critical Configuration Points (Priority Order)

1. **Coordinate Transformation** ⭐⭐⭐⭐⭐
   - Most common cause of issues
   - Measure carefully from both RMF and MiR
   - Need at least 3 points, ideally 4-5

2. **Robot Credentials** ⭐⭐⭐⭐
   - Must be correct or adapter won't connect
   - Get fresh from MiR web UI → System → API

3. **Map Name** ⭐⭐⭐⭐
   - Must exactly match MiR's map name
   - Case-sensitive

4. **Charger Waypoint** ⭐⭐⭐
   - Must match nav_graph waypoint name
   - Must have is_charger: true in building.yaml

5. **Mission Names** ⭐⭐⭐
   - Can be auto-created with missions JSON
   - Or create manually on MiR robot

## First Test Workflow

1. **Build workspace** → `colcon build`
2. **Start RMF core** → `ros2 launch egh common.launch.xml`
3. **Start MiR adapter** → `ros2 run fleet_adapter_mir ...`
4. **Verify connection** → `ros2 topic echo /fleet_states`
5. **Send patrol task** → Use task dispatcher or RMF dashboard
6. **Monitor execution** → Watch adapter logs and robot

## Minimal Working Configuration

For the simplest setup (just 3 waypoints on L1):

**Waypoints needed**:
- deliverybot_charger (with is_charger: true)
- pickup_point
- dropoff_point

**Lanes needed**:
- charger ↔ pickup
- pickup ↔ dropoff
- dropoff ↔ charger

**Config needed**:
- mir_config.yaml with correct IP, credentials, coordinates
- nav_graphs/0.yaml (generated from building.yaml)

**Missions needed** (auto-created or manual):
- rmf_move
- rmf_dock_and_charge

This is sufficient for a basic patrol task!

## Getting Help

- MiR adapter issues: https://github.com/open-rmf/fleet_adapter_mir/issues
- RMF documentation: https://osrf.github.io/ros2multirobotbook/
- Check adapter logs first - they're very informative
- Coordinate transformation issues? Re-measure reference points
- Can't connect? Check network, IP, and credentials
