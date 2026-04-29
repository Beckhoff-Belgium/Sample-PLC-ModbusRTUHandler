# Sample-PLC-ModbusRTUHandler - Queue-Based Modbus RTU Framework

Reusable TwinCAT 3 framework for decoupled, multi-device Modbus RTU communication via a shared FIFO command queue.

## Overview

`ModbusHandler` and `ModbusDevice` form a two-layer architecture: a background task drives all serial I/O through a state-machine-based command processor, while device driver function blocks compose the handler to queue commands and receive results asynchronously. The framework uses a FIFO ring buffer to decouple multiple device drivers from a single shared serial bus, and union-based type punning for zero-overhead endian conversion between big-endian Modbus registers and little-endian TwinCAT memory.

### Key Features

- **Queue-based command dispatch**: Multiple device drivers push `ST_ModbusCommand` entries into a shared `FifoBuffer`; the handler processes them one at a time over the serial bus
- **Asynchronous result notification**: Each command carries a `POINTER TO HRESULT`; the handler writes the result on completion so callers are never blocked
- **Dual-task isolation**: `ModbusHandler` runs in a dedicated 1 ms `BGComm` task; device logic runs at 10 ms in `PlcTask`, preventing serial I/O from blocking control cycles
- **Endian conversion built-in**: `ModbusReal` and `ModbusDint` unions provide zero-copy big-endian ↔ little-endian conversion
- **HMI-ready device template**: `ST_ModbusDevice_HMI` with `TcHmiSymbol.AddSymbol` for automatic TcHMI symbol binding
- **Configurable timeout and queue depth**: `ModbusTimeout` input and `Param.MAXMODBUSBUFFER` constant control runtime behavior
- **All standard Modbus functions**: Read/write coils, read/write registers, input registers, and diagnostics via `E_ModbusFunction`
- **Template-first design**: `ModbusDevice` is explicitly a copy-and-adapt template; register layout, data types, and HMI interface are all customization points

## Use Cases

### Multi-device polling

- **Cyclic sensor reads**: Poll multiple Modbus RTU sensors at independent intervals by assigning different `UpdateTime` values per device instance
- **Shared bus, multiple slaves**: Address up to 247 slave devices via a single EL6001/EL6022 terminal without additional hardware
- **Independent error isolation**: Each device instance tracks its own error state; one faulty device does not stall the queue

### HMI integration

- **Live process values**: Expose `PVProcessValue1`, `PVProcessValue2`, and `SerialNumber` to TcHMI via the `ST_ModbusDevice_HMI` struct
- **Operator commands**: `CmdReinitialize` and `CmdCalibrate` booleans let operators trigger actions from HMI without PLC code changes
- **Status feedback**: `HasError`, `Initialized`, and `CalibrationSucceded` provide real-time device state to HMI panels

### Device driver development

- **Register map adaptation**: Modify `_Initialize()` to match your device's register layout with minimal boilerplate
- **Triggered writes**: Use rising-edge detection (`R_TRIG`) on HMI inputs to fire one-shot calibration or configuration writes
- **Engineering unit conversion**: Use or extend the built-in `_ConvertModbusToReal()` / `_ConvertRealToModbus()` helpers for your data types

## Function Blocks

### ModbusHandler - Command queue processor

**Queue-based Modbus RTU master** with internal state machine. Accepts `ST_ModbusCommand` entries from any number of device drivers and dispatches them sequentially over the serial bus.

**Features**:
- FIFO ring buffer (`Tc3_AnyBuffer.FifoBuffer`) holds up to `MAXMODBUSBUFFER` (200) commands
- State machine: `WAITING` → `EXECUTE` → `COOLDOWN` → `ERROR`
- Configurable `ModbusTimeout` per command (default `T#500MS`)
- Results written back via `POINTER TO HRESULT` — callers poll asynchronously
- Logs error codes via `ADSLOGSTR` on transition to ERROR state

**Performance**:
- `AddToBuffer`: O(1) — single FIFO push, returns immediately
- Command dispatch: O(1) per cycle — one state machine transition
- Memory: fixed O(n) where n = `MAXMODBUSBUFFER`, allocated at startup

**Best for**: any application with multiple Modbus RTU devices on a single serial bus, shared EL6001/EL6022 terminal, priority-isolated communication task.

