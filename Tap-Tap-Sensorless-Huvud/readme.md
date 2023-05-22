**Voron 2.4 with Tap, Sensorless X&Y Homing, and Huvud w/U2C**  
=================================================================================================================

This Klipper configuration is designed for the Voron 2.4 with Tap. It includes the following features:

**File Management:**

The configuration will pull all files with the ".cfg" extension from the ACTIVE directory while ignoring any files in the INACTIVE directory. If you have Samba or other networking file share service set up, you can easily move and edit files as if they were on your desktop. If file sharing service is not available, you can prevent a file from being pulled into Klipper by renaming its extension within the ACTIVE directory. For example, renaming "do_stuff.cfg" to "do_stuff.cfgno" will exclude it. Instructions for setting up Samba network sharing can be found below.

**printer.cfg:**

The "printer.cfg" file in the root directory loads all files with the ".cfg" extension from the ACTIVE directory. "printer.cfg" also contains all the changeable global variables that the included macros use, keeping them in an easy to find spot. Saved parameters written by Klipper are also stored at the bottom of "printer.cfg."

**Printer-Specific Configurations:**

Printer-specific configurations should be placed in a file named something like "CONFIG-VORON24_350.cfg" (you can choose a different name if desired). These configurations are specific to your core printer. If desired, you can further split the contents into separate files. Remember that "printer.cfg" will load all files from the ACTIVE directory with the ".cfg" extension, regardless of their names. Make sure the file names make sense to you and are easily portable for future use.

**PRINT_START** performs the following steps:

1. Homes and levels the gantry.
2. Performs a heatsoak. Note that the heatsoak cycle can be terminated early by selecting RESUME or executing the WAIT_QUIT macro.
3. Re-homes and levels the gantry once heatsoaked.
4. Applies Adaptive Mesh. For more information, refer to Frix-x's github [here](https://github.com/Frix-x/klippain/blob/main/macros/calibration/adaptive_bed_mesh.cfg).
5. Brings the hotend to temperature.
6. Executes a purge line from the front left across the X-axis for easy verification. The extrusion rate of the purge line is calculated based on the [nozzle_diameter] value. 
7. Performs a simple line-based pressure advance proof.
  

The start print G-code for SuperSlicer should contain the following line:

```
PRINT_START BED_TEMP=[first_layer_bed_temperature] EXTRUDER_TEMP=[first_layer_temperature] EXTRUDER_TEMP2=[temperature] ENCLOSURE_TEMP=[chamber_temperature] PA=0.045 ST=0.21 SIZE={first_layer_print_min[0]}{first_layer_print_min[1]}{first_layer_print_max[0]}_{first_layer_print_max[1]}
```



The following optional parameters can be specified, or the values set in "printer.cfg" will be used:

- **PA**: Pressure advance, e.g., `PA=0.045`
- **ST**: Pressure advance smooth-time, e.g., `ST=0.21`
- **SOAK**: Minutes to heat-soak before final G32, meshing, and printing, e.g., `SOAK=15`
- **ENCLOSURE_TEMP**: Desired temperature for enclosure control (if enabled)

**PRINT_END** performs the following steps:

1. Raises the print by 10mm when completed.
2. Activates full cooling by turning the fan on.
3. Parks the toolhead at the top, front-right position.
  The parked Z position will be at least [ParkHeightPercentage] of the maximum axis height or at the printed object's Z height + 10 (whichever is taller). It is recommended to set [ParkHeightPercentage] to 0.5 to allow for easy visual inspection of the toolhead/nozzle and removal of debris. 
4. Additionally, PRINT_END will move the toolhead back to Y20 to provide room for the fans to pull air without being blocked by the doors.

The end print G-code for SuperSlicer should contain the following line:

```
PRINT_END
```


