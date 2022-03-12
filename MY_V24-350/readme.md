**NOTES & INFORMATION**
=================================================================================================================

This Klipper configuration will pull all files from the ACTIVE directory that end in .*cfg*, ignoring any files in the INACTIVE directory. Since I am using Samba networking file share, I can quickly move files back and forth and edit, just as they were in a folder on my desktop. If you cannot install Samba or access a network share as such, renaming any file's extension within the ACTIVE directory will prevent it from being pulled in Klipper. For example, [do_stuff.cfg] will be pulled into Klipper, while [do_stuff.cfgno] will not. Information on how to set up Samba network sharing is below.  

The [printer.cfg] in the root directory loads all the files ending with [.cfg] extension from the ACTIVE directory; the INACTIVE directory is merely a container to place the files that you may not want to be run at this particular time, such as [ARDU_ADXL.cfg]. [printer. cfg] also contains any global variables needed or shared by the macros. Any other dictionary required should be placed here to central localize all settings, making it easier to change parameters later. Additionally, any saved parameters written by Klipper will be at the bottom of [printer.cfg].  

Lastly, all the printer-specific settings should be placed in a file such as [CONFIG-VORON24_350.cfg]; these are the Klipper configurations specific to your core printer. The contents can be further split if you wish; remember that the [printer.cfg] will load all files from the ACTIVE directory, regardless of name, just so long as they end with the [.cfg] extension. Make them something that makes sense to you and may be easily portable for an easy future.  

<br>

**PRINT_START** will home and the level the gantry*, perform a heat soak, then G3201, and lastly bring the hotend to temperature and execute a purge line from the front left across X-axis to check/verify easily. The purge line extrusion rate will be calculated by using the [nozzle_diameter] value.  

 

**PRINT_END** will raise by 10mm when the print has completed, then performs a cooling period by turning the fan fully on and then parks the toolhead at the top, front-right position. The parked Z position will be at least 75% of the max axis height or at the printed object's Z height + 10 - whichever is taller - this allows for easy visual inspection of the toolhead/nozzle (and a reminder to do so) and removal of any debris. Additionally to note, PRINT_END will place the toolhead back at Y20 to allow room for the fan(s) to pull air without being blocked by being pressed up against the doors. 

**NOTE:** These configurations utilize the Euclid probe. There are two slightly different variants; Bed Mounted (BM) docking, where the dock has been mounted on the bottom rail opposite the Z-Endstop, and Gantry Mounted (GM) docking, where the dock is mounted on the gantry by the stepper motor. I have decided to switch and stay with Gantry mount as it provides a better solution for my needs and will update this set (GM) as expected. I will update the (BM) set with any critical fixes, but they are very stable and offer a solid experience as is. For fetching and docking the probe, use M401 and M402 as they have safety checks to prevent crashes! The following STL files were used:  

[Bed mounted dock]  
https://github.com/nionio6915/Euclid_Probe/blob/main/stls/Voron/Voron2.4BedMountV3.stl  
https://github.com/nionio6915/Euclid_Probe/blob/main/stls/Mount%20Adapters/Omron/OmronFotekM2_V3.stl  

[Gantery mounted dock]  
https://github.com/nionio6915/Euclid_Probe/blob/main/stls/Voron/VoronGantryMount2NativeCarriage.stl  
https://github.com/nionio6915/Euclid_Probe/blob/main/stls/Voron/VoronGantryMount2_Bkt.stl  



**Links:**  
My Github (This file)                     https://github.com/rkolbi/voron2.4/tree/main/MY_V24-350  
RepRap G-code Wiki                        https://reprap.org/wiki/G-code  
Klipper Documentation (Main):             https://www.klipper3d.org/Overview.html    
Klipper Configuration Reference:          https://www.klipper3d.org/Config_Reference.html    
Klipper G-Code & Additional Commands:     https://www.klipper3d.org/G-Codes.html    
Klipper Github:                           https://github.com/Klipper3d    
Klipper Reddit:                           https://www.reddit.com/r/klippers/    
Voron Documentation:                      https://docs.vorondesign.com/build/    
Voron Github:                             https://github.com/VoronDesign    
Voron Reddit:                             https://www.reddit.com/r/VORONDesign/    
Euclid Probe:                             https://euclidprobe.github.io/    
Bondtech LGX:                             https://www.bondtech.se/product/lgx-large-gears-extruder/    
Bontech CHT Nozzle:                       https://www.bondtech.se/product/bondtech-cht-coated-brass-nozzle/    

Thanks to Daniel0815 for testing & assistance on Z compensation.



<br><br>

**SAMBA Setup - GCODE File Network Share Setup**
=================================================================================================================

To set up a network file share, making your gcode and Klipper config files available to all your network-attached devices - either windows or mac file explorers, perform the following once ssh'd into your Klipper rPi. Note that this samba configuration is 'open' and therefore accessible to any device attached to the same LAN, just as the Mainsail HTTP interface is. While it is possible to restrict access to these samba file shares, I feel it is a moot point given the nature of the rest of the software suite, so I will not cover that here. If this is worrisome to you, please research alternate configurations. 

 -Install file serving smb service:  
$`sudo apt-get install samba winbind -y`  

 -Configure file share.  Add the following to the end of the smb.conf file:  
$`sudo nano /etc/samba/smb.conf`  
	
```
[voron]  
   comment = Vonon_gCode_files  
   path = /home/pi/gcode_files  
   browseable = Yes  
   writeable = Yes  
   only guest = no  
   create mask = 0777  
   directory mask = 0777  
   public = yes  
   read only = no  
   force user = root  
   force group = root  

[voron-klipper_config]  
   comment = Vonon_Klipper  
   path = /home/pi/klipper_config  
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