---

### ModbusDevice - Device driver template

**Composition-based sample device driver** built on top of `ModbusHandler`. Demonstrates the full pattern: initialization, cyclic reads, triggered writes, endian conversion, and HMI binding.

**Features**:
- Cyclic register reads triggered by `TON` timer at configurable `UpdateTime` (default `T#1000MS`)
- One-shot serial number read on initialization
- Calibration write triggered via `R_TRIG` on `hmi.CmdCalibrate`
- Endian conversion for REAL and UDINT via `ModbusReal`/`ModbusDint` unions
- `ST_ModbusDevice_HMI` struct with `TcHmiSymbol.AddSymbol` attribute for automatic TcHMI symbol registration

**Performance**:
- `_Initialize`: O(1) — sets register addresses and pointers once on first scan
- Cyclic body: O(1) — timer check, queue push, result poll
- Endian conversion: O(1) — union overlay, no copy or shift operations

**Best for**: copying as a starting point for a new Modbus RTU device driver; demonstrating the queue/result pattern to developers new to the framework.

## Usage Examples

**Declare the shared handler in the background communication program:**

```iecst
// BG_Communication.TcPOU — runs in BGComm task (1 ms, priority 4)
PROGRAM BG_Communication
VAR
    modbushandler : ModbusHandler;
END_VAR

modbushandler();
```

**Declare device instances in the main program:**

```iecst
// MAIN.TcPOU — runs in PlcTask (10 ms, priority 8)
PROGRAM MAIN
VAR
    ModbusDevice1 : ModbusDevice;
    ModbusDevice2 : ModbusDevice;
    ModbusDevice3 : ModbusDevice;
END_VAR

ModbusDevice1(
    Param         := (MBSAdress := 20, UpdateTime := T#1000MS),
    ModbusHandler := BG_Communication.modbushandler
);

ModbusDevice2(
    Param         := (MBSAdress := 30, UpdateTime := T#1000MS),
    ModbusHandler := BG_Communication.modbushandler
);

ModbusDevice3(
    Param         := (MBSAdress := 40, UpdateTime := T#500MS),
    ModbusHandler := BG_Communication.modbushandler
);
```

**Submit a command manually from a custom device driver:**

```iecst
VAR
    _cmd    : ST_ModbusCommand;
    _result : HRESULT := S_Pending;
END_VAR

_cmd.ModbusFunction := E_ModbusFunction.ReadRegs;
_cmd.UnitID         := 30;
_cmd.MBAddr         := 4;
_cmd.Quantity       := 2;
_cmd.pMemoryAddr    := ADR(_rawBuffer);
_cmd.cbLength       := SIZEOF(_rawBuffer);
_cmd.Result         := ADR(_result);

ModbusHandler.AddToBuffer(_cmd);

// Later — poll for completion:
IF _result = S_OK THEN
    // process _rawBuffer
ELSIF _result = E_FAIL THEN
    // handle error
END_IF
```

**Adapt `_Initialize()` when copying `ModbusDevice` for a new device:**

```iecst
METHOD _Initialize : BOOL
    // Map your device register layout here
    _readProcessValue1.UnitID         := Param.MBSAdress;
    _readProcessValue1.MBAddr         := 4;      // <-- your register address
    _readProcessValue1.Quantity       := 2;
    _readProcessValue1.pMemoryAddr    := ADR(_rawValueProcessValue1);
    _readProcessValue1.cbLength       := SIZEOF(_rawValueProcessValue1);
    _readProcessValue1.Result         := ADR(_readProcessValue1Result);
    _readProcessValue1.ModbusFunction := E_ModbusFunction.ReadRegs;
```

## Project Structure

