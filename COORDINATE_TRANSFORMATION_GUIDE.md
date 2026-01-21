# Coordinate Transformation Setup Guide for MiR Fleet Adapter

## Why This Is Critical

RMF (your egh.building.yaml) and MiR robots use **different coordinate systems**:
- **RMF**: Uses coordinates from your building.yaml (likely in meters or pixels from reference image)
- **MiR**: Uses its own map coordinate system (meters from map origin)

The fleet adapter needs to **transform** RMF waypoint coordinates to MiR coordinates.

## Step-by-Step Coordinate Calibration

### 1. Identify Reference Points

Choose **at least 3 waypoints** that you can measure in both systems. Good choices:
- Charging station
- Pickup point
- Dropoff point
- Any clearly identifiable physical landmark

From your egh.building.yaml L1, you have:
- `deliverybot_charger` (vertex 10): RMF coords = [630.5, 1386.5]
- `pickup_point` (vertex 21): RMF coords = [533.1, 940.6]
- `dropoff_point` (vertex 22): RMF coords = [1250.4, 1673.0]

### 2. Get MiR Coordinates for These Points

Option A: **Drive robot to each point and read coordinates**

1. Connect to MiR robot web interface (http://ROBOT_IP)
2. Switch to Manual mode
3. Drive robot to the charger location
4. Note the **X, Y position** displayed on screen (e.g., [5.2, 3.1])
5. Repeat for pickup_point and dropoff_point
6. Record all coordinates

Option B: **Use MiR map interface**

1. Go to MiR web interface -> Maps
2. Open your "Level 1" map
3. Click on the charger marker
4. Note the X, Y coordinates
5. Repeat for other points

### 3. Fill in the Configuration

In `mir_config.yaml`, update the `reference_coordinates` section:

```yaml
conversions:
  reference_coordinates:
    L1:
      rmf: [[630.5, 1386.5],    # deliverybot_charger (from egh.building.yaml)
            [533.1, 940.6],      # pickup_point
            [1250.4, 1673.0]]    # dropoff_point
      mir: [[5.2, 3.1],          # deliverybot_charger (from MiR robot)
            [8.5, 10.3],         # pickup_point (from MiR robot)
            [15.2, 2.8]]         # dropoff_point (from MiR robot)
```

**IMPORTANT**: The order must match! First RMF point must correspond to first MiR point.

### 4. Verify Coordinate System Orientation

Check if coordinate systems have the same orientation:
- RMF typically uses: +X = East, +Y = North
- MiR uses: +X = forward on map, +Y = left on map

If orientations differ, you may need additional reference points or rotation adjustments.

### 5. Test Transformation (After Setup)

When you run the adapter, check the logs for transformation warnings:
```bash
ros2 run fleet_adapter_mir fleet_adapter_mir -c mir_config.yaml -n nav_graph.yaml
```

Look for messages like:
- "Coordinate transformation initialized successfully"
- "Transformation error: X meters" (should be < 0.5m for good calibration)

### 6. Fine-Tuning

If robot doesn't reach waypoints precisely:
1. Add more reference points (4-5 is better than 3)
2. Choose points spread across the entire map
3. Verify MiR coordinates were measured at the exact same physical location

## Common Issues

### Issue 1: Robot goes to wrong location
**Cause**: Coordinate transformation is incorrect
**Fix**: Verify you measured the correct points on both systems

### Issue 2: Robot gets close but not exact
**Cause**: Not enough reference points or points too close together
**Fix**: Add more reference points spread across the map

### Issue 3: Robot rotation is wrong
**Cause**: Coordinate systems have different rotation
**Fix**: May need to adjust angle conversion (check adapter logs)

## Quick Calibration Test

1. Send robot to a known waypoint via RMF
2. Check if robot arrives at correct physical location
3. If not, measure the error (how many meters off?)
4. Adjust reference coordinates and retry

## Navigation Graph Requirements

Your `nav_graph.yaml` (or generated from egh.building.yaml) must have:
- Waypoint names matching those used in mir_config.yaml
- At least one waypoint marked with `is_charger: true`
- Lanes connecting all waypoints you want robot to traverse

## Example: Minimal Working Setup

For a simple patrol between 3 points:

**egh.building.yaml** (already exists):
```yaml
vertices:
  - [630.5, 1386.5, 0, {name: "deliverybot_charger", is_charger: true}]
  - [533.1, 940.6, 0, {name: "pickup_point", is_holding_point: true}]
  - [1250.4, 1673.0, 0, {name: "dropoff_point", is_holding_point: true}]
lanes:
  - [0, 1, {graph_idx: 0}]  # charger <-> pickup
  - [1, 2, {graph_idx: 0}]  # pickup <-> dropoff
  - [2, 0, {graph_idx: 0}]  # dropoff <-> charger
```

**mir_config.yaml**:
```yaml
conversions:
  reference_coordinates:
    L1:
      rmf: [[630.5, 1386.5], [533.1, 940.6], [1250.4, 1673.0]]
      mir: [[X1, Y1], [X2, Y2], [X3, Y3]]  # Get these from MiR robot

robots:
  mir_1:
    charger: "deliverybot_charger"  # Must match waypoint name
    mir_config:
      base_url: "http://ROBOT_IP/api/v2.0.0/"
      user: "application/json"
      password: "Basic YOUR_HASH"
```

This setup enables patrol tasks between these 3 points.
