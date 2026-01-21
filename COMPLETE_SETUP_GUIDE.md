# Complete MiR Fleet Adapter Setup Guide for EGH HLL Repository

This guide will help you integrate the MiR fleet adapter into your EGH HLL repository and send your first patrol task to a real MiR robot.

## Prerequisites Checklist

Before starting, ensure you have:

- [ ] MiR robot powered on and connected to network
- [ ] MiR robot IP address
- [ ] MiR robot REST API credentials (username/password or Basic Auth header)
- [ ] ROS 2 environment set up
- [ ] RMF packages installed
- [ ] Access to your HLL repository on AWS server

## Part 1: Copy Files to Your Repository

### 1.1 Clone fleet_adapter_mir Repository

On your AWS server:

```bash
cd /tmp
git clone https://github.com/open-rmf/fleet_adapter_mir.git
```

### 1.2 Copy Packages to Your Repo

```bash
cd /path/to/your/hll_repo

# Copy the two main packages
cp -r /tmp/fleet_adapter_mir/fleet_adapter_mir egh_adapters/
cp -r /tmp/fleet_adapter_mir/fleet_adapter_mir_actions egh_adapters/

# Optional: Copy missions JSON template
mkdir -p egh_adapters/fleet_adapter_mir/missions
cp /tmp/fleet_adapter_mir/missions/rmf_missions.json egh_adapters/fleet_adapter_mir/missions/
```

Your directory structure should now be:
```
egh_adapters/
├── egh_sim_fleet_adapter/          # Existing simulation adapter
├── fleet_adapter_mir/              # NEW: MiR adapter
│   ├── fleet_adapter_mir/
│   │   ├── __init__.py
│   │   ├── fleet_adapter_mir.py
│   │   ├── robot_adapter_mir.py
│   │   └── mir_api.py
│   ├── missions/
│   │   └── rmf_missions.json
│   ├── package.xml
│   └── setup.py
└── fleet_adapter_mir_actions/      # NEW: Action plugins
    ├── fleet_adapter_mir_actions/
    ├── package.xml
    └── setup.py
```

## Part 2: Create Configuration Files

### 2.1 Create MiR Configuration File

Create: `egh/config/mir_config.yaml`

Use the template I provided (TEMPLATE_mir_config_for_egh.yaml) and customize:

**Critical fields to update:**

1. **Robot IP and credentials**:
```yaml
robots:
  mir_1:
    mir_config:
      base_url: "http://YOUR_ROBOT_IP/api/v2.0.0/"
      password: "Basic YOUR_BASE64_HASH"
```

