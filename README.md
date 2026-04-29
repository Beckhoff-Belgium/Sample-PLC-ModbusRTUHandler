# 📦 Sample — ModbusRTU Handler

---

## 🧠 Remarks and limitations

- This is a **sample project** — `ModbusDevice.TcPOU` is a template intended to be copied and adapted for your specific Modbus RTU device.
- Requires **TwinCAT 3.1 Build 4026.18** or higher.
- Requires the third-party library **`Tc3_AnyBuffer`** by KimRo (FIFO buffer implementation).
- The `ModbusHandler` FB must run in a **dedicated, higher-priority task** (separate from device logic). Placing both in the same task will cause blocking.
- Maximum command queue depth is set via `Param.MAXMODBUSBUFFER` (default: `200`). Exceeding this will silently drop commands.
- `MAIN.TcPOU` intentionally contains a faulty device instance (`ModbusDevice4`, address `0`) to demonstrate error handling behaviour.

---


## 🔧 Functionality Overview

### `ModbusHandler` (Function Block)
Queue-based Modbus RTU command processor. Decouples device drivers from serial communication, allowing multiple devices to share a single handler without blocking each other.

- Uses `Tc3_AnyBuffer.FifoBuffer` as a ring buffer for pending commands (FIFO, max `MAXMODBUSBUFFER` entries)
- Internal state machine: `WAITING` → `EXECUTE` → `COOLDOWN` → `ERROR`
- Configurable command timeout via `ModbusTimeout` input (default `T#500MS`)
- Results are written back asynchronously via `POINTER TO HRESULT` stored in each command
- Supports all standard Modbus functions: Read/Write Coils, Read/Write Registers, Read Input Registers, Diagnostics

| I/O | Name | Type | Default | Description |
|-----|------|------|---------|-------------|
| IN | `ModbusTimeout` | `TIME` | `T#500MS` | Timeout per command |
| OUT | `Busy` | `BOOL` | — | Command in progress |
| OUT | `Error` | `BOOL` | — | Error active |
| OUT | `ErrorId` | `UDINT` | — | Error code |

**Method:** `AddToBuffer(command : ST_ModbusCommand)` — queues a command; returns immediately.

---

### `ModbusDevice` (Function Block — Sample)
Sample Modbus RTU device driver built on top of `ModbusHandler`. Demonstrates cyclic reads, triggered writes, endian conversion, and HMI integration.

- Reads **2 process values** (REAL) cyclically at configurable interval (`ST_ModbusDeviceParam.UpdateTime`, default `T#1000MS`)
- Reads **serial number** (UDINT) once on initialisation
- Supports **calibration write** triggered via HMI command
- Handles **big-endian ↔ little-endian** conversion using `ModbusReal` and `ModbusDint` unions
- Exposes `ST_ModbusDevice_HMI` struct for TcHMI / symbol-level access

| I/O | Name | Type | Description |
|-----|------|------|-------------|
| IN | `Param` | `ST_ModbusDeviceParam` | Slave address + update interval |
| IN | `ModbusHandler` | `REFERENCE TO ModbusHandler` | Shared handler instance |
| OUT | `Error` | `BOOL` | Error active |
| OUT | `PVProcessValue1` | `REAL` | Process value 1 |
| OUT | `PVProcessValue2` | `REAL` | Process value 2 |

---

### Dual-Task Architecture

| Task | Cycle | Priority | Runs |
|------|-------|----------|------|
| `BGComm` | 1 ms | 4 (high) | `ModbusHandler` |
| `PlcTask` | 10 ms | 8 | `ModbusDevice` instances |

The faster `BGComm` task processes the serial bus independently of the device logic cycle.

---

## 🧪 Example

Declare and call one handler shared across multiple devices:

```iecst
(* BG_Communication.TcPOU — runs in BGComm task (1 ms) *)
PROGRAM BG_Communication
VAR
    modbushandler   : ModbusHandler;
END_VAR

modbushandler();
```

```iecst
(* MAIN.TcPOU — runs in PlcTask (10 ms) *)
PROGRAM MAIN
VAR
    ModbusDevice1   : ModbusDevice;
    ModbusDevice2   : ModbusDevice;
END_VAR

ModbusDevice1(
    Param           := (MBSAdress := 20, UpdateTime := T#1000MS),
    ModbusHandler   := BG_Communication.modbushandler
);

ModbusDevice2(
    Param           := (MBSAdress := 30, UpdateTime := T#1000MS),
    ModbusHandler   := BG_Communication.modbushandler
);
```

To create your own device driver, copy `ModbusDevice.TcPOU`, update the register addresses in `_Initialize()`, and adapt `_HandleResults()` for your data layout.

---

## 🧠 Notes

- **Endian conversion**: Modbus registers are big-endian; TwinCAT is little-endian. Use the provided `ModbusReal` and `ModbusDint` unions, or the `_ConvertModbusToReal` / `_ConvertModbusToUDint` methods as a reference.
- **Multiple devices, one handler**: All `ModbusDevice` instances queue their commands into the shared `ModbusHandler` FIFO. The handler processes them one at a time on the serial bus.
- **Result checking**: Each `ST_ModbusCommand` holds a `POINTER TO HRESULT`. Poll this pointer after calling the handler to check whether a command succeeded or failed.
- **Hardware**: Tested with Beckhoff **EL6001** and **EL6022** serial interface terminals connected via an EtherCAT coupler (EK1100 / EK1200).

---

## 🔢 Additional information

**Required libraries**

| Library | Vendor | Purpose |
|---------|--------|---------|
| `Tc2_ModbusRTU` | Beckhoff Automation | `ModbusRtuMasterV2_KL6x22B` FB |
| `Tc2_Standard` | Beckhoff Automation | Standard IEC 61131-3 (TON, R_TRIG, …) |
| `Tc2_System` | Beckhoff Automation | `MEMCPY`, `ADSLOGSTR`, … |
| `Tc3_Module` | Beckhoff Automation | Module support |
| `Tc3_AnyBuffer` | KimRo | `FifoBuffer` for command queuing |

**Supported platforms**: TwinCAT OS (ARMV7-A, ARMV7-M, ARMV8-A, x64, x64-E), TwinCAT RT (x64, x86)

**Minimum TwinCAT version**: 3.1.4026.18

**License**: [BSD Zero Clause License (0BSD)](LICENSE.md) — use freely with or without modification, no attribution required.
