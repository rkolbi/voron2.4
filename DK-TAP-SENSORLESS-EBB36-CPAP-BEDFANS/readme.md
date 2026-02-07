# Voron 2.4 Configuration
## with Voron TAP, Sensorless X&Y Homing, EBB36 CAN Toolhead, and CPAP Part Cooling

---

This Kalico (DangerKlipper) configuration is optimized for the Voron 2.4 3D printer, providing comprehensive automation and advanced features for exceptional print quality and user experience.

---

## Configuration Overview

### Modular File Management

This configuration uses an intelligent directory-based system for organizing configuration files:

- **ACTIVE Directory**: All `.cfg` files in this directory are automatically loaded by Klipper
- **INACTIVE Directory**: Files stored here are ignored, allowing easy testing and organization
- **Easy Management**: Edit files via Samba network shares or SSH as if they were on your desktop

**To disable a configuration file:**
- Rename the extension (e.g., `feature.cfg` → `feature.cfgno`), OR
- Move it to the INACTIVE directory

### Central Configuration (printer.cfg)

The root `printer.cfg` file serves as the central hub:
- Automatically loads all `.cfg` files from the ACTIVE directory
- Contains global variables used throughout the macro system
- Stores auto-generated parameters (input shaper, saved mesh, etc.) at the bottom

### Printer-Specific Settings

Hardware-specific configurations are organized in dedicated files like `CONFIG-V24-350-TAP-SENSORLESS.cfg`. The modular approach ensures portability and makes updates straightforward.

---

## Key Features

### Dynamic Motor Current System

This configuration implements an intelligent safety feature:
- **Low currents** during homing and maintenance (prevents damage from collisions)
- **High currents** automatically applied during printing (full performance)
- **Collision protection** reduces risk of bent rods or damaged components

### Advanced Macro Suite

- **Adaptive Bed Mesh**: Meshes only the print area for faster, smarter leveling
- **Heat Soak Management**: Configurable pre-print chamber stabilization
- **Automated Homing & Leveling**: QGL and mesh calibration with thermal protection
- **Bed Fan Control**: Automatic chamber temperature management
- **Filament Handling**: Load/unload macros with customizable settings
- **LED Effects Integration**: Dynamic status indication (requires klipper-led_effect plugin)
- **Calibration Helpers**: PA_CAL, PID tuning, ADXL measurements

---

## Configuration Highlights

### PRINT_START Macro

The PRINT_START macro automates the entire print preparation workflow:

**Key Features:**
1. **Heat Soak Cycle**: Stabilizes chamber temperature before printing (configurable duration)
2. **Intelligent Homing**: Performs QGL and homing at optimal temperatures
3. **Adaptive Mesh**: Automatically meshes only the required print area
4. **Thermal Protection**: Ensures safe probing temperatures for Voron TAP
5. **Purge Line**: Two-stage purge for flow verification and nozzle priming
6. **Flexible Control**: Heat soak can be terminated early via RESUME or WAIT_QUIT

**Slicer Integration:**

For **OrcaSlicer/PrusaSlicer**, use this Start G-code:

```gcode
PRINT_START BED_TEMP=[first_layer_bed_temperature] EXTRUDER_TEMP=[first_layer_temperature] EXTRUDER_TEMP2=[temperature] CHAMBER_TEMP=[chamber_temperature] PA=0.040 ST=0.020 SIZE={first_layer_print_min[0]}_{first_layer_print_min[1]}_{first_layer_print_max[0]}_{first_layer_print_max[1]}
```

For **SuperSlicer**, use:

```gcode
PRINT_START BED_TEMP=[first_layer_bed_temperature] EXTRUDER_TEMP=[first_layer_temperature] EXTRUDER_TEMP2=[temperature] ENCLOSURE_TEMP=[chamber_temperature] PA=0.040 ST=0.020 SIZE={first_layer_print_min[0]}_{first_layer_print_min[1]}_{first_layer_print_max[0]}_{first_layer_print_max[1]}
```

**Optional Parameters:**

The following parameters can be specified in your slicer start code, or defaults from `printer.cfg` will be used:

- `PA=0.040` - Pressure advance value
- `ST=0.020` - Pressure advance smooth time
- `SOAK=15` - Heat soak duration in minutes (reduce for PLA/PETG, increase for ABS/ASA)
- `CHAMBER_TEMP=45` - Target chamber temperature for bed fan control

---

### LC_PAUSE (Layer Change Pause) Macro

