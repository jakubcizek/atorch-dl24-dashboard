# CLAUDE.md - Technical Specification for DL24 Bluetooth Dashboard

This document describes the internal architecture of `dl24.html` for developers and LLMs extending the project.

## Overview

Single-file HTML application that connects to Atorch DL24P electronic load via Web Bluetooth API and displays real-time measurements.

## File Structure

```
dl24.html
├── <style>           CSS with CSS variables for theming
├── <body>            HTML structure (header, value cards, chart, controls, log)
└── <script>          JavaScript application logic
```

## Bluetooth Communication

### UUIDs
```javascript
SERVICE_UUID        = "0000ffe0-0000-1000-8000-00805f9b34fb"
CHARACTERISTIC_UUID = "0000ffe1-0000-1000-8000-00805f9b34fb"
```

### Message Format (36 bytes for DC meter report)
```
Offset  Size  Description
------  ----  -----------
0-1     2     Header: 0xFF 0x55
2       1     Message type: 0x01 (measurement report)
3       1     Device type: 0x02 (DC meter)
4-6     3     Voltage (big-endian, multiply by 0.1 for V)
7-9     3     Current (big-endian, multiply by 0.001 for A)
10-12   3     Capacity (big-endian, multiply by 10 for mAh)
...
24-25   2     Temperature (big-endian, degrees C)
...
35      1     Checksum (not validated)
```

**Note:** DL24P only partially implements the protocol. Fields not listed above may not work correctly. Protocol reference: https://github.com/syssi/esphome-atorch-dl24/blob/main/docs/protocol-design.md

### Data Reception Flow
1. BLE notifications arrive in chunks (may be fragmented)
2. `handleNotification()` buffers incoming bytes in `data.frameBuffer`
3. When buffer starts with 0xFF 0x55, it's a new frame
4. When buffer reaches 36 bytes, `decodeDCReport()` parses it
5. Decoded values update UI and optionally chart (if recording)

## Data Structures

### Global State Object (`data`)
```javascript
data = {
    device: null,                    // BluetoothDevice
    server: null,                    // BluetoothRemoteGATTServer
    notifyCharacteristic: null,      // BluetoothRemoteGATTCharacteristic
    isConnected: false,              // Connection state
    isReceiving: false,              // Recording state (START/STOP)
    frameBuffer: Uint8Array(64),     // BLE message buffer
    frameBufferLen: 0,               // Current buffer length
    runtimeStartTime: null,          // Timer start timestamp
    runtimeElapsed: 0,               // Accumulated seconds
    runtimeInterval: null,           // setInterval ID
    points: {                        // Recorded data arrays
        timestamps: [],
        voltage: [],
        current: [],
        power: [],
        capacity: [],
        energy: [],
        temperature: []
    },
    accumulatedWh: 0,                // Calculated energy (Wh)
    maxChartDataPoints: 600          // Chart display limit
}
```

### DOM Elements Cache (`elements`)
```javascript
elements = {
    btnConnect, btnStart, btnStop, btnClear, btnSave, btnLogClear,
    statusDot, statusText, logContent, sampleCount, apiWarning,
    values: { voltage, current, power, capacity, energy, temperature, runtime }
}
```

## Chart Configuration

Uses Chart.js with 5 datasets and 5 Y-axes:

| Dataset      | Color   | Y-Axis | Visible |
|--------------|---------|--------|---------|
| Voltage (V)  | #ecf74e | y      | Yes (left) |
| Current (A)  | #5eb02f | y1     | Yes (right) |
| Power (W)    | #5fc7ec | y3     | Yes (right) |
| Charge (mAh) | #3b7eca | y4     | No |
| Energy (Wh)  | #a7e5f0 | y2     | No |

Legend items are clickable to toggle series visibility.

## Key Functions

### Connection
- `connect()` - Initiates Bluetooth pairing and subscribes to notifications
- `disconnect()` - Disconnects from device
- `onDisconnected()` - Handles unexpected disconnection

### Data Processing
- `handleNotification(event)` - Receives and buffers BLE data
- `decodeDCReport(data)` - Parses 36-byte message into values object
- `get16bit/get24bit/get32bit(data, offset)` - Big-endian byte readers

### UI Updates
- `updateValues(values)` - Updates value cards display
- `addChartData(values)` - Adds point to chart and data arrays
- `log(message, type)` - Adds entry to system log

### Recording
- `startRuntime()` - Starts local timer
- `stopRuntime()` - Pauses timer, preserves elapsed time
- `clearData()` - Resets all recorded data and chart
- `saveCSV()` - Exports data as CSV file

## Extending the Application

### Adding a New Measurement Field
1. Add element to HTML value cards section
2. Add element reference to `elements.values`
3. Add array to `data.points`
4. Parse field in `decodeDCReport()` (if available in protocol)
5. Update `updateValues()` to display it
6. Update `addChartData()` to record it
7. Optionally add dataset to chart configuration
8. Update CSV export in `saveCSV()`

### Adding Device Control (future)
The BLE characteristic supports writing. To send commands:
```javascript
const command = new Uint8Array([0xFF, 0x55, /* command bytes */]);
await data.notifyCharacteristic.writeValue(command);
```
Command format is documented in the protocol reference but not tested with DL24P.

### Adding New Chart Series
1. Add dataset object to `chart.data.datasets[]`
2. Add corresponding Y-axis to `chart.options.scales`
3. Add legend item to `.chart-legend` div
4. Update `addChartData()` to populate the dataset

## CSS Variables (Theming)

```css
--bg-primary: #0f172a      /* Main background */
--bg-secondary: #1e293b    /* Cards background */
--bg-card: #334155         /* Buttons, borders */
--text-primary: #f8fafc    /* Main text */
--text-secondary: #94a3b8  /* Labels, muted text */
--accent-blue: #3b82f6
--accent-green: #22c55e
--accent-red: #ef4444
--accent-yellow: #eab308
--accent-purple: #a855f7
--accent-cyan: #06b6d4
```

## Browser Requirements

- Web Bluetooth API (Chrome, Edge, Opera)
- ES6+ (async/await, template literals, arrow functions)
- No build step required - runs directly in browser