```
Sample-PLC-ModbusRTUHandler/
├── LICENSE.md
├── README.md
├── THIRD-PARTY-LICENSES.md
├── Documentation/
│   └── TF6255_TC3_Modbus_RTU_EN.pdf
└── Source/Solution/
    └── ModbusRTUHandler/
        ├── ModbusRTUHandler.tsproj
        ├── _Config/
        │   ├── IO/Device 1 (EtherCAT)/
        │   │   ├── Term 1 (EK1100)/
        │   │   │   └── Term 2 (EL6001).xti
        │   │   └── Term 1 (EK1200)/
        │   │       └── Term 2 (EL6022).xti
        │   └── PLC/
        └── PLC/
            ├── BGComm.TcTTO               (1 ms, priority 4)
            ├── PlcTask.TcTTO              (10 ms, priority 8)
            ├── Program/
            │   ├── MAIN.TcPOU
            │   ├── BG_Communication.TcPOU
            │   └── Param.TcGVL
            └── Components/
                ├── ModbusHandler/
                │   ├── ModbusHandler.TcPOU
                │   └── datatypes/
                │       ├── E_ModbusFunction.TcDUT
                │       └── ST_ModbusCommand.TcDUT
                └── ModbusDeviceSample/
                    ├── ModbusDevice.TcPOU
                    ├── ST_ModbusDeviceParam.TcDUT
                    ├── ST_ModbusDevice_HMI.TcDUT
                    ├── ModbusReal.TcDUT
                    └── ModbusDint.TcDUT
```

## Installation

### Option 1: Add to an existing TwinCAT project

1. Copy the `Components/ModbusHandler/` folder into your PLC project
2. Install `Tc3_AnyBuffer` from the KimRo library repository and add it as a library reference
3. Add `Tc2_ModbusRTU`, `Tc2_Standard`, `Tc2_System`, and `Tc3_Module` library references (Beckhoff Automation)
4. Create a dedicated background task (e.g., 1 ms, priority ≤ 5) and add a `BG_Communication` program to it
5. Declare a `ModbusHandler` instance in `BG_Communication` and call it once per cycle
6. Copy `ModbusDeviceSample/ModbusDevice.TcPOU` as a template for each new device driver
7. Configure your EL6001 or EL6022 hardware terminal and link `ModbusRtuMasterV2_KL6x22B` I/O variables

### Option 2: Clone and run the sample

1. Clone the repository
2. Open `Source/Solution/ModbusRTUHandler.sln` in TwinCAT XAE (minimum version 3.1.4026.18)
3. Install `Tc3_AnyBuffer` by KimRo and resolve all library references
4. Activate the configuration and connect to target hardware with EL6001 or EL6022
5. Build and download — `MAIN.TcPOU` instantiates four devices (addresses 20, 30, 40, 50); device 4 is intentionally misconfigured to demonstrate error handling behavior

## Design Principles

### SOLID principles

- **Single Responsibility**: `ModbusHandler` owns only serial dispatch; `ModbusDevice` owns only device-specific register logic — neither does both
- **Open/Closed**: Add new device drivers by copying and extending `ModbusDevice` without modifying `ModbusHandler`
- **Liskov Substitution**: All device drivers use the same `REFERENCE TO ModbusHandler` input — any compliant handler instance can be substituted
- **Interface Segregation**: `ST_ModbusCommand` carries only what the handler needs; device state and HMI data are isolated in `ST_ModbusDevice_HMI`
- **Dependency Inversion**: `ModbusDevice` depends on the `ModbusHandler` abstraction passed by reference, not a concrete instance it constructs internally

### Design patterns

- **State machine**: `ModbusHandler` cycles through `WAITING` → `EXECUTE` → `COOLDOWN` → `ERROR`, enabling non-blocking serial dispatch without recursion or busy-wait
- **Command queue (FIFO)**: Device drivers fire-and-forget via `AddToBuffer()`; the handler serializes execution independently of caller cycles
- **Pointer-to-result**: Each `ST_ModbusCommand` stores a `POINTER TO HRESULT`; the handler writes the outcome asynchronously, eliminating polling at the bus level
- **Composition over inheritance**: `ModbusDevice` holds a reference to `ModbusHandler` rather than extending it, keeping both independently reusable
- **Union-based type punning**: `ModbusReal` and `ModbusDint` overlay `WORD` arrays with `REAL`/`DINT` to convert endianness without `MEMCPY`
- **Dual-task producer/consumer**: The 10 ms `PlcTask` produces commands; the 1 ms `BGComm` task consumes them — the FIFO is the only handoff point between tasks

## Best Practices