Provides intelligent layer-based pausing functionality:

**Features:**
- Pause at next layer change (no parameter needed)
- Pause at specific layer (pass layer number as parameter)
- Toggle behavior (calling LC_PAUSE again cancels the pause)
- Seamless slicer integration

**Slicer Integration:**

Add to **After Layer Change G-code** in your slicer:

```gcode
_LAYER_CHANGE LAYER={layer_num}
```

**Usage Examples:**
```gcode
LC_PAUSE           # Pause on next layer
LC_PAUSE LAYER=50  # Pause when layer 50 starts
```

---

### PRINT_END Macro

Handles post-print cleanup and safety:

**Actions Performed:**
1. **Print Clearance**: Raises toolhead 10mm to prevent nozzle contact
2. **Cooling Activation**: Part cooling fan to 100% for rapid cooldown
3. **Smart Parking**: Positions toolhead for easy inspection and access
4. **Fan Clearance**: Moves to Y20 for unobstructed airflow
5. **Motor Current Reset**: Returns to low currents for safety

**Slicer Integration:**

Add to **End G-code**:

```gcode
PRINT_END
```

---

### PA_CAL (Pressure Advance Calibration) Macro

Simplified pressure advance tuning:

**Features:**
- Line-based PA test pattern
- Automatic value incrementation
- Customizable test parameters
- Visual validation of results

**Usage:**

```gcode
PA_CAL                                    # Use defaults from printer.cfg
PA_CAL BED=110 EXTRUDER_TEMP=245          # Specify temperatures
PA_CAL PA_START=0.020 PA_STEP=0.005       # Custom PA range
```

The macro prints test lines with incrementing PA values. Inspect the lines to find where corners look best (no bulging or gaps), then update your `printer.cfg` and slicer settings.

---

## Hardware Specifications

### Current Configuration

- **Printer**: Voron 2.4 (350mm build volume)
- **Firmware**: Kalico (DangerKlipper fork)
- **Mainboard**: BTT Octopus or similar (TMC2209 drivers)
- **Toolhead**: BTT EBB36 CAN board via U2C bridge
- **Probe**: Voron TAP with thermal protection
- **Homing**: Sensorless X/Y via TMC2209 StallGuard
- **Part Cooling**: CPAP blower with WS7040 driver
- **Extruder**: BMG-style (Bondtech LGX or similar)

### Performance Settings

- **Max Velocity**: 300 mm/s
- **Max Acceleration**: 4000 mm/s²
- **Print Acceleration**: 4000 mm/s² (via globalvariables)
- **Square Corner Velocity**: 6.0 mm/s
- **Pressure Advance**: 0.040 (tune for your filament)
- **Input Shaper**: Auto-calibrated (stored in SAVE_CONFIG)

---

## Network File Sharing Setup (Optional)

Enable easy file access from your desktop via Samba network shares.

### Installation

SSH into your Raspberry Pi and run:

```bash
sudo apt-get install samba winbind -y
```

### Configuration

Edit the Samba configuration file:

```bash
sudo nano /etc/samba/smb.conf
```

Add the following to the end of the file:

```ini
[Print_Files]
   comment = Voron G-Code Files
   path = /home/pi/printer_data/gcodes
   browseable = Yes
   writeable = Yes
   only guest = no
   create mask = 0777
   directory mask = 0777
   public = yes
   read only = no
   force user = root
   force group = root

[Klipper_Configs]
   comment = Voron Klipper Configuration
   path = /home/pi/printer_data/config
   browseable = Yes
   writeable = Yes
   only guest = no
   create mask = 0777
   directory mask = 0777
   public = yes
   read only = no
   force user = root
   force group = root

[Klipper_Storage]
   comment = Voron Storage
   path = /home/pi/Storage
   browseable = Yes
   writeable = Yes
   only guest = no
   create mask = 0777
   directory mask = 0777
   public = yes
   read only = no
   force user = root
   force group = root
```

### Apply Changes

Reboot your Raspberry Pi:

```bash
sudo reboot
```

### Windows Troubleshooting

If Windows shows a red X on mapped network shares, run CMD as administrator and execute:

```cmd
net stop workstation /y
net start workstation
```

### Accessing Shares

From Windows File Explorer or macOS Finder, navigate to:
```
\\your-printer-ip\Print_Files
\\your-printer-ip\Klipper_Configs
\\your-printer-ip\Klipper_Storage
```

---

## Tuning & Calibration

### Recommended Calibration Sequence

