# GitHub Copilot Instructions for Node-RED Matter Bridge

## Project Overview
This is a Node-RED package that creates a Matter Bridge allowing users to expose virtual devices to Matter controllers (Apple Home, Google Home, Alexa, etc.). The bridge translates commands between Matter protocol and non-Matter devices in a Node-RED flow.

**Package Name:** `@sammachin/node-red-matter-bridge`  
**Version:** 0.12.3  
**Author:** Sam Machin  
**License:** MIT  
**Node.js:** >=18.0.0  
**Node-RED:** >=3.0.0

## Architecture

### Core Components
1. **Bridge Node** (`bridge.js`) - Main Matter bridge server that manages the Matter environment
2. **Device Nodes** - Individual device type implementations (e.g., `onofflight.js`, `thermostat.js`)
3. **Device Modules** - Matter device definitions in `devices/` directory
4. **HTML Files** - Node-RED editor configuration for each node type
5. **Utilities** (`utils.js`) - Helper functions for state management and validation
6. **Battery Module** (`battery.js`) - Battery status support for devices

### Project Structure
```
Root/
├── bridge.js, bridge.html          # Main bridge node
├── [device].js, [device].html      # Node-RED node implementations
├── devices/[device].js             # Matter device endpoint definitions
├── utils.js                        # Utility functions
├── battery.js                      # Battery support
└── examples/                       # Example flows
```

## Supported Device Types
- **Lighting:** OnOff Light, OnOff Socket, Dimmable Light, Color Temperature Light, Full Color (RGB) Light
- **Sensors:** Contact Sensor, Light Sensor, Temperature Sensor, Pressure Sensor, Humidity Sensor, Occupancy Sensor
- **Controls:** Generic Switch (Button), Window Covering (Blinds - simple & position-aware), Thermostat, Door Lock, Fan

## Key Concepts

### Matter.js Integration
- Uses `@matter/main` package (v0.12.5) for Matter protocol implementation
- Each device is an `Endpoint` with specific behaviors and clusters
- Bridge uses `AggregatorEndpoint` to manage multiple virtual devices

### Node-RED Pattern
- Each device type has both:
  - A Node-RED node file (e.g., `onofflight.js`) - handles Node-RED runtime logic
  - A device module file (e.g., `devices/onofflight.js`) - defines Matter endpoint

### Device Registration Flow
1. Device node registers with bridge via `node.bridge.addDevice(child)`
2. Bridge creates Matter endpoint using device module
3. Device node handles bidirectional communication between Matter and Node-RED

### Message Flow
- **Input Messages:** Control device from Node-RED flows
- **Output Messages:** Device state changes from Matter controller
- **Passthrough Mode:** When enabled, input messages are passed through to output (prevents feedback loops)

## Coding Standards

### Matter Device Modules (`devices/*.js`)
```javascript
// Pattern for device modules
const { Endpoint } = require("@matter/main");
const { BridgedDeviceBasicInformationServer, IdentifyServer } = require("@matter/main/behaviors");
const { DeviceType } = require("@matter/main/devices");
const { batFeatures, batCluster } = require("../battery");

module.exports = {
    devicename: function(child) {
        const device = new Endpoint(
            DeviceType.with(
                BridgedDeviceBasicInformationServer, 
                IdentifyServer, 
                ...child.bat ? batCluster(child) : []
            ), {
                id: child.id,
                bridgedDeviceBasicInformation: {
                    nodeLabel: child.name,
                    productName: child.name,
                    productLabel: child.name,
                    serialNumber: child.id.replace('-', ''),
                    uniqueId: child.id.replace('-', '').split("").reverse().join(""),
                    reachable: true,
                },
                ...child.bat ? {powerSource: batFeatures(child)} : {}
            }
        );
        return device;
    }
}
```

### Node-RED Node Pattern
- Use `RED.nodes.createNode(this, config)` for initialization
- Store bridge reference: `node.bridge = RED.nodes.getNode(config.bridge)`
- Register device with bridge in `node.on('configured', ...)`
- Handle input with `node.on('input', ...)`
- Clean up in `node.on('close', ...)`

### State Management
- Use `utils.willUpdate(data)` to check if state actually changed before updating
- Devices maintain internal state to avoid unnecessary Matter updates
- Always validate input types with `utils.isNumber()`, `utils.isBoolean()`, etc.

### Battery Support
- Devices can have: None, Replaceable, or Rechargeable battery
- Battery messages use topic `battery`:
  - `msg.battery.level`: 0 (Ok), 1 (Low), 2 (Critical)
  - `msg.battery.percent`: 0-100
  - `msg.battery.charge`: 0 (Not Charging), 1 (Charging)

## Important Implementation Details

### Bridge Initialization
- Generates random passcode and discriminator for Matter pairing
- Creates QR code for easy device commissioning
- Manages Matter storage in user-specified location
- Handles network interface selection for Matter networking

### Device Child Configuration
Each device passed to the bridge contains:
```javascript
{
    id: node.id,
    name: config.name,
    type: 'devicetype',
    node: node,
    bat: config.bat,        // Battery configuration
    batType: config.batType // 'replaceable' or 'rechargeable'
}
```

### Error Handling
- Always wrap Matter operations in try-catch blocks
- Log errors using `node.error()` for Node-RED compatibility
- Set node status to indicate errors: `node.status({fill:"red", shape:"ring", text:"error"})`

### Logging
- Use Node-RED's built-in logging: `node.log()`, `node.warn()`, `node.error()`
- Bridge supports log levels: FATAL, ERROR, WARN, INFO, DEBUG
- Maps to Matter.js `Logger.defaultLogLevel`

## Testing Considerations

### Development Notes
- This is **not a certified Matter device** - for development/experimentation only
- Requires Node.js 18+ for Matter.js compatibility
- Test with multiple controllers (Apple Home, Google Home, Alexa) as behavior varies
- Battery percentage support is inconsistent across controllers

### Common Issues
- Feedback loops: Use passthrough mode carefully
- State synchronization: Ensure bidirectional updates work correctly
- Controller compatibility: Not all features work on all platforms
- Network interface: May need manual selection for Matter networking

## Dependencies
- `@matter/main`: ^0.12.5 - Matter protocol implementation
- `traverse`: ^0.6.10 - Deep object traversal for state comparison

## Adding New Device Types

1. Create device module in `devices/[devicename].js` following the pattern
2. Create Node-RED node file `[devicename].js` with runtime logic
3. Create HTML editor config `[devicename].html`
4. Add to `package.json` under `node-red.nodes`
5. Import device module in `bridge.js`
6. Update README with new device type

## Special Commands

### Topics
- `battery` - Update battery status
- Device-specific topics vary by type (e.g., `color`, `position`, `lock`, etc.)

### Message Formats
Messages should include appropriate payloads for device type:
- Boolean states: `{payload: true/false}`
- Numeric values: `{payload: number}` with validation
- Complex states: `{payload: {property: value}}`

## File Access
All files in this repository should be accessible for reading and modification. The project follows a modular structure where:
- Root-level files are Node-RED node implementations
- `devices/` contains Matter endpoint definitions
- `examples/` contains sample flows
- `resources/` contains web resources (QR code generator)

## Best Practices
1. Maintain consistency with existing device implementations
2. Always include battery support in new devices
3. Test passthrough mode behavior
4. Validate all input values before updating Matter state
5. Use semantic versioning for releases
6. Document breaking changes in CHANGELOG.md
7. Keep device modules focused on Matter endpoint configuration
8. Keep Node-RED nodes focused on flow integration and message handling
