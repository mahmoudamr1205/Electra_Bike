# Software Board General Protocol V13

## Table of Contents

1. [Physical Interface](#physical-interface)
2. [Frame Structure](#frame-structure)
3. [Command Explanation](#command-explanation)
   - [Basic Information 0x03 Command](#basic-information-0x03-command)
   - [Cell Voltage 0x04 Command](#cell-voltage-0x04-command)
   - [Hardware Version 0x05 Command](#hardware-version-0x05-command)
   - [Protection Statistics Count](#protection-statistics-count)
4. [Control MOS Command (FB)](#control-mos-command-fb)
5. [Parameter Read and Set Mode (0xFA)](#parameter-read-and-set-mode-0xfa)
6. [Control Command Explanation](#control-command-explanation)
   - [Control Command (0x0A Command)](#control-command-0x0a-command)
   - [Clear Historical Data (0x0B Command)](#clear-historical-data-0x0b-command)
   - [Enter Factory Password Command (0x0F Command, default is 0x5678)](#enter-factory-password-command-0x0f-command)
7. [Add Battery Internal Resistance Command (F6) - Not Yet Implemented](#add-battery-internal-resistance-command-f6)
8. [Bluetooth Password Protocol](#bluetooth-password-protocol)
9. [Chip Type Read Command](#chip-type-read-command)
10. [Control Heating Command (FC)](#control-heating-command-fc)
11. [Practical Example Analysis](#practical-example-analysis)
12. [Revision History](#revision-history)

---

## 1. Physical Interface

This protocol supports the general protocol for Jiabaida software board RS485/RS232/UART interfaces, consistent with the host computer protocol, with a baud rate of 9600 BPS or other custom rates. All 16-bit data is in big-endian mode, with the high byte first and the low byte last. (Note: There are example explanations at the end of the protocol.)

Since the protection board has a sleep mode, the first data packet cannot be responded to in sleep mode and needs to be sent again.

---

## 2. Frame Structure

**Host Send Command:**

| Start | Read | Command | Length | Data Content | Checksum | Stop | CALLBACK_ACK_ID |
|-------|------|---------|--------|--------------|----------|------|-----------------|
| 0xDD  | 0xA5 | Command | Length | Data Content | Checksum | 0x77 | Callback ID     |

**BMS Response:**

| Start | Command | Status | Length | Data Content | Checksum | Stop | CALLBACK_ID |
|-------|---------|--------|--------|--------------|----------|------|-------------|
| 0xDD  | Command | Status | Length | Data Content | Checksum | 0x77 | Callback ID |

**Main Command Codes:**

- **0x03**: Read basic information and status, including capacity, total voltage, current, temperature, protection status, etc.
- **0x04**: Read cell voltage, including the voltage of each cell in the battery.
- **0x05**: Read the hardware version number of the protection board.
- **0xAA**: Read the historical protection count of the protection board.

**Status Bits:**

- **0x00**: Success
- **0x80**: Command does not exist
- **0x81**: Invalid operation (e.g., not in factory mode or password mismatch)
- **0x82**: Checksum error
- **0x83**: Password pairing error

---

## 3. Command Explanation

### 3.1 Basic Information 0x03 Command

**Host Send:**

| Start | Read | Command | Length | Data Content | Checksum | Stop |
|-------|------|---------|--------|--------------|----------|------|
| 0xDD  | 0xA5 | 0x03    | 0x00   | --           | Checksum | 0x77 |

**BMS Response:**

| Start | Command | Status | Length | Data Content | Checksum | Stop |
|-------|---------|--------|--------|--------------|----------|------|
| 0xDD  | 0x03    | 0x00   | Length | Data Content | Checksum | 0x77 |

**Example:**

- Host Send: `DD A5 03 00 FF FD 77`
- BMS Response: `DD 03 00 1B 17 00 00 00 02 D0 03 E8 00 00 20 78 00 00 00 00 00 00 10 48 03 0F 02 0B 76 0B 82 AA BB CC FB FF 77`

**Data Content Explanation:**

| Data Content       | Byte Size | Description                                                                 |
|--------------------|-----------|-----------------------------------------------------------------------------|
| Total Voltage      | 2 Bytes   | Unit: 10mV, high byte first. Example: 0x1700 = 23.00V                      |
| Current            | 2 Bytes   | Unit: 10mA, signed 16-bit integer. Positive for charging, negative for discharging. |
| Remaining Capacity | 2 Bytes   | Unit: 10mAh. Example: 0x02D0 = 7200mAh                                     |
| Nominal Capacity   | 2 Bytes   | Unit: 10mAh. Example: 0x03E8 = 10000mAh                                    |
| Cycle Count        | 2 Bytes   | Example: 0x0002 = 2 cycles                                                 |
| Production Date    | 2 Bytes   | Example: 0x2078 = March 24, 2016                                           |
| Balance Status     | 2 Bytes   | Each bit represents a cell's balance status (0: off, 1: on)                |
| Protection Status  | 2 Bytes   | Each bit represents a protection status (0: no protection, 1: protection)  |
| Software Version   | 1 Byte    | Example: 0x10 = Version 1.0                                                |
| RSOC               | 1 Byte    | Remaining State of Charge. Example: 0x48 = 72%                             |
| FET Control Status | 1 Byte    | Bit 0: Charge MOS, Bit 1: Discharge MOS (0: off, 1: on)                    |
| Number of Cells    | 1 Byte    | Example: 0x0F = 15 cells                                                   |
| NTC Count          | 1 Byte    | Example: 0x02 = 2 temperature sensors                                      |
| NTC Data           | 2*N Bytes | Unit: 0.1K. Example: 0x0B76 = 2934 (20.3°C)                                |
| Humidity           | 1 Byte    | Unit: 1%                                                                   |
| Alarm Status       | 2 Bytes   | Each bit represents an alarm status                                         |
| Full Charge Capacity | 2 Bytes | Unit: 10mAh                                                               |
| Remaining Capacity | 2 Bytes   | Unit: 10mAh                                                               |
| Balance Current    | 2 Bytes   | Unit: mA                                                                   |

---

### 3.2 Cell Voltage 0x04 Command

**Host Send:**

| Start | Read | Command | Length | Data Content | Checksum | Stop |
|-------|------|---------|--------|--------------|----------|------|
| 0xDD  | 0xA5 | 0x04    | 0x00   | --           | Checksum | 0x77 |

**BMS Response:**

| Start | Command | Status | Length | Data Content | Checksum | Stop |
|-------|---------|--------|--------|--------------|----------|------|
| 0xDD  | 0x04    | 0x00   | Length | Data Content | Checksum | 0x77 |

**Example:**

- Host Send: `DD A5 04 00 FF FC 77`
- BMS Response: `DD 04 00 1E 0F 66 0F 63 0F 63 0F 64 0F 3E 0F 63 0F 37 0F 5B 0F 65 0F 3B 0F 63 0F 63 0F 3C 0F 66 0F 3D F9 F9 77`

**Data Content Explanation:**

| Data Content       | Byte Size | Description                                                                 |
|--------------------|-----------|-----------------------------------------------------------------------------|
| Cell Voltage       | 2 Bytes   | Unit: mV, high byte first. Example: 0x0F66 = 3942mV                        |

---

### 3.3 Hardware Version 0x05 Command

**Host Send:**

| Start | Read | Command | Length | Data Content | Checksum | Stop |
|-------|------|---------|--------|--------------|----------|------|
| 0xDD  | 0xA5 | 0x05    | 0x00   | --           | Checksum | 0x77 |

**BMS Response:**

| Start | Command | Status | Length | Data Content | Checksum | Stop |
|-------|---------|--------|--------|--------------|----------|------|
| 0xDD  | 0x05    | 0x00   | Length | Data Content | Checksum | 0x77 |

**Example:**

- Host Send: `DD A5 05 00 FF FB 77`
- BMS Response: `DD 05 00 0A 30 31 32 33 34 35 36 37 38 39 FD E9 77`

**Data Content Explanation:**

| Data Content       | Byte Size | Description                                                                 |
|--------------------|-----------|-----------------------------------------------------------------------------|
| Hardware Version   | N Bytes   | ASCII characters representing the hardware version. Example: "0123456789"  |

---

### 3.4 Protection Statistics Count

**Host Send:**

| Start | Read | Command | Length | Data Content | Checksum | Stop |
|-------|------|---------|--------|--------------|----------|------|
| 0xDD  | 0xA5 | 0xAA    | 0x00   | --           | Checksum | 0x77 |

**BMS Response:**

| Start | Command | Status | Length | Data Content | Checksum | Stop |
|-------|---------|--------|--------|--------------|----------|------|
| 0xDD  | 0xAA    | 0x00   | Length | Data Content | Checksum | 0x77 |

**Example:**

- Host Send: `DD A5 AA 00 FF 56 77`
- BMS Response: `DD AA 00 18 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 FF E8 77`

**Data Content Explanation:**

| Data Content       | Byte Size | Description                                                                 |
|--------------------|-----------|-----------------------------------------------------------------------------|
| Short Circuit Count | 2 Bytes   | Example: 0x0001 = 1 short circuit event                                    |
| Charge Overcurrent Count | 2 Bytes | Example: 0x0002 = 2 charge overcurrent events                              |
| Discharge Overcurrent Count | 2 Bytes | Example: 0x0003 = 3 discharge overcurrent events                          |
| Cell Overvoltage Count | 2 Bytes | Example: 0x0004 = 4 cell overvoltage events                                |
| Cell Undervoltage Count | 2 Bytes | Example: 0x0005 = 5 cell undervoltage events                              |
| Charge Overtemperature Count | 2 Bytes | Example: 0x0006 = 6 charge overtemperature events                        |
| Charge Undertemperature Count | 2 Bytes | Example: 0x0007 = 7 charge undertemperature events                      |
| Discharge Overtemperature Count | 2 Bytes | Example: 0x0008 = 8 discharge overtemperature events                    |
| Discharge Undertemperature Count | 2 Bytes | Example: 0x0009 = 9 discharge undertemperature events                  |
| Total Overvoltage Count | 2 Bytes | Example: 0x000A = 10 total overvoltage events                            |
| Total Undervoltage Count | 2 Bytes | Example: 0x000B = 11 total undervoltage events                          |
| System Restart Count | 2 Bytes | Example: 0x000C = 12 system restarts                                     |

---

## 4. Control MOS Command (FB)

**Host Send:**

| Start | Status | Command | Length | Data Content | Checksum | Stop |
|-------|--------|---------|--------|--------------|----------|------|
| 0xDD  | 0x5A   | 0xFB    | 0x02   | YY XX        | Checksum | 0x77 |

**BMS Response:**

| Start | Command | Status | Length | Data Content | Checksum | Stop |
|-------|---------|--------|--------|--------------|----------|------|
| 0xDD  | 0xFB    | 0x00   | 0x00   | --           | Checksum | 0x77 |

**Example:**

- Host Send: `DD 5A FB 02 01 01 FF 01 77` (Close Charge MOS)
- BMS Response: `DD FB 00 00 00 00 77`

---

## 5. Parameter Read and Set Mode (0xFA)

### 5.1 Read Parameters

**Host Send:**

| Start | Status | Command | Length | Data Content | Checksum | Stop |
|-------|--------|---------|--------|--------------|----------|------|
| 0xDD  | 0xA5   | 0xFA    | 0x03   | Parameter ID | Checksum | 0x77 |

**BMS Response:**

| Start | Command | Status | Length | Data Content | Checksum | Stop |
|-------|---------|--------|--------|--------------|----------|------|
| 0xDD  | 0xFA    | 0x00   | Length | Data Content | Checksum | 0x77 |

**Example:**

- Host Send: `DD A5 FA 03 00 01 02 FF 00 77` (Read 2 registers starting from 0x0001)
- BMS Response: `DD FA 00 07 00 01 02 0F A0 10 36 FF 01 77` (Register 0x0001 = 0x0FA0, Register 0x0002 = 0x1036)

---

### 5.2 Write Parameters

**Enter Factory Mode:**

| Start | Status | Command | Length | Data Content | Checksum | Stop |
|-------|--------|---------|--------|--------------|----------|------|
| 0xDD  | 0x5A   | 0x00    | 0x02   | 0x5678       | Checksum | 0x77 |

**Exit Factory Mode:**

| Start | Status | Command | Length | Data Content | Checksum | Stop |
|-------|--------|---------|--------|--------------|----------|------|
| 0xDD  | 0x5A   | 0x01    | 0x02   | 0x2828       | Checksum | 0x77 |

---

## 6. Control Command Explanation

### 6.1 Control Command (0x0A Command)

**Host Send:**

| Start | Status | Command | Length | Data Content | Checksum | Stop |
|-------|--------|---------|--------|--------------|----------|------|
| 0xDD  | 0x5A   | 0x0A    | 0x02   | 0xAA 0xBB    | Checksum | 0x77 |

**BMS Response:**

| Start | Command | Status | Length | Data Content | Checksum | Stop |
|-------|---------|--------|--------|--------------|----------|------|
| 0xDD  | 0x0A    | 0x00   | 0x00   | --           | Checksum | 0x77 |

**Example:**

- Host Send: `DD 5A 0A 02 01 00 FF F3 77` (Reset Capacity)
- BMS Response: `DD 0A 00 00 00 00 77`

---

### 6.2 Clear Historical Data (0x0B Command)

**Host Send:**

| Start | Status | Command | Length | Data Content | Checksum | Stop |
|-------|--------|---------|--------|--------------|----------|------|
| 0xDD  | 0x5A   | 0x0B    | 0x02   | 0x18 0x81    | Checksum | 0x77 |

**BMS Response:**

| Start | Command | Status | Length | Data Content | Checksum | Stop |
|-------|---------|--------|--------|--------------|----------|------|
| 0xDD  | 0x0B    | 0x00   | 0x00   | --           | Checksum | 0x77 |

**Example:**

- Host Send: `DD 5A 0B 02 18 81 FF 5A 77` (Clear Historical Data)
- BMS Response: `DD 0B 00 00 00 00 77`

---

### 6.3 Enter Factory Password Command (0x0F Command)

**Host Send:**

| Start | Status | Command | Length | Data Content | Checksum | Stop |
|-------|--------|---------|--------|--------------|----------|------|
| 0xDD  | 0x5A   | 0x0F    | Length | Password     | Checksum | 0x77 |

**BMS Response:**

| Start | Command | Status | Length | Data Content | Checksum | Stop |
|-------|---------|--------|--------|--------------|----------|------|
| 0xDD  | 0x0F    | 0x00   | 0x00   | --           | Checksum | 0x77 |

**Example:**

- Host Send: `DD 5A 0F 04 56 78 12 34 FF 56 77` (Change Password from 0x5678 to 0x1234)
- BMS Response: `DD 0F 00 00 00 00 77`

---

## 7. Add Battery Internal Resistance Command (F6) - Not Yet Implemented

**Host Send:**

| Start | Status | Command | Length | Data Content | Checksum | Stop |
|-------|--------|---------|--------|--------------|----------|------|
| 0xDD  | 0xA5   | 0xF6    | 0x00   | --           | Checksum | 0x77 |

**BMS Response:**

| Start | Command | Status | Length | Data Content | Checksum | Stop |
|-------|---------|--------|--------|--------------|----------|------|
| 0xDD  | 0xF6    | 0x00   | Length | Data Content | Checksum | 0x77 |

---

## 8. Bluetooth Password Protocol

### 8.1 Password Pairing

**Host Send:**

| Start | Status | Command | Length | Data Content | Checksum | Stop |
|-------|--------|---------|--------|--------------|----------|------|
| 0xDD  | 0x5A   | 0x06    | 0x07   | Password     | Checksum | 0x77 |

**BMS Response:**

| Start | Command | Status | Length | Data Content | Checksum | Stop |
|-------|---------|--------|--------|--------------|----------|------|
| 0xDD  | 0x06    | 0x00   | 0x00   | --           | Checksum | 0x77 |

---

### 8.2 Password Change

**Host Send:**

| Start | Status | Command | Length | Data Content | Checksum | Stop |
|-------|--------|---------|--------|--------------|----------|------|
| 0xDD  | 0x5A   | 0x07    | 0x0D   | Old + New Password | Checksum | 0x77 |

**BMS Response:**

| Start | Command | Status | Length | Data Content | Checksum | Stop |
|-------|---------|--------|--------|--------------|----------|------|
| 0xDD  | 0x07    | 0x00   | 0x00   | --           | Checksum | 0x77 |

---

## 9. Chip Type Read Command

**Host Send:**

| Start | Status | Command | Length | Data Content | Checksum | Stop |
|-------|--------|---------|--------|--------------|----------|------|
| 0xDD  | 0xA5   | 0x00    | 0x00   | --           | Checksum | 0x77 |

**BMS Response:**

| Start | Command | Status | Length | Data Content | Checksum | Stop |
|-------|---------|--------|--------|--------------|----------|------|
| 0xDD  | 0x00    | 0x00   | 0x02   | Chip Type    | Checksum | 0x77 |

**Chip Types:**

- **0x00**: TI Solution
- **0x01**: AOT Solution
- **0x02**: Panasonic Solution
- **0x03**: SinoWealth 309 Solution
- **0x04**: SinoWealth 303 Solution
- **0x05**: JieChe DC10XX Solution

---

## 10. Control Heating Command (FC)

**Host Send:**

| Start | Status | Command | Length | Data Content | Checksum | Stop |
|-------|--------|---------|--------|--------------|----------|------|
| 0xDD  | 0x5A   | 0xFC    | 0x05   | XX HH MM ZZ WW | Checksum | 0x77 |

**BMS Response:**

| Start | Command | Status | Length | Data Content | Checksum | Stop |
|-------|---------|--------|--------|--------------|----------|------|
| 0xDD  | 0xFC    | 0x00   | 0x00   | --           | Checksum | 0x77 |

**Example:**

- Host Send: `DD 5A FC 05 01 00 00 05 FF 56 77` (Start Heating)
- BMS Response: `DD FC 00 00 00 00 77`

---

## 11. Practical Example Analysis

**Example 1: Read Cell Voltage**

- Host Send: `DD A5 04 00 FF FC 77`
- BMS Response: `DD 04 00 22 0E C8 0E C8 0E CB 0E CF 0E CA 0E C7 0E CA 0E CD 0E C9 0E CA 0E CB 0E CB 0E C8 0E CC 0E C8 0E C9 0E C9 F1 87 77`

**Example 2: Read Basic Information**

- Host Send: `DD A5 03 00 FF FD 77`
- BMS Response: `DD 03 00 1F 19 DF F8 24 0D A5 0F A0 00 02 24 91 00 00 00 00 00 00 12 57 03 11 04 0B 98 0B A9 0B 96 0B 97 F8 9A 77`

---

## 12. Revision History

| Version | Description                                                                 |
|---------|-----------------------------------------------------------------------------|
| V0      | Initial Draft                                                              |
| V1      | Modified 0xE1 command to 0xFB                                              |
| V2      | Fixed errors in parameter list                                              |
| V3      | Added callback_id in frame protocol                                         |
| V4      | Added chip type read command and adjusted hardware overcurrent values       |
| V5      | Added heating control command (0xFC) and forced start function              |
| V6      | Added discharge overcurrent and short-circuit values for chip type 5        |
| V7      | Optimized overcurrent values for chip type 3                                |
| V8      | Added storage mode command in 0x0A control command                          |
| V9      | Added current and capacity unit identifier (Bit 12 in function configuration) |
| V10     | Added forced start button in 0x0A command                                   |
| V11     | Added discharge overcurrent and short-circuit values for chip type 6        |
| V12     | Added factory mode password modification command                            |

---

This document provides a comprehensive guide to the Jiabaida software board protocol, including command structures, examples, and revision history.