1. Always run `ModbusHandler` in a dedicated task with **higher priority** than your device logic tasks — sharing a task causes slower serial I/O to block PLC cycle execution
2. Keep `ModbusTimeout` (default `T#500MS`) shorter than your application's watchdog or alarm scan interval so Modbus errors surface before higher-level logic times out
3. Poll the `HRESULT` result pointer only after at least one `BGComm` task cycle — it remains `S_Pending` until the handler completes the command
4. Do not increase `MAXMODBUSBUFFER` beyond the number of commands that can realistically be processed per application cycle; a perpetually full queue silently drops new entries
5. Use `_ConvertModbusToReal()` / `_ConvertRealToModbus()` as reference implementations when adding support for new data types — the `ModbusReal` union approach avoids manual pointer arithmetic

## Thread Safety

The FIFO buffer provided by `Tc3_AnyBuffer.FifoBuffer` is the **synchronization boundary** between the 10 ms `PlcTask` (producers) and the 1 ms `BGComm` task (consumer). Each `AddToBuffer()` call is an atomic push into the ring buffer; the handler pops one entry per cycle. Because TwinCAT tasks run on a single-core real-time scheduler with non-preemptive switching at task boundaries, no additional locking is required — provided `ModbusHandler()` is called exclusively from one task and `AddToBuffer()` is called exclusively from the other. Do not call `ModbusHandler()` and `AddToBuffer()` from the same task.

## Error Handling

When a command fails due to a Modbus timeout or device error, `ModbusHandler` writes `E_FAIL` to the command's `POINTER TO HRESULT`, sets its `Error` output, captures the raw error code in `ErrorId`, and logs the event via `ADSLOGSTR` before transitioning through `COOLDOWN` and resuming. Device drivers detect failure by polling their stored `HRESULT` and propagate the `Error` output accordingly.

```iecst
// Success path
IF _readProcessValue1Result = S_OK THEN
    _realConverter.w[0] := _rawValueProcessValue1[1];
    _realConverter.w[1] := _rawValueProcessValue1[0];
    PVProcessValue1      := _realConverter.r;
END_IF

// Failure path
IF _readProcessValue1Result = E_FAIL THEN
    Error := TRUE;
END_IF
```

## Dependencies

- **`Tc2_ModbusRTU`**: Beckhoff Automation — provides `ModbusRtuMasterV2_KL6x22B`, the hardware-level Modbus RTU master function block
- **`Tc2_Standard`**: Beckhoff Automation — standard IEC 61131-3 blocks (`TON`, `R_TRIG`)
- **`Tc2_System`**: Beckhoff Automation — system utilities (`MEMCPY`, `ADSLOGSTR`)
- **`Tc3_Module`**: Beckhoff Automation — TwinCAT 3 module support
- **`Tc3_AnyBuffer`**: KimRo — `FifoBuffer` ring buffer used as the command queue; third-party, must be installed separately

## Contributing

Contributions are welcome. Areas where extension is most useful:

- Additional device driver examples for common Modbus RTU sensors and actuators
- Extended endian conversion helpers for additional 32-bit and 64-bit types
- Support for Modbus TCP alongside RTU in the handler layer
- Unit test fixtures using TcUnit

## License

[BSD Zero Clause License (0BSD)](LICENSE.md) — use, copy, and modify freely with or without attribution.

## Authors

Robin Cardinaels — Beckhoff Belgium

## Acknowledgments

- [Tc3_AnyBuffer by KimRo](https://github.com/Roald87/TcAnyBuffer) — FIFO ring buffer implementation used for the command queue
- [Beckhoff TF6255 Modbus RTU documentation](./Documentation/TF6255_TC3_Modbus_RTU_EN.pdf) — hardware and library reference

## Support

- Open an issue in this repository for bugs or feature requests
- Consult the [Beckhoff Information System](https://infosys.beckhoff.com) for `Tc2_ModbusRTU` and EL6001/EL6022 hardware documentation
- For `Tc3_AnyBuffer` questions, refer to the KimRo repository

---

**Quick Links**

**Components** | [ModbusHandler](#modbustandler---command-queue-processor) | [ModbusDevice](#modbusdevice---device-driver-template)

**Getting Started** | [Installation](#installation) | [Usage Examples](#usage-examples) | [Project Structure](#project-structure)

**Reference** | [Design Principles](#design-principles) | [Error Handling](#error-handling) | [Dependencies](#dependencies)
