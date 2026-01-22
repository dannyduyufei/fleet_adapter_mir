# ✅ CORRECTED Action Plan for MIR Fleet Adapter Setup

## Current Status Check

You mentioned you've uploaded to your HLL repository:
- ✅ `fleet_adapter_mir/` folder
- ✅ `fleet_adapter_mir_actions/` folder
- ✅ Created `HLL_grab_mir_config.yaml` in `egh/config/`

## What's Already Complete in Those Folders

The folders you uploaded already contain:
- ✅ `setup.py` and `setup.cfg` (both packages)
- ✅ `package.xml` (both packages)
- ✅ Python source code (fleet_adapter_mir.py, robot_adapter_mir.py, mir_api.py)
- ✅ Action plugins (rmf_cart_delivery.py, rmf_wait_until.py, etc.)

## What You Need to ADD in Your HLL Repo

### 1. Create Launch File Structure

In your HLL repo on AWS:

```bash
cd /path/to/HLL_repo/egh_adapters/fleet_adapter_mir

# Create launch directory
mkdir -p launch

# Create the launch file (see fleet_adapter_mir_launch_template.xml)
nano launch/fleet_adapter_mir.launch.xml
```

Copy the contents from `fleet_adapter_mir_launch_template.xml` I just created.

### 2. Add Missions Folder

```bash
cd /path/to/HLL_repo/egh_adapters/fleet_adapter_mir

# Create missions directory
mkdir -p missions
```

Copy `rmf_missions.json` from the fleet_adapter_mir repo:
```bash
# Copy from where you cloned fleet_adapter_mir originally
cp /path/to/original/fleet_adapter_mir/missions/rmf_missions.json missions/
```

Or download it from: https://github.com/open-rmf/fleet_adapter_mir/blob/main/missions/rmf_missions.json

### 3. Update setup.py

Replace the `setup.py` in `egh_adapters/fleet_adapter_mir/setup.py` with the content from `updated_setup.py` I just created.

This adds:
- Launch file installation
- Missions file installation
- `nudged` dependency (needed for coordinate transformation)

### 4. Create or Verify Navigation Graph

Check if this file exists:
```bash
ls -la egh_maps/maps/egh/nav_graphs/0.yaml
```

If it **doesn't exist**, you need to generate it:

**Option A: Generate from building.yaml**
```bash
cd egh_maps/maps/egh
ros2 run rmf_building_map_tools building_map_generator egh.building.yaml
```

**Option B: Create manually** - Use the template nav_graph I provided earlier with your waypoint coordinates.

### 5. Review Your Config File

I need to see your `HLL_grab_mir_config.yaml` to verify it's correct. The critical fields are:

```yaml
# Robot connection
robots:
  mir_1:
    charger: "deliverybot_charger"  # Must match nav_graph waypoint name
    mir_config:
      base_url: "http://10.103.177.2/api/v2.0.0/"  # Your robot IP
      password: "Basic YOUR_BASE64_HASH"  # From MiR web UI

# Coordinate transformation (CRITICAL!)
conversions:
  reference_coordinates:
    L1:
      rmf: [[630.5, 1386.5], [533.1, 940.6], [1250.4, 1673.0]]
      mir: [[?, ?], [?, ?], [?, ?]]  # YOU MUST MEASURE THESE

  maps:
    L1: "YOUR_MIR_MAP_NAME"  # Check MiR web UI → Maps
```

---

## File Structure You Should Have in HLL Repo

After completing the above steps:

```
HLL_repo/
├── egh/
│   └── config/
│       ├── deliveryRobot_config.yaml        # Existing sim config
│       └── HLL_grab_mir_config.yaml         # ✅ Your new MiR config
│
├── egh_adapters/
│   ├── egh_sim_fleet_adapter/               # Existing
│   ├── fleet_adapter_mir/                   # ✅ You uploaded this
│   │   ├── fleet_adapter_mir/               # ✅ Source code (exists)
│   │   ├── resource/                        # ✅ Exists
│   │   ├── launch/                          # ⚠️ ADD THIS
│   │   │   └── fleet_adapter_mir.launch.xml
│   │   ├── missions/                        # ⚠️ ADD THIS
│   │   │   └── rmf_missions.json
│   │   ├── package.xml                      # ✅ Exists
│   │   ├── setup.py                         # ⚠️ REPLACE with updated version
│   │   └── setup.cfg                        # ✅ Exists
│   │
│   └── fleet_adapter_mir_actions/           # ✅ You uploaded this
│       ├── fleet_adapter_mir_actions/       # ✅ Complete
│       ├── package.xml                      # ✅ Exists
│       ├── setup.py                         # ✅ Exists
│       └── setup.cfg                        # ✅ Exists
│
└── egh_maps/
    └── maps/egh/
        ├── egh.building.yaml                # ✅ Exists
        └── nav_graphs/
            └── 0.yaml                       # ⚠️ Verify this exists
```