1. **PID Tuning** (Bed & Hotend)
   ```gcode
   PID_BED_CAL
   PID_NZL_CAL
   ```

2. **Input Shaper** (after toolhead assembly)
   ```gcode
   SHAPER_CALIBRATE
   # Or use: HEATSOAKED_ADXL for thermally stable measurements
   ```

3. **Pressure Advance** (for each filament type)
   ```gcode
   PA_CAL BED=110 EXTRUDER_TEMP=245 PA_START=0.020 PA_STEP=0.005
   ```

4. **First Layer & Z Offset** (with Voron TAP)
   - Heat bed to printing temperature
   - Run `QUAD_GANTRY_LEVEL`
   - Use paper test or first layer calibration print
   - Adjust via `SET_GCODE_OFFSET Z_ADJUST=±0.xxx` then `Z_OFFSET_APPLY_PROBE`

### Material-Specific Heat Soak Times

Adjust `SOAK` parameter in your slicer for different materials:

- **PLA**: `SOAK=0` or `SOAK=5` (minimal heat soak needed)
- **PETG**: `SOAK=5` to `SOAK=10`
- **ABS/ASA**: `SOAK=15` to `SOAK=20` (default, recommended)
- **Nylon/PC**: `SOAK=20` to `SOAK=30`

---

## Additional Resources

### Documentation & References

- **This Configuration**: [GitHub Repository](https://github.com/rkolbi/voron2.4/tree/main/MY_V24-350)
- **Adaptive Mesh**: [Frix-x's Klippain](https://github.com/Frix-x/klippain/blob/main/macros/calibration/adaptive_bed_mesh.cfg)
- **LED Effects**: [klipper-led_effect Plugin](https://github.com/julianschill/klipper-led_effect)
- **Sensorless Homing**: [Clee's Guide](https://docs.vorondesign.com/community/howto/clee/sensorless_xy_homing.html)

### Klipper Resources

- **Klipper Documentation**: [Main Docs](https://www.klipper3d.org/Overview.html)
- **Configuration Reference**: [Config Reference](https://www.klipper3d.org/Config_Reference.html)
- **G-Code Commands**: [G-Code Reference](https://www.klipper3d.org/G-Codes.html)
- **TMC Drivers**: [TMC Documentation](https://www.klipper3d.org/TMC_Drivers.html)
- **Klipper GitHub**: [Main Repository](https://github.com/Klipper3d)
- **Klipper Reddit**: [r/klippers](https://www.reddit.com/r/klippers/)

### Voron Resources

- **Voron Documentation**: [Build Guides](https://docs.vorondesign.com/build/)
- **Voron GitHub**: [VoronDesign](https://github.com/VoronDesign)
- **Voron Reddit**: [r/VORONDesign](https://www.reddit.com/r/VORONDesign/)

### Hardware Links

- **Euclid Probe**: [Documentation](https://euclidprobe.github.io/)
- **Bondtech LGX Extruder**: [Product Page](https://www.bondtech.se/product/lgx-large-gears-extruder/)
- **Bondtech CHT Nozzle**: [Product Page](https://www.bondtech.se/product/bondtech-cht-coated-brass-nozzle/)
- **MandalaRoseWorks Kinematic Mounts**: [Product Page](https://mandalaroseworks.com/products/matched-height-kinematic-kit)
- **MandalaRoseWorks Ultraflat Bed**: [Product Page](https://mandalaroseworks.com/products/voron-350-ultraflat-bed)

### General 3D Printing

- **RepRap G-code Wiki**: [G-code Reference](https://reprap.org/wiki/G-code)
- **Jinja2 Templates**: [Syntax Guide](https://jinja.palletsprojects.com/en/2.10.x/templates/)

---

## Configuration Philosophy

This configuration emphasizes:

1. **Safety First**: Dynamic current control, thermal protection, collision prevention
2. **Automation**: Minimize manual steps, maximize reliability
3. **Flexibility**: Easy to modify, well-documented, modular structure
4. **Performance**: Optimized for speed without sacrificing quality
5. **User Experience**: Clear feedback, intuitive controls, comprehensive macros

---

## Support & Contributions

For issues, improvements, or questions about this configuration:
- Review the documentation above
- Check Klipper and Voron documentation
- Consult the community resources linked above
- Submit issues or pull requests to the GitHub repository

---

**Last Updated**: February 7, 2026  
**Firmware**: Kalico v2026.02.00-0-gc70d21ab