:bulb: These configs utilize the Klipper LED Effects plugin, which can be found at [this GitHub repository](https://github.com/julianschill/klipper-led_effect).

:bulb: Sensorless homing is based on Clee's work. You can find more information on sensorless XY homing for Voron printers [here](https://docs.vorondesign.com/community/howto/clee/sensorless_xy_homing.html), and Klipper documentation on TMC Drivers [here](https://www.klipper3d.org/TMC_Drivers.html#sensorless-homing).

<br>  
<br>  

**HUVUD Setup**
=================================================================================================================

TAP probe (optical) with Huvud toolhead controller wiring diagram follows. Detailed information on CAN [here](CAN-Application.pdf).

![HUVUD_TAP_Wiring](huvud_tap_connection.jpg)


​          

**Solving Excessive CAN0 RX Errors, running rPi4 64bit, klipper latest 64bit**  
*-Linux 5.15.84-v8+ #1613 SMP PREEMPT Thu Jan 5 12:03:08 GMT 2023 aarch64 GNU/Linux*  
*-WaveShare RS485 CAN HAT for Raspberry Pi - 12M crystal*  

I got many daily RX errors (`ip -details -statistics link show can0`). While not showing any errors in Klipper, I suffered a communication timeout error twice over the past few days. Through some internet diving, SPI speeds with 64bit OS are an issue, suggesting they are about halved and change based on the core speed of the rPi. I think this situation was made worse by WaveShare's documentation to set `spimaxfrequency=2000000`. As of now, I have been RX error free by using the following @ bitrate of 1,000,000.  

:zap:*Please research these changes before implementing them, as I am no rPi expert - just letting you know what worked for me.*  
<br>  
To get the WaveShare canhat working properly, I set these in ` /boot/config.txt`  
```
dtparam=spi=on  
dtoverlay=mcp2515-can0,oscillator=12000000,interrupt=25,spimaxfrequency=10000000  
dtoverlay=spi0-hw-cs  
enable_uart=0  
```

And also set `/etc/network/interfaces.d/can0` as:  
```
auto can0  
iface can0 can static  
 bitrate 1000000  
 up ifconfig $IFACE txqueuelen 256  
 pre-up ip link set can0 type can bitrate 1000000  
 pre-up ip link set can0 txqueuelen 256  
```

<br>

<br>

May be releated to WaveShare success - not sure, I lock my speed to 1200000 by adding the following to `/boot/config.txt`  
```
arm_freq=1200  
core_freq=500  
core_freq_min=500  
```

And made the following changes to `/etc/rc.local`  
```
_IP=$(hostname -I) || true  
if [ "$_IP" ]; then  
  printf "My IP address is %s\n" "$_IP"  
fi  
iwconfig wlan0 power off  
echo "performance" | sudo tee /sys/devices/system/cpu/cpufreq/policy0/scaling_governor  
exit 0  
```

<br>

**SAMBA Setup - GCODE File Network Share Setup**
=================================================================================================================

To set up a network file share, making your gcode and Klipper config files available to all your network-attached devices - either windows or mac file explorers, perform the following once ssh'd into your Klipper rPi. Note that this samba configuration is 'open' and therefore accessible to any device attached to the same LAN, just as the Mainsail HTTP interface is. While it is possible to restrict access to these samba file shares, I feel it is a moot point given the nature of the rest of the software suite, so I will not cover that here. If this is worrisome to you, please research alternate configurations. 

 -Install file serving smb service:  
$`sudo apt-get install samba winbind -y`  

 -Configure file share.  Add the following to the end of the smb.conf file:  
$`sudo nano /etc/samba/smb.conf`  
	
```
[Print_Files]  
   comment = Voron_gCode_files  
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
   comment = Voron_Klipper  
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
   comment = Voron_Storage
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



 -Reboot rPi:  
$`sudo reboot`  
	

In the event that windows shows a red-x through the mapped share, I have found the following helpful in restoring mapped network shares:  
- Run `CMD` as administrator  
- `net stop workstation /y`  
- `net start workstation`  

<br>  
	


**Links**
=================================================================================================================



My Github (This file)                     https://github.com/rkolbi/voron2.4/tree/main/MY_V24-350  
RepRap G-code Wiki                        https://reprap.org/wiki/G-code  
Klipper Documentation (Main):             https://www.klipper3d.org/Overview.html    
Klipper Configuration Reference:          https://www.klipper3d.org/Config_Reference.html    
Klipper G-Code & Additional Commands:     https://www.klipper3d.org/G-Codes.html    
Klipper Github:                           https://github.com/Klipper3d    
Klipper Reddit:                           https://www.reddit.com/r/klippers/    
Jinja2 Syntax & Semantics:                https://jinja.palletsprojects.com/en/2.10.x/templates/    
Voron Documentation:                      https://docs.vorondesign.com/build/    
Voron Github:                             https://github.com/VoronDesign    
Voron Reddit:                             https://www.reddit.com/r/VORONDesign/    
Euclid Probe:                             https://euclidprobe.github.io/    
Bondtech LGX:                             https://www.bondtech.se/product/lgx-large-gears-extruder/    
Bontech CHT Nozzle:                       https://www.bondtech.se/product/bondtech-cht-coated-brass-nozzle/    
MandalaRoseWorks Kinematic Mounts:        https://mandalaroseworks.com/products/matched-height-kinematic-kit   
MandalaRoseWorks Ultraflat Mag Bed:       https://mandalaroseworks.com/products/voron-350-ultraflat-bed	 