To get the password:
- Go to your MiR robot web interface (http://ROBOT_IP)
- Navigate to: System → API
- Copy the "Basic Authentication" header value
- It should look like: "Basic YWRtaW46ODkyOWRmNTg..."

2. **MiR map name**:
```yaml
conversions:
  maps:
    L1: "YOUR_MIR_MAP_NAME"  # Check MiR web interface → Maps
```

3. **Charger waypoint name**:
```yaml
robots:
  mir_1:
    charger: "deliverybot_charger"  # Must match waypoint in nav_graph
```

4. **Robot physical parameters** (adjust for your MiR model):

| Parameter | MiR 100 | MiR 250 | MiR 500 |
|-----------|---------|---------|---------|
| footprint | 0.45m | 0.6m | 0.8m |
| mass | 60kg | 100kg | 250kg |
| max velocity | 1.2 m/s | 1.2 m/s | 1.5 m/s |
| battery capacity | 20 Ahr | 40 Ahr | 80 Ahr |

### 2.2 Configure Coordinate Transformation

**This is the most critical step!**

You need to measure at least 3 reference points in both coordinate systems.

#### Step-by-step:

1. **Identify reference points from egh.building.yaml**:
   - deliverybot_charger: [630.5, 1386.5]
   - pickup_point: [533.1, 940.6]
   - dropoff_point: [1250.4, 1673.0]

2. **Get corresponding MiR coordinates**:

   Option A - Drive robot to each location:
   ```
   1. Connect to MiR web interface
   2. Switch to Manual mode (hamburger menu → Manual)
   3. Drive robot to charger
   4. Note X, Y position from status display
   5. Repeat for pickup_point and dropoff_point
   ```

   Option B - Use MiR map interface:
   ```
   1. Go to Maps in MiR web interface
   2. Open your Level 1 map
   3. Find charger marker, note coordinates
   4. Find other reference points
   ```

3. **Update mir_config.yaml**:
```yaml
conversions:
  reference_coordinates:
    L1:
      rmf: [[630.5, 1386.5],   # deliverybot_charger
            [533.1, 940.6],     # pickup_point
            [1250.4, 1673.0]]   # dropoff_point
      mir: [[X1, Y1],           # FROM MiR robot
            [X2, Y2],           # FROM MiR robot
            [X3, Y3]]           # FROM MiR robot
```

**IMPORTANT**: Order must match - first RMF point = first MiR point!

### 2.3 Verify Navigation Graph

Check if `egh_maps/maps/egh/nav_graphs/0.yaml` exists:

```bash
ls -la egh_maps/maps/egh/nav_graphs/
```

If it doesn't exist, you need to generate it from `egh.building.yaml`:

```bash
cd egh_maps/maps/egh
ros2 run rmf_building_map_tools building_map_generator egh.building.yaml
```

This will create the `nav_graphs/` directory with navigation graph files.

## Part 3: Build the Workspace

```bash
cd /path/to/your/hll_repo

# Source ROS 2 and RMF
source /opt/ros/humble/setup.bash  # Or your ROS 2 distro
source /path/to/rmf_ws/install/setup.bash  # If you have separate RMF workspace

# Build MiR packages
colcon build --packages-select fleet_adapter_mir fleet_adapter_mir_actions

# Source your workspace
source install/setup.bash
```

## Part 4: Configure MiR Robot

### 4.1 Create Required Missions on MiR

The adapter needs these missions to exist on your MiR robot:

1. **rmf_move** - Basic movement mission
2. **rmf_dock_and_charge** - Charging mission
3. **rmf_localize** - Map switching mission (for multi-floor)
4. **rmf_move_to_position** - Named position movement

#### Option A: Auto-create missions (Recommended)

The adapter can auto-create these missions if you provide the missions JSON:

```bash
ros2 run fleet_adapter_mir fleet_adapter_mir \
  -c egh/config/mir_config.yaml \
  -n egh_maps/maps/egh/nav_graphs/0.yaml \
  -r egh_adapters/fleet_adapter_mir/missions/rmf_missions.json
```

#### Option B: Manual creation

On MiR robot web interface:
1. Go to Missions
2. Create new mission: "rmf_move"
3. Add action: "Move to coordinate" with parameters [x], [y], [orientation]
4. Save mission
5. Repeat for other missions

### 4.2 Configure MiR Charger

Ensure charger is properly configured:
1. In MiR web interface → Positions
2. Find or create charger position
3. Ensure it matches the name and coordinates you're using
4. Type should be "Charging station"

## Part 5: Launch the Adapter

### 5.1 Start RMF Core Components

```bash
# Terminal 1: Launch RMF core (traffic schedule, etc.)
cd /path/to/your/hll_repo
source install/setup.bash
ros2 launch egh common.launch.xml
```

### 5.2 Start MiR Fleet Adapter

```bash
# Terminal 2: Launch MiR fleet adapter
cd /path/to/your/hll_repo
source install/setup.bash

ros2 run fleet_adapter_mir fleet_adapter_mir \
  -c $(pwd)/egh/config/mir_config.yaml \
  -n $(pwd)/egh_maps/maps/egh/nav_graphs/0.yaml \
  -r $(pwd)/egh_adapters/fleet_adapter_mir/missions/rmf_missions.json
```

### 5.3 Verify Connection

Check the adapter logs for:
```
✓ "Successfully connected to robot mir_1"
✓ "Coordinate transformation initialized"
✓ "Fleet adapter ready"
```

If you see errors:
- "Connection refused" → Check robot IP and network
- "Authentication failed" → Check password/credentials
- "Map not found" → Check map name in config matches MiR robot

### 5.4 Monitor Fleet State

In another terminal, check if robot is visible to RMF:

```bash
# Terminal 3: Monitor fleet states
ros2 topic echo /fleet_states
```

You should see your robot with current position and battery level.

## Part 6: Send Your First Patrol Task

### 6.1 Using RMF Task Dispatcher (GUI)

If you have rmf-web-dashboard running:
1. Open dashboard in browser
2. Go to Tasks tab
3. Create new "Loop" task
4. Select waypoints: deliverybot_charger → pickup_point → dropoff_point → deliverybot_charger
5. Submit task

### 6.2 Using ROS 2 CLI

Create a simple patrol task:

```bash
ros2 run rmf_demos_tasks dispatch_loop \
  -s deliverybot_charger \
  -f pickup_point \
  -n 1 \
  --use_sim_time false
```

Parameters:
- `-s` : Start waypoint
- `-f` : Finish waypoint (will go there and back)
- `-n` : Number of loops

### 6.3 Using Custom Task Dispatch Script

Create a Python script to send tasks:

```python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from rmf_task_msgs.msg import ApiRequest
import json
import uuid

def create_patrol_task():
    task = {
        "type": "dispatch_task_request",
        "request": {
            "category": "patrol",
            "description": {
                "places": [
                    "deliverybot_charger",
                    "pickup_point",
                    "dropoff_point",
                    "deliverybot_charger"
                ],
                "rounds": 1
            }
        }
    }
    return task

rclpy.init()
node = Node('patrol_task_sender')
pub = node.create_publisher(ApiRequest, '/task_api_requests', 10)

# Wait for connection
import time
time.sleep(1)

# Send task
request = ApiRequest()
request.request_id = str(uuid.uuid4())
request.json_msg = json.dumps(create_patrol_task())
pub.publish(request)

print(f"Sent patrol task with ID: {request.request_id}")
node.destroy_node()
rclpy.shutdown()
```

Save as `send_patrol_task.py` and run:
```bash
python3 send_patrol_task.py
```

## Part 7: Verify and Debug

### 7.1 Check Robot Behavior

The robot should:
1. Accept the task from RMF
2. Navigate to pickup_point
3. Navigate to dropoff_point
4. Return to deliverybot_charger
5. Report task complete

### 7.2 Monitor Logs

Watch the fleet adapter logs for:
```
"Received navigation request to [x, y, yaw]"
"Transformed RMF coordinates [x1, y1] to MiR [x2, y2]"
"Robot moving to waypoint: pickup_point"
"Robot arrived at waypoint"
"Task completed successfully"
```

### 7.3 Common Issues and Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| Robot doesn't move | Coordinate transformation wrong | Verify reference points, re-measure |
| Robot goes to wrong location | Map name mismatch | Check MiR map name matches config |
| "Mission not found" error | Missing missions on robot | Create missions or use missions JSON |
| Robot stops mid-route | RMF traffic conflict | Check for lane closures, other robots |
| Task rejected | Battery too low | Charge robot first |
| Authentication error | Wrong password | Get fresh Basic Auth from MiR API page |

### 7.4 Debug Mode

Run adapter with debug logging:
```bash
# Edit mir_config.yaml
rmf_fleet:
  debug: True  # Enable debug output
```

Or use mock mode for testing without robot:
```bash
ros2 run fleet_adapter_mir fleet_adapter_mir \
  -c mir_config.yaml \
  -n nav_graph.yaml \
  -m  # Mock mode
```

## Part 8: Integration with Existing Launch Files

### Option A: Create Separate Launch File

Create `egh/launch/egh_mir.launch.xml`:

```xml
<launch>
  <!-- Include common RMF components -->
  <include file="$(find-pkg-share egh)/launch/common.launch.xml">
    <arg name="use_sim_time" value="false"/>
  </include>

  <!-- MiR Fleet Adapter -->
  <executable
    cmd="ros2 run fleet_adapter_mir fleet_adapter_mir
         -c $(find-pkg-share egh)/config/mir_config.yaml
         -n $(find-pkg-share egh_maps)/maps/egh/nav_graphs/0.yaml
         -r $(find-pkg-share fleet_adapter_mir)/missions/rmf_missions.json"
    output="screen"
    shell="true"/>
</launch>
```

Launch with:
```bash
ros2 launch egh egh_mir.launch.xml
```

### Option B: Modify Existing egh.launch.xml

Add the MiR adapter section to your existing `egh/launch/egh.launch.xml`:

```xml
<!-- Add this before the closing </launch> tag -->

<!-- MiR Fleet Adapter -->
<group>
  <executable
    cmd="ros2 run fleet_adapter_mir fleet_adapter_mir
         -c $(find-pkg-share egh)/config/mir_config.yaml
         -n $(find-pkg-share egh_maps)/maps/egh/nav_graphs/0.yaml
         -r $(find-pkg-share fleet_adapter_mir)/missions/rmf_missions.json"
    output="screen"
    shell="true"/>
</group>

<!-- Comment out simulation adapter when using real robots -->
<!-- <include file="$(find-pkg-share egh_sim_fleet_adapter)/launch/fleet_adapter.launch.xml"/> -->
```

## Part 9: Testing Checklist

- [ ] Adapter connects to robot successfully
- [ ] Robot state updates appear in /fleet_states topic
- [ ] Robot can receive and execute simple navigation command
- [ ] Robot reaches correct waypoint (coordinate transformation works)
- [ ] Robot reports task completion
- [ ] Robot returns to charger when idle
- [ ] Robot can execute full patrol loop: charger → pickup → dropoff → charger

## Part 10: Next Steps

Once basic patrol works, you can:

1. **Add more robots**: Duplicate robot config in mir_config.yaml
2. **Enable delivery tasks**: Configure rmf_cart_delivery plugin
3. **Multi-floor operation**: Set up lifts and L2/L3 coordinate transforms
4. **Custom actions**: Implement custom MiR actions using fleet_adapter_mir_actions
5. **Monitoring dashboard**: Set up rmf-web-dashboard for visualization

## Troubleshooting Resources

- Fleet adapter logs: Check Terminal 2 output
- MiR robot logs: Web interface → System → Log
- RMF traffic: `ros2 topic echo /rmf_traffic_schedule`
- Fleet states: `ros2 topic echo /fleet_states`
- Task states: `ros2 topic echo /task_summaries`

## Support

If you encounter issues:
1. Check fleet_adapter_mir GitHub issues: https://github.com/open-rmf/fleet_adapter_mir/issues
2. RMF documentation: https://osrf.github.io/ros2multirobotbook/
3. Check coordinate transformation first - it's the #1 cause of issues

## Quick Reference Commands

```bash
# Build workspace
colcon build --packages-select fleet_adapter_mir fleet_adapter_mir_actions

# Run adapter (full command)
ros2 run fleet_adapter_mir fleet_adapter_mir \
  -c $(pwd)/egh/config/mir_config.yaml \
  -n $(pwd)/egh_maps/maps/egh/nav_graphs/0.yaml \
  -r $(pwd)/egh_adapters/fleet_adapter_mir/missions/rmf_missions.json

# Check fleet state
ros2 topic echo /fleet_states

# List ROS 2 topics
ros2 topic list

# Monitor traffic schedule
ros2 topic echo /rmf_traffic_schedule

# Check task status
ros2 topic echo /task_summaries
```

Good luck with your MiR integration! 🤖
