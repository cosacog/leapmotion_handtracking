# Leap Motion Hand Tracking - AI Agent Instructions

## Project Overview

This is a Python project for recording and analyzing Leap Motion hand tracking data. The codebase consists of:
- **leapc-cffi**: CFFI bindings to the Leap C SDK (compiles C bindings to Python)
- **leapc-python-api**: High-level Python API wrapping CFFI bindings
- **Recording/Analysis Tools**: Scripts for capturing, visualizing, and analyzing hand tracking data

## Architecture & Data Flow

### Event-Driven Model
The project uses an event listener pattern with three main workflow types:

1. **Connection Layer** (`connection.py`): Manages connection to Leap server with polling thread
   - Listeners subscribe to events (connection, tracking, device, errors)
   - `on_tracking_event()` fires ~100 Hz when hands are detected
   
2. **Data Capture** (`record_handtracking.py`): Uses producer-consumer threading
   - **Producer**: `RecordingListener` runs on Leap callback thread, pushes `FrameData` to queue
   - **Consumer**: Writer thread pulls frames from queue, buffers in memory, flushes to HDF5 every 0.5s
   - **Main thread**: Handles Ctrl+C for graceful shutdown

3. **Data Analysis**: Process-to-file workflows
   - HDF5 files store hand pose data (palm position/orientation, finger bones, arm joints)
   - `analyze_recording.py` detects frame drops via timestamp gaps
   - `visualize_recording.py` plays back recordings with OpenCV

### Key Data Structures

**Hand pose per frame** (in `record_handtracking.py`):
- Palm: position [x,y,z], orientation [x,y,z,w] quaternion, width
- Arm: wrist position, elbow position
- Fingers: 5 digits × 4 bones × 2 joints (prev/next) × 3 coordinates
- Task status: binary flag toggled by Space key press (for marking task windows)

**HDF5 File Structure** (see `record_handtracking.py` lines 20-35):
```
leap_timestamp: [N]         # Frame timestamp in microseconds
task_status: [N]            # Task on/off binary marker
left_hand/right_hand:       # Parallel datasets for each hand
  valid: [N]                # Hand presence indicator
  palm_pos: [N, 3]
  palm_ori: [N, 4]
  wrist_pos, elbow_pos: [N, 3]
  fingers: [N, 5, 4, 2, 3]  # (finger, bone, joint, coord)
```

## Critical Patterns & Conventions

### Listener Implementation
Always subclass `leap.Listener` and override specific event methods. The base `on_event()` dispatches to `on_tracking_event()`, `on_connection_event()`, etc. (see `event_listener.py` for event dispatch mapping).

Example (from `tracking_event_example.py`):
```python
class MyListener(leap.Listener):
    def on_tracking_event(self, event):
        for hand in event.hands:
            print(hand.palm.position.x)
```

### Threading for Recording
- Use `queue.Queue` for thread-safe producer-consumer communication (not multiprocessing)
- Producer runs on Leap's callback thread, must be fast (avoid I/O)
- Consumer handles slow I/O (HDF5 writes); buffer frames in memory before flush
- Set queue size large enough: 2000 frames at 100 Hz ≈ 20 seconds of tolerance

### Hand Type Detection
Always check `str(hand.type) == "HandType.Left"` (returns string, not enum) to distinguish left/right hands.

### Configuration & Flexibility
Environment variable `LEAPSDK_INSTALL_LOCATION` overrides default SDK path (see README). Some Python versions need to compile CFFI bindings; check `pyproject.toml` for supported versions.

## Developer Workflows

### Setup & Running
```bash
# Install base dependencies
pip install -r requirements.txt

# Install API bindings (assumes Leap SDK installed at default location)
pip install -e leapc-python-api

# Run basic example
python examples/tracking_event_example.py

# For custom SDK location
export LEAPSDK_INSTALL_LOCATION="C:\Program Files\CustomDir\Ultraleap\LeapSDK"
pip install -e leapc-cffi  # Recompile if needed
```

### Recording & Analysis Workflow
```bash
# Record hand data to HDF5
python record_handtracking.py  # Saves to data/leap_recording_<timestamp>.h5

# Analyze recording (frame drops, timing stats)
python analyze_recording.py data/leap_recording_20260120_233030.h5

# Visualize playback
python visualize_recording.py  # Select file from UI
```

### Testing Strategy
- **No automated tests** (hardware-dependent: requires Leap Motion device)
- Manual verification: run examples with device connected, check console output for frame counts
- For timing: compare recorded timestamp intervals (~10ms expected at 100 Hz) using `analyze_recording.py`

## Project Constraints & Trade-offs

1. **Platform SDK Dependency**: Code is tightly coupled to Leap SDK C headers (LeapC.h in `leapc-cffi`). CFFI bindings must match SDK version.

2. **Threading Model**: Producer on callback thread is intentionally minimal; all buffering/I/O on consumer thread to avoid blocking Leap events.

3. **HDF5 Chunking**: 1000-frame chunks balance write performance vs memory usage; don't change without load testing.

4. **Frame Drop Detection**: Based on timestamp gaps, not queue overflow events; assumes ~100 Hz acquisition rate.

## Key Files Reference

- **API Core**: `leapc-python-api/src/leap/{connection,events,datatypes,event_listener}.py`
- **Recording**: `record_handtracking.py` (producer-consumer pattern), `analyze_recording.py` (statistics)
- **Visualization**: `visualize_recording.py` (OpenCV rendering, 2D coordinate mapping)
- **Examples**: `examples/tracking_event_example.py` (simplest listener), `examples/multi_device_example.py` (multi-device patterns)

## Integration Points

- **Leap SDK**: Accessed via `leapc_cffi._leapc_cffi` (compiled C bindings)
- **HDF5**: Via `h5py` for recording and analysis
- **Input**: `pynput.keyboard` for task status toggling (Space key)
- **Visualization**: `opencv-python` for rendering, `numpy` for coordinate math