---

## Build and Test Steps

### On Friend's Laptop (Testing)

```bash
# 1. Clone your HLL repo
git clone <your-repo-url> HLL
cd HLL
git checkout <your-branch>  # Whatever branch you're working on

# 2. Install dependencies
sudo apt install python3-pip
pip3 install nudged

# 3. Build
cd /path/to/ros2_workspace
colcon build --packages-select fleet_adapter_mir fleet_adapter_mir_actions

# 4. Source workspace
source install/setup.bash

# 5. Test in mock mode (no robot needed)
ros2 launch fleet_adapter_mir fleet_adapter_mir.launch.xml \
  config_file:=$(pwd)/src/HLL/egh/config/HLL_grab_mir_config.yaml \
  nav_graph_file:=$(pwd)/src/HLL/egh_maps/maps/egh/nav_graphs/0.yaml \
  rmf_missions_file:=$(pwd)/src/HLL/egh_adapters/fleet_adapter_mir/missions/rmf_missions.json \
  mock:=true

# Expected output: Fleet adapter starts, no errors about missing files
```

### On AWS Server (Real Robot)

```bash
# 1. Pull latest code
cd /path/to/HLL_repo
git pull origin <your-branch>

# 2. Build
cd /path/to/ros2_workspace
colcon build --packages-select fleet_adapter_mir fleet_adapter_mir_actions

# 3. Source
source install/setup.bash

# 4. Launch with real robot
ros2 launch fleet_adapter_mir fleet_adapter_mir.launch.xml \
  config_file:=$(pwd)/src/HLL/egh/config/HLL_grab_mir_config.yaml \
  nav_graph_file:=$(pwd)/src/HLL/egh_maps/maps/egh/nav_graphs/0.yaml \
  rmf_missions_file:=$(pwd)/src/HLL/egh_adapters/fleet_adapter_mir/missions/rmf_missions.json

# Expected: Connects to robot at 10.103.177.2, shows fleet state
```

---

## Alternative: Run Without Launch File

You can also run the fleet adapter directly (the original way):

```bash
ros2 run fleet_adapter_mir fleet_adapter_mir \
  -c $(pwd)/src/HLL/egh/config/HLL_grab_mir_config.yaml \
  -n $(pwd)/src/HLL/egh_maps/maps/egh/nav_graphs/0.yaml \
  -r $(pwd)/src/HLL/egh_adapters/fleet_adapter_mir/missions/rmf_missions.json
```

This is simpler and doesn't require the launch file!

---

## Critical Next Steps

1. **FIRST**: Share your `HLL_grab_mir_config.yaml` so I can review it
2. **SECOND**: Measure MiR coordinates for the reference points
3. **THIRD**: Verify nav_graphs/0.yaml exists or generate it
4. **FOURTH**: Add launch folder and missions folder
5. **FIFTH**: Build and test

---

## Getting MiR Coordinates for Reference Points

This is **THE MOST CRITICAL STEP**. You need to:

1. **Drive robot to deliverybot_charger location**
   - Open MiR web UI: http://10.103.177.2
   - Go to Manual mode
   - Drive robot to the physical charger location
   - Note the X, Y coordinates shown on screen
   - Example: [5.2, 3.1]

2. **Drive robot to pickup_point location**
   - Drive to where pickup_point should be
   - Note X, Y coordinates
   - Example: [8.5, 10.3]

3. **Drive robot to dropoff_point location**
   - Drive to where dropoff_point should be
   - Note X, Y coordinates
   - Example: [15.2, 2.8]

4. **Update config file**:
```yaml
conversions:
  reference_coordinates:
    L1:
      rmf: [[630.5, 1386.5], [533.1, 940.6], [1250.4, 1673.0]]
      mir: [[5.2, 3.1], [8.5, 10.3], [15.2, 2.8]]  # Your measured values
```

---

## Quick Checklist

Before you can test:

- [ ] Launch folder created with fleet_adapter_mir.launch.xml
- [ ] Missions folder created with rmf_missions.json
- [ ] setup.py updated to include launch and missions
- [ ] nav_graphs/0.yaml exists
- [ ] HLL_grab_mir_config.yaml reviewed and correct
- [ ] MiR coordinates measured and added to config
- [ ] Built with colcon
- [ ] Tested in mock mode on laptop
- [ ] Deployed to AWS server
- [ ] Connected to real robot

---

## Need Help?

Share your `HLL_grab_mir_config.yaml` file and I'll review it step by step!
