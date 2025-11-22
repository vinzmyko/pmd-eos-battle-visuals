This guide covers setting up and using Lua scripts in DeSmuME for reverse engineering Pokemon Mystery Dungeon: Explorers of Sky.

---
## Setup

### DeSmuME Version
- **Tested Version:** DeSmuME 0.9.13 x64 SSE2
- **Platform:** Windows

### lua51.dll Installation

#### Check if lua51.dll is already present
Navigate to your DeSmuME folder and look for `lua51.dll`:
```
C:\path\to\DeSmuME\
├── DeSmuME_0.9.13_x64.exe
├── lua51.dll          ← Should be here
└── ...
```
#### If lua51.dll is missing:
1. **Download the correct version:**
   - Go to: https://sourceforge.net/projects/luabinaries/files/5.1.5/Windows%20Libraries/Dynamic/
   - Download: `lua-5.1.5_Win64_dll17_lib.zip` (for 64-bit DeSmuME)
2. **Extract and rename:**
   - Extract the zip file
   - Find `lua5.1.dll` inside
   - **Rename it to:** `lua51.dll`
1. **Place in DeSmuME folder:**
   - Copy `lua51.dll` to the same folder as `DeSmuME_0.9.13_x64.exe`
### Verification Test
```lua
print("Lua is working!")
gui.text(10, 10, "Hello from Lua!")
```
**To run:**
1. Open DeSmuME
2. Load your ROM
3. Go to: Tools → Lua Scripting → New Lua Script Window
4. Click "Browse" and select `test.lua`

---

## Important Notes

### Memory Overlays
Remember the correct overlay must be loaded when checking memory. Running the Lua scripts when the right memory overlay is loaded will give you expected results.

---

## Basic Script Structure
### Output to Console
Use `print()` to output text to the console:
```lua
print("This appears in the Output Console tab")
print(string.format("Value: %d", 123))
```
### Continuous Execution
For scripts that need to run continuously (updating display every frame), use `gui.register()`:
```lua
function updateDisplay()
    local value = memory.readword(0x022C9064)
    gui.text(10, 10, string.format("Value: %d", value))
end

gui.register(updateDisplay)
```
**Note:** DeSmuME handles loops differently than other emulators. The standard `while true do ... emu.frameadvance() end` pattern that works in other emulators will crash DeSmuME. Use `gui.register()` instead.

---

## Reading Memory for This Project
### Memory Read Functions
DeSmuME provides three main functions for reading memory:

| Function                    | Size    | Range        | Use Case                      |
| --------------------------- | ------- | ------------ | ----------------------------- |
| `memory.readbyte(address)`  | 1 byte  | 0-255        | Single byte fields, flags     |
| `memory.readword(address)`  | 2 bytes | 0-65535      | Most ID fields, small numbers |
| `memory.readdword(address)` | 4 bytes | 0-4294967295 | Large numbers, pointers       |

**Important:** The Nintendo DS uses little-endian byte order.

### Address Calculation
Most data structures follow this pattern:
```
address = base_address + (ID × structure_size) + field_offset
```
**Example:** To read the effect_id from move animation 251 (Razor Leaf):
- Base address: `0x022C9064`
- Structure size: `24` bytes
- Move ID: `251`
- Field offset: `0x0` (effect_id is the first field)
```lua
local MOVE_ANIM_BASE = 0x022C9064
local MOVE_ANIM_SIZE = 24
local move_id = 251

local address = MOVE_ANIM_BASE + (move_id * MOVE_ANIM_SIZE) + 0x0
local effect_id = memory.readword(address)

print(string.format("Razor Leaf Effect ID: %d", effect_id))
-- Output: Razor Leaf Effect ID: 296
```
### String Formatting Reference
Lua uses `string.format()` for formatted output:

| Format | Type                    | Example                         | Output       |
| ------ | ----------------------- | ------------------------------- | ------------ |
| `%d`   | Decimal integer         | `string.format("%d", 296)`      | `296`        |
| `%04X` | 4-digit hex (uppercase) | `string.format("0x%04X", 296)`  | `0x0128`     |
| `%08X` | 8-digit hex (uppercase) | `string.format("0x%08X", addr)` | `0x022CA7EC` |
| `%02X` | 2-digit hex (uppercase) | `string.format("0x%02X", 3)`    | `0x03`       |

---

## Quick Reference
### Move Animation Table (NA)
- **Base Address:** `0x022C9064`
- **Structure Size:** `24` bytes (0x18)
- **Number of Entries:** 560 (IDs 0-559)
- **Address Formula:** `0x022C9064 + (move_id × 24)`
### Effect Animation Table
- **Base Address:** `0x022CC52C`
- **Structure Size:** `28` bytes (0x1C)
- **Address Formula:** `0x022CC52C + (effect_id × 28)`

---