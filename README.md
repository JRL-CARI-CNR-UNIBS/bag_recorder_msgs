# bag_recorder

Two ROS 2 packages that add **start / stop bag-recording services** to your system.

| Package | Purpose |
|---|---|
| `bag_recorder_msgs` | Custom `.srv` definitions |
| `bag_recorder` | Python node that hosts the services |

---

## Services

### `/bag_recorder/start`  (`bag_recorder_msgs/srv/StartRecording`)

```
# Request
string[]  topics      # e.g. ["/camera/image_raw", "/imu/data"]
string    bag_name    # output path, e.g. "my_recording" or "/data/bags/run1"
---
# Response
bool    success
string  message
```

### `/bag_recorder/stop`  (`bag_recorder_msgs/srv/StopRecording`)

```
# Request
string  bag_name      # must match the name used at start
---
# Response
bool    success
string  message
```

---

## Build

```bash
cd ~/ros2_ws
colcon build --packages-select bag_recorder_msgs bag_recorder
source install/setup.bash
```

> **Note:** `bag_recorder_msgs` must be built first (or together) because
> `bag_recorder` depends on it.

---

## Run

```bash
ros2 run bag_recorder bag_recorder_node
```

---

## Usage examples

**Start recording:**
```bash
ros2 service call /bag_recorder/start bag_recorder_msgs/srv/StartRecording \
  "{topics: ['/chatter', '/tf'], bag_name: 'my_bag'}"
```

**Stop recording:**
```bash
ros2 service call /bag_recorder/stop bag_recorder_msgs/srv/StopRecording \
  "{bag_name: 'my_bag'}"
```

**From Python:**
```python
import rclpy
from rclpy.node import Node
from bag_recorder_msgs.srv import StartRecording, StopRecording

rclpy.init()
node = Node('test_client')

start_cli = node.create_client(StartRecording, '/bag_recorder/start')
start_cli.wait_for_service()

req = StartRecording.Request()
req.topics = ['/chatter', '/tf']
req.bag_name = 'my_bag'
future = start_cli.call_async(req)
rclpy.spin_until_future_complete(node, future)
print(future.result())
```

---

## How it works

1. On `start`, a **daemon thread** is spawned per recording.
2. The thread creates a `rosbag2_py.SequentialWriter` (MCAP format by default).
3. A lightweight helper `rclpy.Node` is created inside the thread to
   host generic subscriptions.  Topics are discovered dynamically —
   the node waits up to **5 s** for each topic to appear.
4. On `stop`, a `threading.Event` signals the thread to flush and close
   the writer cleanly.
5. Multiple bags can be recorded **simultaneously** (keyed by `bag_name`).

### Storage format

Default is `mcap`.  To use SQLite3 instead, change `storage_id='mcap'`
to `storage_id='sqlite3'` in `bag_recorder_node.py`.

---

## Dependencies

- `rclpy`
- `rosbag2_py`
- `bag_recorder_msgs` (this repo)
